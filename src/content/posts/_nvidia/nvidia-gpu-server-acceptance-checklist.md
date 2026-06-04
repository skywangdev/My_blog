---
title: "GPU 服务器交付前验收：驱动、CUDA、拓扑和压测"
author: "Sky Wang"
pubDatetime: 2024-01-20T10:10:00+08:00
featured: true
draft: false
tags:
  - NVIDIA
  - GPU
  - CUDA
  - Linux
description: "记录 NVIDIA GPU 服务器交付前常做的验收项，包括 nvidia-smi、CUDA samples、拓扑、Fabric Manager、DCGM 和 NCCL Tests。"
---

GPU 服务器交付前，不能只看 `nvidia-smi` 能不能显示卡。驱动、CUDA、拓扑、Fabric Manager、容器、日志和基础压测都要看一遍。这里整理一套常用验收清单。

## Table of contents

## 一、先确认 GPU 是否都识别

最基础命令：

```bash
nvidia-smi
nvidia-smi -L
```

重点看：

- GPU 数量是否正确。
- 型号是否正确。
- Driver Version 是否符合要求。
- 是否有 `ERR!`、`Unknown Error`。
- 温度和功耗是否异常。

如果 `lspci` 能看到 GPU，但 `nvidia-smi` 看不到，优先排查驱动、Secure Boot、内核头文件、nouveau 和 PCIe 资源分配。

## 二、检查拓扑

多卡服务器一定要看拓扑：

```bash
nvidia-smi topo -m
```

常见标记：

| 标记  | 含义               |
| ----- | ------------------ |
| `NV#` | 通过 NVLink 连接   |
| `PIX` | 同一个 PCIe switch |
| `PXB` | 跨 PCIe bridge     |
| `PHB` | 跨 CPU host bridge |
| `SYS` | 跨 CPU socket      |

A100、H800 这类机器如果预期有 NVLink/NVSwitch，但拓扑看不到，先不要继续压测。

## 三、检查 Fabric Manager

带 NVSwitch 的 A100/H800 SXM 平台需要重点检查 Fabric Manager。PCIe 卡形态机器通常不需要它，不要为了“清单完整”硬装。检查：

```bash
systemctl status nvidia-fabricmanager
nvidia-smi topo -m
```

如果 Fabric Manager 没启动，单卡可能正常，多卡通信会出问题。Fabric Manager 包版本也要和驱动栈匹配，至少保持在同一驱动分支内。

## 四、检查 CUDA

如果安装了 CUDA Toolkit：

```bash
nvcc -V
```

如果没有完整 Toolkit，也可以通过容器或 CUDA samples 做验证。常见样例：

```bash
./deviceQuery
./bandwidthTest
```

`deviceQuery` 里看到：

```txt
Result = PASS
```

说明 CUDA 基础调用正常。

## 五、看内核日志

GPU 交付前建议看 Xid：

```bash
dmesg | grep -i xid
journalctl -k | grep -i xid
```

如果有 Xid，要结合时间、GPU 编号、驱动版本和负载判断。不要只清日志后交付。

## 六、跑 DCGM

安装 DCGM 后，可以跑基础诊断：

```bash
dcgmi discovery -l
dcgmi diag -r 1
dcgmi diag -r 2
```

官方 run level 里 `-r 1` 是 Quick，`-r 2` 是 Medium，`-r 3` 是 Long，`-r 4` 是 Extended。`-r 1` 适合快速巡检，正式交付或疑难排查可以按窗口期跑更长等级，但不要和业务负载混跑。

## 七、跑 NCCL Tests

单机 8 卡常用：

```bash
./build/all_reduce_perf -b 8M -e 8G -f 2 -g 8 -w 10 -n 50
```

多机测试时，要确认 MPI、网卡、RDMA 和 NCCL 环境变量。先小规模跑，再扩大规模。

看结果时重点看：

- 是否报错。
- 是否长时间卡住。
- `#wrong` 是否为 0。
- `busbw` 是否稳定。
- 同型号机器之间差距是否过大。

## 八、容器验证

如果业务跑在容器里，还要验证容器内能看到 GPU：

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

如果容器里看不到 GPU，优先检查 NVIDIA Container Toolkit、Docker 运行时配置和宿主机驱动。

## 九、交付记录

建议记录：

- GPU 型号和数量。
- 驱动版本。
- CUDA 版本。
- Fabric Manager 状态。
- `nvidia-smi topo -m` 输出。
- DCGM 诊断结果。
- NCCL Tests 结果。
- Xid 日志检查结果。

这些记录后面排障很有用，尤其是多节点性能差异问题。

## 十、小结

GPU 服务器验收要分层看：硬件识别、驱动、CUDA、拓扑、Fabric Manager、日志、容器和通信测试。每层都正常，交付结果才更可靠。

## 参考资料

- [NVIDIA Fabric Manager User Guide](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html)
- [NVIDIA DCGM Diagnostics](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/dcgm-diagnostics.html)
- [NVIDIA NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [NVIDIA Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
