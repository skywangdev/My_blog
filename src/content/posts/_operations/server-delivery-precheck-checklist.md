---
title: "服务器交付前检查清单：硬件、系统、网络和 GPU"
author: "Sky Wang"
pubDatetime: 2023-02-18T09:30:00+08:00
featured: true
draft: false
tags:
  - 服务器
  - Linux
  - 运维
  - 交付
description: "记录服务器交付前常做的检查项，包括硬件识别、RAID、系统版本、网络、日志、GPU 和交付文档。"
---

服务器交付不是把系统装上就结束。真正交付前，至少要确认硬件、系统、网络、存储、驱动和日志都没有明显问题。否则机器到了客户现场或进入业务池后，再排查会很被动。

## Table of contents

## 一、先确认硬件清单

拿到服务器后，先按配置单核对硬件：

- 服务器型号。
- CPU 型号和数量。
- 内存容量、条数和插槽位置。
- 硬盘数量、容量和类型。
- RAID 卡、HBA、网卡、GPU 等扩展卡。
- 电源数量和功率。

Linux 下可以先看：

```bash
lscpu
free -h
lsblk
lspci
dmidecode -t memory
dmidecode -t system
```

如果配置单写了 8 块盘，系统只看到 7 块，不要继续装系统。先确认硬盘、背板、线缆、RAID 卡和 BMC 告警。

## 二、检查 BIOS 和 BMC

交付前建议记录 BIOS 和 BMC 版本：

```bash
dmidecode -t bios
ipmitool mc info
```

需要关注：

- BIOS 版本是否符合项目要求。
- BMC/IPMI/iDRAC 是否能登录。
- 管理口 IP 是否配置正确。
- 风扇、电源、温度是否有告警。
- GPU 服务器是否开启 Above 4G Decoding。
- 启动模式是 UEFI 还是 Legacy。

BMC 是后续远程排障的入口，交付时不能只看系统能启动。

## 三、检查 RAID 和硬盘

如果使用 RAID 卡，先确认虚拟磁盘和物理盘状态。不同厂商工具不同，常见有：

```bash
storcli /call show
perccli /call show
megacli -AdpAllInfo -aALL
```

至少记录：

- RAID 级别。
- 成员盘数量。
- 虚拟磁盘容量。
- 初始化状态。
- 物理盘是否有 `Failed`、`Rebuild`、`Predictive Failure`。

系统盘常见用 RAID1，数据盘按业务选择 RAID5、RAID6 或 RAID10。不要把 RAID 当备份，交付文档里最好明确写清楚。

## 四、系统安装后检查

系统装完后，不要只看能登录。建议统一检查：

```bash
cat /etc/os-release
uname -r
hostname
ip a
df -h
lsblk
timedatectl
```

再看基础服务：

```bash
systemctl --failed
journalctl -p err -b
dmesg | tail -n 100
```

如果 `systemctl --failed` 里有失败服务，要在交付前处理或记录原因。

## 五、网络检查

网络至少确认三件事：

- IP、网关、DNS 配置是否正确。
- 业务网、管理网是否接到正确端口。
- 节点之间是否能互通。

常用命令：

```bash
ip a
ip route
cat /etc/resolv.conf
ping <gateway>
ping <peer-node>
ethtool <nic>
```

如果是多网卡服务器，还要记录网卡名和物理端口对应关系。后面做 PXE、NCCL、存储网络或业务绑定时，这个信息很有用。

## 六、GPU 服务器额外检查

GPU 服务器交付时，除了普通服务器检查，还要看：

```bash
nvidia-smi
nvidia-smi topo -m
lsmod | grep nvidia
dmesg | grep -i xid
```

如果是 A100/H800 SXM 这类机器，还要确认 Fabric Manager：

```bash
systemctl status nvidia-fabricmanager
```

GPU 交付不能只看 `nvidia-smi` 有输出，还要看驱动版本、GPU 数量、拓扑、温度、功耗和错误日志。

## 七、基础压力测试

交付前可以做轻量测试，不一定每台都跑很长时间，但至少要发现明显问题。

磁盘：

```bash
fio --name=test --filename=/tmp/fio.test --size=4G --rw=readwrite --bs=1M --numjobs=1 --direct=1
```

网络：

```bash
iperf3 -s
iperf3 -c <server-ip>
```

GPU：

```bash
./deviceQuery
./bandwidthTest
```

正式项目里测试项要按交付标准来，不要临时想起什么跑什么。

## 八、交付文档建议记录什么

每台机器建议至少记录：

- 资产编号和序列号。
- CPU、内存、硬盘、网卡、GPU 配置。
- BIOS、BMC、RAID 卡固件版本。
- 系统版本和内核版本。
- IP 地址、主机名和网卡用途。
- RAID 信息。
- 驱动、CUDA、Docker 等软件版本。
- 检查命令和测试结果。
- 遗留问题和处理说明。

交付文档不是形式。后续出问题时，最有用的往往就是这些版本和测试记录。

## 九、小结

服务器交付前检查的核心是把问题尽量留在交付前。硬件、系统、网络、存储、GPU 和日志都看一遍，虽然会多花一点时间，但比交付后返修或远程排障要划算得多。
