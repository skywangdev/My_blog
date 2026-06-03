---
title: "GPU 集群压测技术笔记：nvidia-smi、DCGM、NCCL Tests 与结果判断"
author: "Sky Wang"
pubDatetime: 2024-11-12T16:10:00+08:00
featured: false
draft: false
tags:
  - Linux
  - NVIDIA
  - GPU
  - 压测
description: "整理 GPU 集群压测的基本方法，包括单机状态检查、DCGM 诊断、NCCL Tests 通信压测和结果判断。"
---

这篇记录 GPU 集群压测的基本方法。压测不是为了跑一个漂亮数字，而是确认节点、GPU、网络、容器和训练通信链路是否稳定。A100、H800 这类集群尤其要重视 NCCL 和拓扑。

## Table of contents

## 一、压测前先定义目标

压测前先说清楚要验证什么：

- 单张 GPU 是否稳定。
- 单节点多 GPU 是否稳定。
- 多节点通信是否稳定。
- 容器环境是否稳定。
- 调度系统下任务是否稳定。

如果目标不清楚，很容易出现“跑了很多命令，但不知道结果是否合格”的情况。

## 二、基础状态检查

先看 GPU：

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi -q | less
```

持续观察：

```bash
nvidia-smi dmon -s pucvmet -d 1
```

字段大致含义：

| 字段 | 说明 |
| --- | --- |
| `pwr` | 功耗 |
| `gtemp` | GPU 温度 |
| `mtemp` | 显存温度 |
| `sm` | SM 利用率 |
| `mem` | 显存利用率 |
| `enc/dec` | 编解码利用率 |

压测时要同时看温度、功耗和错误，不要只看利用率。

## 三、DCGM 诊断

DCGM 可以做数据中心 GPU 的诊断和监控。

安装后启动：

```bash
systemctl enable --now nvidia-dcgm
```

查看 GPU：

```bash
dcgmi discovery -l
```

跑诊断：

```bash
dcgmi diag -r 1
dcgmi diag -r 2
```

`-r 1` 偏快速检查，`-r 2` 更深入。正式交付前可以用更长时间的诊断，但要注意它会占用 GPU。

## 四、单机 CUDA 样例测试

如果安装了 CUDA samples，可以跑：

```bash
cd /usr/local/cuda/samples/1_Utilities/deviceQuery
make
./deviceQuery
```

结果里出现：

```txt
Result = PASS
```

说明 CUDA 运行时基本正常。

再跑带宽测试：

```bash
cd /usr/local/cuda/samples/1_Utilities/bandwidthTest
make
./bandwidthTest
```

这个测试不能代表训练性能，但可以快速发现明显异常。

## 五、NCCL Tests 编译

NCCL Tests 是 GPU 集群通信压测常用工具。

```bash
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests
make MPI=0 CUDA_HOME=/usr/local/cuda
```

如果做多机测试，需要 MPI：

```bash
make MPI=1 MPI_HOME=/path/to/mpi CUDA_HOME=/usr/local/cuda
```

## 六、单机 all_reduce 压测

8 卡单机测试：

```bash
./build/all_reduce_perf -b 8M -e 16G -f 2 -g 8
```

常看两列：

- `algbw`：算法带宽。
- `busbw`：总线带宽，更适合看通信效率。

同型号、同拓扑机器之间，`busbw` 差距如果明显，优先检查：

- Fabric Manager。
- NVLink/NVSwitch 状态。
- 驱动版本。
- GPU 是否降频。
- 温度和功耗限制。

## 七、多机 NCCL 压测

多机测试示例：

```bash
mpirun -np 16 \
  -H node01:8,node02:8 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_SOCKET_IFNAME=eth0 \
  ./build/all_reduce_perf -b 8M -e 16G -f 2 -g 1
```

如果是 IB/RDMA 网络，结合现场环境增加：

```bash
export NCCL_IB_DISABLE=0
export NCCL_IB_HCA=mlx5
export NCCL_IB_GID_INDEX=3
```

不要盲目复制变量。不同集群的网卡名、GID index、路由策略都可能不同。

## 八、压测时同步采集

压测时建议同时采集：

```bash
nvidia-smi dmon -s pucvmet -d 1 > gpu-dmon.log
dmesg -w > dmesg.log
journalctl -f > journal.log
```

网络侧：

```bash
sar -n DEV 1
ip -s link
```

如果压测异常，日志比最终数字更重要。

## 九、结果判断

判断压测结果时不要只看平均值。至少看：

- 是否报错。
- 是否出现 GPU 掉卡。
- 是否有 Xid 错误。
- 带宽是否稳定。
- 多轮结果是否一致。
- 节点之间差距是否过大。

查看 Xid：

```bash
dmesg | grep -i xid
journalctl -k | grep -i xid
```

常见异常包括：

- 某张卡温度过高导致降频。
- 网络丢包或带宽不达标。
- NCCL 选错网卡。
- Fabric Manager 异常。
- 容器和宿主机 CUDA/NCCL 版本不匹配。

## 十、压测报告建议

每次压测至少记录：

- 节点型号。
- GPU 型号和数量。
- 驱动版本。
- CUDA/NCCL 版本。
- 网络类型和网卡型号。
- 测试命令。
- 测试结果。
- 异常日志。

可以按节点保存：

```txt
benchmark/
  node01/
    nvidia-smi.txt
    topo.txt
    all_reduce.log
    gpu-dmon.log
  node02/
    ...
```

## 小结

GPU 集群压测的核心是建立基线。先确认单机，再确认多机；先看是否稳定，再看性能数字。没有日志和环境信息的压测结果，很难用于后续排障。
