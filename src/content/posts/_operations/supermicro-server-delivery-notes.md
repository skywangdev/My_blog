---
title: "Supermicro 服务器交付记录：硬件装配、BIOS、BMC 和系统安装"
author: "Sky Wang"
pubDatetime: 2023-04-15T14:20:00+08:00
featured: false
draft: false
tags:
  - 服务器
  - Supermicro
  - Linux
  - 交付
description: "记录 Supermicro 服务器交付时常做的硬件装配、BIOS 检查、BMC 配置、RAID 和系统安装步骤。"
---

这篇记录 Supermicro 服务器交付时的一些常规步骤。不同型号界面会有差异，但大方向差不多：先把硬件和管理口处理好，再做 BIOS、存储和系统安装。

## Table of contents

## 一、硬件装配先按清单来

装机前先核对配置单：

- CPU 型号和数量。
- 内存容量、频率、条数和插槽位置。
- 硬盘类型和数量。
- RAID 卡、HBA、网卡、GPU 等扩展卡。
- 电源模块数量。
- 导轨、线缆、挡板和配件。

内存插槽一定要按主板手册来。多 CPU 服务器尤其要注意内存通道，插错虽然可能能开机，但性能和稳定性都会受影响。

## 二、第一次上电看什么

第一次上电先不要急着装系统。先观察：

- 风扇是否异常高转。
- 前面板是否有告警灯。
- BMC Web 是否能打开。
- BIOS 是否能进入。
- 硬盘背板和硬盘灯是否正常。

如果机器刚上电风扇一直满速，常见原因是 BMC 没正常读取传感器、固件异常、风扇策略不对，或者某些硬件未识别。

## 三、配置 BMC

Supermicro 的 BMC 是远程管理入口。交付前至少要配置：

- 管理口 IP。
- 管理账号密码。
- NTP 时间。
- 远程控制台是否可用。
- 传感器和事件日志是否有告警。

常用检查：

```bash
ipmitool lan print
ipmitool sensor
ipmitool sel list
```

如果 BMC 事件日志里有历史告警，建议先确认是否和当前硬件有关。处理完成后可以清理日志，但交付文档里要说明。

## 四、BIOS 里常看的项目

进入 BIOS 后，一般看这些：

- CPU 是否全部识别。
- 内存容量是否正确。
- 启动模式是 UEFI 还是 Legacy。
- 启动顺序。
- 是否开启 Above 4G Decoding。
- PCIe 插槽设备是否识别。
- 风扇模式和电源策略。

GPU 服务器通常要开启 Above 4G Decoding。否则多卡或大显存 GPU 可能出现资源分配问题。

## 五、存储和 RAID

如果服务器带 RAID 卡，先在 RAID 配置界面里看物理盘状态。常见目标是：

- 系统盘做 RAID1。
- 数据盘按项目要求做 RAID5、RAID6 或 RAID10。
- 所有物理盘状态正常。
- 虚拟磁盘初始化正常。

系统安装前要确认 BIOS 能看到目标启动盘。很多安装失败不是系统问题，而是 RAID 虚拟盘没创建好，或者启动模式不匹配。

## 六、安装系统

系统安装可以用 ISO、PXE 或批量部署平台。手工安装时，建议统一记录：

- 系统版本。
- 分区方式。
- 主机名。
- IP 地址。
- 管理账号。
- 软件源。

Linux 安装完成后先看：

```bash
cat /etc/os-release
uname -r
lsblk
ip a
systemctl --failed
```

不要只看能进系统。失败服务、磁盘识别、网卡名、时间同步都要确认。

## 七、驱动和工具

按项目需求安装：

- 网卡驱动。
- RAID 管理工具。
- GPU 驱动。
- IPMI 工具。
- 常用排查工具。

常用包：

```bash
dnf install -y pciutils lsof vim net-tools ipmitool
apt install -y pciutils lsof vim net-tools ipmitool
```

如果是 GPU 服务器，再安装 NVIDIA 驱动、CUDA 或 Fabric Manager，并用 `nvidia-smi` 确认。

## 八、交付前检查

最后做一次简单检查：

```bash
lscpu
free -h
lsblk
lspci
ip a
df -h
dmesg | tail -n 100
journalctl -p err -b
```

BMC 侧也要看：

```bash
ipmitool sensor
ipmitool sel list
```

如果有硬件告警、温度异常、风扇异常、硬盘状态异常，建议先处理，不要带着问题交付。

## 九、小结

Supermicro 服务器交付的重点是硬件和管理口先稳定，再进入系统安装。BIOS、BMC、RAID、系统版本和日志都要留下记录。这样后续出现问题时，能快速判断是交付前遗留，还是上线后新产生的问题。
