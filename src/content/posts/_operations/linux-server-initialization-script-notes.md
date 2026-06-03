---
title: "Linux 服务器初始化脚本一般做哪些事"
author: "Sky Wang"
pubDatetime: 2023-08-20T15:30:00+08:00
featured: false
draft: false
tags:
  - Linux
  - 服务器
  - Shell
  - 运维
description: "记录 Linux 服务器装完系统后常做的初始化动作，包括主机名、时间、软件源、SSH、用户、基础工具和日志检查。"
---

批量交付服务器时，系统装完只是第一步。后面还要做主机名、网络、时间、软件源、用户、SSH、基础工具和日志检查。很多动作可以整理成初始化脚本，但脚本要简单、明确、可回滚。

## Table of contents

## 一、初始化脚本的原则

我的习惯是把初始化脚本分成几类：

- 必须执行的基础项。
- 按项目选择执行的配置项。
- 只检查不修改的巡检项。

不要把所有东西都写成一个巨大脚本。脚本越大，越不容易排查。尤其是生产服务器，改系统配置前要知道每一步在做什么。

## 二、主机名和时间

主机名最好按项目规范统一：

```bash
hostnamectl set-hostname node01
```

时间同步也要尽早处理：

```bash
timedatectl
timedatectl set-timezone Asia/Shanghai
```

如果用 chrony：

```bash
systemctl enable --now chronyd
chronyc sources -v
```

时间不准会影响日志、证书、集群调度和很多认证服务。

## 三、软件源和基础包

先确认系统版本：

```bash
cat /etc/os-release
uname -r
```

基础包可以按系统选择安装：

RHEL、Rocky、AlmaLinux：

```bash
dnf install -y vim lsof wget curl tar unzip net-tools pciutils ipmitool
```

Ubuntu：

```bash
apt update
apt install -y vim lsof wget curl tar unzip net-tools pciutils ipmitool
```

如果项目要求内网源，要先配置软件源，再安装软件。不要让不同节点各自从不同源安装，后面版本会不一致。

## 四、用户和 SSH

交付服务器一般要处理：

- 管理用户。
- SSH 登录策略。
- sudo 权限。
- 密钥登录。
- root 是否允许远程登录。

查看 SSH 配置：

```bash
sshd -T | less
```

修改后重启：

```bash
systemctl restart sshd
```

远程修改 SSH 时要小心，建议保留一个已登录窗口，确认新连接没问题后再退出。

## 五、网络检查

初始化脚本可以做检查，但不建议盲目改网卡配置。至少输出：

```bash
ip a
ip route
cat /etc/resolv.conf
```

如果需要检查连通性：

```bash
ping -c 3 <gateway>
ping -c 3 <dns-server>
```

多网卡服务器最好记录网卡名、MAC 地址和用途：

```bash
ip -br link
ethtool <nic>
```

## 六、系统限制和基础安全

按项目需要设置：

- 文件句柄限制。
- 防火墙策略。
- SELinux 状态。
- 登录超时。
- 密码策略。

查看：

```bash
ulimit -n
getenforce
systemctl status firewalld
```

这些不要随便“一刀切”关闭。比如防火墙和 SELinux，有些项目可以关，有些项目必须按策略配置。

## 七、磁盘和挂载

初始化时建议检查：

```bash
lsblk
blkid
df -h
mount
```

如果要写 `/etc/fstab`，建议用 UUID，不要只写 `/dev/sdb1`。设备名在重启或换盘后可能变化。

写完后先测试：

```bash
mount -a
```

这一步很重要。`fstab` 写错可能导致服务器重启后进不了系统。

## 八、日志和失败服务

初始化完成后，看一遍系统状态：

```bash
systemctl --failed
journalctl -p err -b
dmesg | tail -n 100
```

如果是交付节点，最好把这些输出保存下来，作为验收记录的一部分。

## 九、一个简单脚本框架

脚本可以先写得很简单：

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "[1/5] system info"
cat /etc/os-release
uname -r

echo "[2/5] time"
timedatectl

echo "[3/5] disk"
lsblk
df -h

echo "[4/5] network"
ip -br a
ip route

echo "[5/5] services"
systemctl --failed || true
journalctl -p err -b --no-pager | tail -n 50 || true
```

先让脚本会检查，再逐步增加配置动作。这样更稳。

## 十、小结

Linux 初始化脚本不是为了炫技，而是为了减少重复操作和配置差异。真正重要的是：每一步都明确、日志能保存、出问题能回退。批量交付时，这比写一个复杂脚本更有价值。
