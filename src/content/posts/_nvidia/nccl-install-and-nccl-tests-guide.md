---
title: "NCCL 测试笔记：安装、编译 nccl-tests 和多机压测"
author: "Sky Wang"
pubDatetime: 2024-11-03T12:00:00+08:00
featured: false
draft: false
tags:
  - Linux
  - NVIDIA
  - CUDA
  - NCCL
  - GPU
description: "记录 NCCL 安装、nccl-tests 编译、单机多卡和多机压测的常用命令，以及现场排查时常看的环境变量和日志。"
---

这篇记录我平时做 NCCL 测试时常用的一套流程：安装 NCCL、编译 `nccl-tests`、跑单机多卡和多机压测、看输出结果，再根据日志排查问题。内容参考 NVIDIA 官方 NCCL User Guide、Installation Guide 和 Troubleshooting 文档，也结合了一些现场常用命令。

如果你的节点还没有装好驱动、CUDA 和 Fabric Manager，建议先把这一层打稳，再做 NCCL 测试。NCCL 只是通信库，底层驱动、GPU 拓扑、网卡、RDMA、容器共享内存这些前置条件没处理好，测试结果很容易失真。

## Table of contents

## 一、先明确 NCCL 测什么

NCCL 是 NVIDIA 的多 GPU 集合通信库，支持：

- `AllReduce`
- `Broadcast`
- `Reduce`
- `AllGather`
- `ReduceScatter`

官方文档强调，NCCL 是为多 GPU 通信做优化的库，不是完整的并行编程框架。它会根据底层拓扑自动选择通信策略，支持 PCIe、NVLink、InfiniBand Verbs 和 IP sockets。

平时说“跑 NCCL”，一般是在看两件事：

- 通信能不能正常初始化，测试能不能跑完。
- 带宽是否稳定，是否明显低于同型号、同拓扑机器的正常水平。

