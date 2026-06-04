---
title: "H800 集群部署记录：节点、网络、NCCL 和调度前检查"
author: "Sky Wang"
pubDatetime: 2024-05-16T09:40:00+08:00
featured: true
draft: false
tags:
  - Linux
  - NVIDIA
  - H800
  - GPU
description: "记录 H800 集群交付时需要检查的节点状态、网络、容器、NCCL 参数和调度前确认项。"
---

这篇记录 H800 GPU 集群部署时需要关注的几个层面：节点状态、网络、容器、NCCL 和调度前检查。H800 这类集群的问题通常不是某一条命令能解决，而是要把硬件、驱动、网络和任务运行环境放在一起看。

## Table of contents

## 一、部署目标

一个可用的 H800 集群，至少要满足：

- 单节点 8 卡稳定可用。
- 多节点网络连通和带宽符合预期。
- 容器能访问 GPU。
- NCCL 能正常跑单机和多机测试。
- 调度系统能正确识别 GPU 资源。

在这个基础上，再谈训练框架、模型部署和业务任务。

## 二、节点准备

每台节点先做基础检查：

```bash
hostname
cat /etc/os-release
uname -r
nvidia-smi
nvidia-smi topo -m
systemctl status nvidia-fabricmanager --no-pager
```

H800 常见是 HGX/SXM + NVSwitch 形态，这类机器要特别确认 Fabric Manager。它不是普通守护进程那么简单，而是 NVSwitch fabric 初始化和管理链路的一部分；如果服务异常，单卡 `nvidia-smi` 可能还能看起来正常，但多卡通信会埋雷。

建议把输出保存下来：

```bash
mkdir -p /data/check/$(hostname)
nvidia-smi > /data/check/$(hostname)/nvidia-smi.txt
nvidia-smi topo -m > /data/check/$(hostname)/topo.txt
```

后面出现性能差异时，这些基线信息很有用。

## 三、网络检查

多机 GPU 集群里，网络经常是瓶颈。先确认网卡：

```bash
ip link
lspci | grep -Ei 'ethernet|infiniband|mellanox'
```

如果使用 InfiniBand 或 RoCE，需要确认驱动和链路状态：

```bash
ibstat
ibv_devinfo
```

如果是以太网，也要确认 MTU、bond、VLAN、路由策略：

```bash
ip addr
ip route
ping -c 3 <peer-ip>
```

基础带宽测试可以先用 `iperf3`：

```bash
iperf3 -s
```

另一台节点：

```bash
iperf3 -c <server-ip> -P 8 -t 60
```

`iperf3` 不能替代 NCCL 压测，但能先排除明显的网络问题。

## 四、容器运行时检查

容器里确认 GPU：

```bash
docker run --rm --gpus all nvidia/cuda:12.1.1-base-ubuntu22.04 nvidia-smi
```

如果调度系统使用 containerd，也要确认对应运行时：

```bash
crictl info | grep -i nvidia
```

Kubernetes 环境里还需要 NVIDIA Device Plugin：

```bash
kubectl get pods -n kube-system | grep -i nvidia
kubectl describe node <node-name> | grep -i nvidia.com/gpu
```

## 五、NCCL 环境变量

NCCL 变量不要一开始就堆很多。先从最少变量开始，确认能跑，再根据网络和拓扑调优。

常见变量：

```bash
export NCCL_DEBUG=INFO
export NCCL_IB_DISABLE=0
export NCCL_SOCKET_IFNAME='=eth0'
```

`NCCL_SOCKET_IFNAME` 支持前缀、排除和精确匹配。这里写 `=eth0` 是精确匹配示例，不是推荐所有集群都叫这个名字；实际要按训练网络接口来填，比如 `=bond0`、`=ib0` 或 `^docker` 这类写法。

如果是 IB/RDMA 网络，需要确认实际 HCA 名和端口。RoCE 环境里有时还会涉及 GID index，但不要默认写死：

```bash
export NCCL_IB_HCA=mlx5
```

这些值不要照抄，要根据现场环境确认。`NCCL_IB_HCA=mlx5` 是前缀匹配；需要精确控制时可以写成 `=mlx5_0:1,mlx5_1:1`。NCCL 2.21 及以后通常会动态选择 GID index，只有老版本或网络方案明确要求时，再用 `show_gids` 之类工具确认后设置 `NCCL_IB_GID_INDEX`。

## 六、单机 NCCL 测试

单节点先跑：

```bash
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests
make MPI=0 CUDA_HOME=/usr/local/cuda
```

测试 8 卡 all_reduce：

```bash
./build/all_reduce_perf -b 8M -e 16G -f 2 -g 8
```

关注输出里的 `busbw`。同型号节点之间差距不应过大。

## 七、多机 NCCL 测试

多机通常需要 MPI 或集群调度系统配合。以 MPI 思路举例：

```bash
mpirun -np 16 \
  -H node01:8,node02:8 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_SOCKET_IFNAME='=eth0' \
  ./build/all_reduce_perf -b 8M -e 16G -f 2 -g 1
```

如果使用 IB/RDMA，再加对应变量：

```bash
mpirun -np 16 \
  -H node01:8,node02:8 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_IB_DISABLE=0 \
  -x NCCL_IB_HCA=mlx5 \
  ./build/all_reduce_perf -b 8M -e 16G -f 2 -g 1
```

这里的重点不是记命令，而是保存完整输出。NCCL 报错信息里通常会暴露网卡、路由、权限、库版本问题。

## 八、调度前检查清单

节点加入训练集群前建议确认：

- `nvidia-smi` 正常。
- Fabric Manager 正常。
- 容器内能看到 GPU。
- 单机 NCCL 测试通过。
- 多机 NCCL 测试通过。
- 网络带宽和延迟符合预期。
- 调度系统能识别 `nvidia.com/gpu`。
- 日志采集和监控已接入。

## 小结

H800 集群部署要把节点、网络、容器和 NCCL 放在同一条链路里验证。只验证单机 GPU 不够，只验证网络也不够。真正可靠的判断，是单机、多机、容器和调度链路都能跑通。

## 参考资料

- [NVIDIA Fabric Manager User Guide](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html)
- [NVIDIA NCCL Environment Variables](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)
- [NVIDIA NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [NVIDIA Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