最常用的测试工具不是手写程序，而是 NVIDIA 的 [nccl-tests](https://github.com/NVIDIA/nccl-tests)。

## 二、安装前准备

开始前至少确认下面几项：

- GPU 驱动正常，`nvidia-smi` 可用。
- CUDA 环境可用，宿主机可以执行 `nvcc -V`，或者容器内能正常调用 CUDA。
- 单机多卡节点如果是 A100/H800 SXM 这类 NVSwitch 机器，Fabric Manager 已启动。
- 多机测试时，MPI 已安装。
- 如果走 IB/RDMA，网卡驱动、OFED 或 RDMA 栈已安装，且现场网络本身可通。

补一句容易混淆的：`nvidia-smi` 顶部的 `CUDA Version` 表示驱动最高支持的 CUDA runtime 能力，不等于宿主机已经安装了这个版本的 CUDA Toolkit。NCCL 编译时真正要看的还是 `nvcc -V`、`CUDA_HOME` 和链接到的 NCCL 库。

先看几条最基础的信息：

```bash
nvidia-smi
nvidia-smi topo -m
ibstat
ip a
```

如果 `nvidia-smi topo -m` 已经显示出明显异常，比如所有卡都走 `SYS`、`PHB`，或者你预期有 NVLink 但拓扑里完全看不到 `NV#`，那就不建议直接开始跑 NCCL。

## 三、安装 NCCL

NVIDIA 官方安装文档把 NCCL 安装分成三类：

- Ubuntu
- RHEL/CentOS
- 其他发行版的 tar 包安装

这里的重点不是死记版本号，而是记住两条原则：

- NCCL 包要选择匹配当前 CUDA 版本的构建，并确保驱动分支满足这个 CUDA runtime 的最低要求。
- 如果只是运行上层训练框架，很多时候容器里已经自带 NCCL；但如果你要编译 `nccl-tests`，通常还需要安装开发包。

### 1. Ubuntu 安装

官方安装流程是先加 NVIDIA 仓库，再安装 `libnccl2` 和开发包 `libnccl-dev`。

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/<distro>/<architecture>/cuda-keyring_1.0-1_all.deb
sudo dpkg -i cuda-keyring_1.0-1_all.deb
sudo apt update
sudo apt install -y libnccl2 libnccl-dev
```

说明：

- `<distro>` 需要替换成你的发行版目录，比如 `ubuntu2204`。
- `<architecture>` 一般是 `x86_64`。
- 如果你不想让仓库顺带把 CUDA 升到最新，安装时要显式指定版本。

检查安装结果：

```bash
dpkg -l | grep nccl
ldconfig -p | grep nccl
```

### 2. RHEL / Rocky / Alma / CentOS 安装

官方文档的思路也是先加仓库，再安装运行库和开发包。

RHEL 8/兼容发行版常见写法：

```bash
sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
sudo dnf install -y libnccl libnccl-devel libnccl-static
```

如果你的系统是 RHEL 9、Rocky 9、AlmaLinux 9，仓库路径要改成对应版本目录。不要直接照抄 `rhel8`。

检查安装结果：

```bash
rpm -qa | grep nccl
ldconfig -p | grep nccl
```

### 3. 其他发行版或离线安装

如果你不走包管理器，也可以按官方文档使用 tar 包。典型流程是解压到 `/usr/local` 之类的位置，再在编译时显式指定 `NCCL_HOME`。

```bash
sudo tar -C /usr/local -xvf nccl-<version>.txz
```

如果用了 tar 包安装，后续编译 `nccl-tests` 时通常这样写：

```bash
make CUDA_HOME=/usr/local/cuda NCCL_HOME=/usr/local/nccl-<version>
```

## 四、安装后先做库级检查

装完 NCCL，不要马上跑大规模压测，先确认库、驱动和 GPU 状态都正常。

建议检查：

```bash
nvidia-smi
nvcc -V
ldconfig -p | grep nccl
nvidia-smi topo -m
```

如果是多机环境，再补两类检查：

```bash
ping <peer-node>
ssh <peer-node> hostname
ibstat
ibdev2netdev
```

目的很简单：先确认系统层没问题，再看 NCCL。

## 五、编译 nccl-tests

`nccl-tests` 是 NVIDIA 官方测试仓库，最常用的二进制就是 `all_reduce_perf`。

拉代码：

```bash
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests
```

### 1. 单机版本编译

如果只是做单机多卡测试，不依赖 MPI，直接编译：

```bash
make -j CUDA_HOME=/usr/local/cuda
```

如果 NCCL 不在系统默认路径 `/usr`，加上 `NCCL_HOME`：

```bash
make -j CUDA_HOME=/usr/local/cuda NCCL_HOME=/usr/local/nccl-<version>
```

### 2. 多机版本编译

多进程、多节点压测需要 MPI 支持。按 `nccl-tests` README 的方式编译：

```bash
make -j MPI=1 MPI_HOME=/path/to/mpi CUDA_HOME=/usr/local/cuda
```

如果你想把 MPI 版本和非 MPI 版本区分开，可以加后缀：

```bash
make -j MPI=1 NAME_SUFFIX=_mpi MPI_HOME=/path/to/mpi CUDA_HOME=/usr/local/cuda
```

生成后的可执行文件类似：

- `build/all_reduce_perf`
- `build/all_reduce_perf_mpi`
- `build/all_gather_perf`
- `build/reduce_scatter_perf`

## 六、先跑最小化单机测试

第一次测试建议从最小命令开始，先看程序能不能启动、识别 GPU、正常退出。

单卡最小验证：

```bash
./build/all_reduce_perf -b 8M -e 128M -f 2 -g 1
```

单机 8 卡示例：

```bash
./build/all_reduce_perf -b 8M -e 8G -f 2 -g 8
```

参数含义：

- `-b`：最小消息大小。
- `-e`：最大消息大小。
- `-f`：按倍数递增。
- `-g`：每个线程使用的 GPU 数。

更稳妥的压测写法通常会增加预热和正式迭代次数：

```bash
./build/all_reduce_perf -b 8M -e 8G -f 2 -g 8 -w 10 -n 50
```

这里：

- `-w 10` 表示先做 10 次预热。
- `-n 50` 表示正式跑 50 次。

## 七、看懂 nccl-tests 输出

`all_reduce_perf` 输出很多，现场一般先看这几项：

- 是否初始化成功。
- 是否报错，或者长时间卡住没有输出。
- `algbw` 是否稳定。
- `busbw` 是否稳定。
- `#wrong` 是否为 0。

日常判断里，`busbw` 往往比 `algbw` 更适合看通信效率。`#wrong` 不为 0 时，说明结果已经不正确，先别讨论性能。

判断结果时建议这样看：

- 同一台机器多跑几轮，结果是否波动过大。
- 同型号、同拓扑节点之间差距是否明显。
- 大消息段的带宽是否持续偏低。
- 是否只有某一张卡、某一台节点异常。

## 八、单机多卡常用测试命令

下面这些命令在现场最常用。

### 1. AllReduce

```bash
./build/all_reduce_perf -b 8M -e 8G -f 2 -g 8 -w 10 -n 50
```

这是训练场景里最常见的集合通信测试。

### 2. AllGather

```bash
./build/all_gather_perf -b 8M -e 8G -f 2 -g 8 -w 10 -n 50
```

可以补看非归约类集合通信的表现。

### 3. ReduceScatter

```bash
./build/reduce_scatter_perf -b 8M -e 8G -f 2 -g 8 -w 10 -n 50
```

一些分布式训练场景会用到这类通信模式。

### 4. Broadcast

```bash
./build/broadcast_perf -b 8M -e 1G -f 2 -g 8 -w 10 -n 50
```

适合补看从 root 广播到其他 GPU 的性能。

## 九、多机测试怎么跑

多机测试先想清楚进程和 GPU 怎么对应，不要一上来就堆很多环境变量。

常见做法是每个进程用 1 张 GPU，也就是：

- 每台机器 8 卡，就起 8 个进程。
- `-g 1`，让每个进程只负责 1 张卡。

两台 8 卡节点示例：

```bash
mpirun -np 16 -N 8 \
  -H node01:8,node02:8 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_SOCKET_IFNAME=eth0 \
  ./build/all_reduce_perf -b 8M -e 8G -f 2 -g 1 -w 10 -n 50
```

这里的意思是：

- 总共 16 个进程。
- 每台机器 8 个进程。
- 每个进程 1 张 GPU。

如果你的管理网络和训练网络不是同一张网卡，`NCCL_SOCKET_IFNAME` 一定要显式指定，不然 NCCL 可能会选错网卡。

### 1. IB / RoCE 常见写法

如果使用 RDMA，现场经常会先指定 HCA，并按需补充其他变量：

```bash
export NCCL_IB_DISABLE=0
export NCCL_IB_HCA=mlx5
```

然后执行：

```bash
mpirun -np 16 -N 8 \
  -H node01:8,node02:8 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_IB_DISABLE \
  -x NCCL_IB_HCA \
  ./build/all_reduce_perf -b 8M -e 8G -f 2 -g 1 -w 10 -n 50
```

注意：

- `NCCL_IB_HCA=mlx5` 是前缀匹配，不一定要写到具体端口。
- 更精确时可以写成 `=mlx5_0:1,mlx5_1:1` 这种形式。
- 按官方 Troubleshooting 文档，NCCL 2.21 及以后版本会动态选择 GID index，通常不需要手工设置 `NCCL_IB_GID_INDEX`。
- 如果你用的是更老版本，或厂商网络方案明确要求手工指定，再去执行 `show_gids` 并按现场结果设置。

### 2. 多网卡环境

如果是多 NIC 机器，是否允许跨网卡通信还会影响性能。官方文档里 `NCCL_CROSS_NIC` 的意思是：

- `0`：尽量固定同一条 rail。
- `1`：允许更自由地跨 NIC。
- `2`：默认值，自动折中。

如果你的网络是明显的 rail-optimized 设计，通常优先从默认值开始看，再决定是否手工干预。

## 十、常用环境变量速查

官方 User Guide 的环境变量章节很全，但现场不用一次记完。先把最常用的一批掌握住就够了。

| 变量                       | 作用                   | 常见用法                         |
| -------------------------- | ---------------------- | -------------------------------- |
| `NCCL_DEBUG`               | 控制日志级别           | `WARN`、`INFO`                   |
| `NCCL_DEBUG_SUBSYS`        | 过滤 `INFO` 日志子系统 | `INIT,BOOTSTRAP,ENV` 或 `NET`    |
| `NCCL_DEBUG_FILE`          | 把日志写到文件         | `NCCL_DEBUG_FILE=nccl.%h.%p.log` |
| `NCCL_SOCKET_IFNAME`       | 指定 IP 通信网卡       | `=eth0`、`ib`、`^docker`         |
| `NCCL_SOCKET_FAMILY`       | 强制 IPv4 或 IPv6      | `AF_INET`                        |
| `NCCL_IB_DISABLE`          | 禁用或启用 IB Verbs    | `0` 或 `1`                       |
| `NCCL_IB_HCA`              | 指定 RDMA 网卡         | `mlx5`、`=mlx5_0:1`              |
| `NCCL_IB_GID_INDEX`        | 指定 RoCE GID index    | 老版本或现场要求时再设           |
| `NCCL_SOCKET_NTHREADS`     | socket helper 线程数   | 100G 网络常见会试 `4`            |
| `NCCL_NSOCKS_PERTHREAD`    | 每线程 socket 数       | 100G 网络常见会试 `4`            |
| `NCCL_CROSS_NIC`           | 多 NIC 选择策略        | `0`、`1`、`2`                    |
| `NCCL_IGNORE_CPU_AFFINITY` | 忽略作业的 CPU 绑核    | `1`                              |

几个实践要点：

- `NCCL_SOCKET_IFNAME` 支持前缀匹配、排除匹配和精确匹配。
- 默认情况下，`lo` 和 `docker*` 不会优先被选中。
- `NCCL_SOCKET_NTHREADS * NCCL_NSOCKS_PERTHREAD` 不能超过 64，提高它们可能改善 socket 性能，也会增加 CPU 开销。
- 官方文档明确提醒，很多调试类变量不要长期固化到生产环境里。

### 1. 静态配置文件

环境变量除了临时 `export`，也可以写进配置文件。

系统级：

```bash
/etc/nccl.conf
```

从较新版本开始，也可以通过：

```bash
export NCCL_CONF_FILE=/path/to/custom-nccl.conf
```

`NCCL_CONF_FILE` 是 NCCL 2.23 开始支持的自定义配置文件入口；老版本还是用 `/etc/nccl.conf` 或临时环境变量。

示例：

```bash
NCCL_DEBUG=WARN
NCCL_SOCKET_IFNAME==eth0
```

注意这里的 `==eth0` 不是笔误。配置文件里的值本身就支持精确匹配语义。

## 十一、容器里跑 NCCL 的注意事项

现在很多测试是在容器里跑，不是在宿主机直接跑。这时最容易踩的是共享内存和锁页内存限制。

NVIDIA 官方 Troubleshooting 文档明确提到，Docker 默认共享内存和锁页内存限制通常不够，容易导致 NCCL 初始化失败。

常见启动参数：

```bash
docker run --gpus all --rm -it \
  --shm-size=1g \
  --ulimit memlock=-1 \
  <image> /bin/bash
```

NCCL 2.24 开始在满足 CUDA driver/runtime 条件时默认优先使用 cuMem host allocations；但老版本、容器 NUMA 能力不足或虚拟化场景仍可能回到 `/dev/shm` 这条路径。现场排查时还是建议把 `/dev/shm` 和 `memlock` 一起看。

如果容器里跑多机压测，还要额外确认：

- 容器能看到正确的网卡。
- `/sys` 已正确挂载，NCCL 需要它来发现 PCI 拓扑。
- RDMA 设备已透传。

## 十二、常见故障和排查思路

### 1. 初始化时报共享内存错误

典型现象：

- 程序启动后直接报错。
- `NCCL_DEBUG=WARN` 下看到 `/dev/shm` 扩容失败。

先查：

```bash
df -h /dev/shm
ulimit -l
```

如果在容器里，优先加大 `--shm-size`，并放开 `memlock`。

### 2. 多机测试一直卡住

常见原因：

- 选错网卡。
- 某张网卡虽然是 `UP`，但节点间不通。
- MPI 和 NCCL 走了不同网络。

先加日志：

```bash
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,BOOTSTRAP,NET
```

再显式指定网卡：

```bash
export NCCL_SOCKET_IFNAME='=eth0'
```

如果管理员已经给 NCCL 固定了网卡，MPI 最好也使用同一张网卡。

### 3. IB / RoCE 性能异常或根本没走 RDMA

先看：

```bash
ibstat
ibdev2netdev
lsmod | grep -E 'nvidia_peermem|nv_peer_mem'
```

NVIDIA 文档在 Troubleshooting 里提到，GPUDirect RDMA 依赖兼容的网卡驱动和额外内核模块 `nvidia-peermem`。如果系统满足 DMA-BUF 路径要求，NCCL 可以自动检测并启用 DMA-BUF，这时不一定需要 `nvidia-peermem`；否则缺这层时，多机通信可能退化到更慢路径。

### 4. 容器或虚拟机里拓扑识别异常

NCCL 依赖 `/sys` 去发现 GPU 和网卡拓扑。如果容器或虚拟机暴露的是不完整或虚拟化过的 PCI 拓扑，性能可能明显偏低。

优先检查：

```bash
nvidia-smi topo -m
mount | grep sysfs
```

### 5. 某些平台上的 PCI ACS 影响 GPUDirect

官方 Troubleshooting 文档也提到，PCI ACS 可能影响 GPU Direct 路径，尤其是虚拟化环境里更要注意。物理机环境如果性能异常且确认拓扑无误，可以再往这条线排查；虚拟机场景则通常不能简单粗暴地关闭 ACS。

## 十三、一套更实用的测试顺序

现场做 NCCL 测试，我更建议按下面顺序走：

1. `nvidia-smi`、`nvidia-smi topo -m`、`ibstat` 做静态检查。
2. 跑单卡 `all_reduce_perf`，确认程序能启动，库也能加载。
3. 跑单机多卡 `all_reduce_perf`，确认机内互联正常。
4. 跑单机 `all_gather_perf` 和 `reduce_scatter_perf`，补看不同集合通信模式。
5. 跑双机小规模测试，确认多机链路和网卡选择无误。
6. 最后再跑完整规模和更大消息量压测。

这样做的好处是，一旦出问题，可以更快判断大概卡在哪一层：

- GPU 本身
- 机内 NVLink / PCIe
- MPI 启动层
- 机间网络
- RDMA 栈
- 容器资源限制

## 十四、常用命令速查

单机 8 卡 AllReduce：

```bash
./build/all_reduce_perf -b 8M -e 8G -f 2 -g 8 -w 10 -n 50
```

双机 16 卡 AllReduce：

```bash
mpirun -np 16 -N 8 -H node01:8,node02:8 \
  -x NCCL_DEBUG=INFO \
  -x NCCL_SOCKET_IFNAME=eth0 \
  ./build/all_reduce_perf -b 8M -e 8G -f 2 -g 1 -w 10 -n 50
```

只看网络相关日志：

```bash
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=NET
```

固定 RDMA 网卡：

```bash
export NCCL_IB_HCA='=mlx5_0:1'
```

固定 socket 网卡：

```bash
export NCCL_SOCKET_IFNAME='=eth0'
```

查看 NCCL 库：

```bash
ldconfig -p | grep nccl
```

查看 GPU 拓扑：

```bash
nvidia-smi topo -m
```

## 十五、参考资料

- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/index.html)
- [NVIDIA NCCL Installation Guide](https://docs.nvidia.com/deeplearning/nccl/index.html)
- [NVIDIA NCCL Environment Variables](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)
- [NVIDIA NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [NVIDIA nccl-tests](https://github.com/NVIDIA/nccl-tests)

## 十六、小结

NCCL 测试的关键不是把命令抄出来，而是把链路分层看清楚。先确认驱动、CUDA、拓扑、网卡和共享内存，再做 `nccl-tests`，结果会更可靠。

如果只记一条经验，就是先从最小规模跑起来，再逐步放大到单机多卡、双机多卡和全量集群。这样排障成本最低，也更符合现场节奏。
