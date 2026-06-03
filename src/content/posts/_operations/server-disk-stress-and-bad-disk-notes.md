---
title: "服务器硬盘压力测试和坏盘排查记录"
author: "Sky Wang"
pubDatetime: 2024-04-12T09:40:00+08:00
featured: false
draft: false
tags:
  - 服务器
  - 硬盘
  - RAID
  - 运维
description: "记录服务器交付和运维中检查硬盘状态、做压力测试、查看 SMART 信息和排查坏盘的常用方法。"
---

服务器交付时，硬盘问题很常见：系统识别不到盘、RAID 卡里有坏盘、阵列重建、SMART 告警、压力测试报错。这里整理一些常用检查方法。

## Table of contents

## 一、先确认系统看到哪些盘

基础命令：

```bash
lsblk
fdisk -l
blkid
df -h
```

看块设备时，要区分：

- 物理盘。
- RAID 虚拟盘。
- 系统盘。
- 数据盘。
- NVMe 盘。

如果服务器使用硬件 RAID，操作系统通常只看到 RAID 后的虚拟磁盘，不一定能直接看到每块物理盘。

## 二、看 RAID 卡状态

不同 RAID 卡工具不同，常见有：

```bash
storcli /call show
perccli /call show
megacli -PDList -aALL
```

重点看：

- 物理盘是否 Online。
- 是否有 Failed。
- 是否有 Predictive Failure。
- 是否在 Rebuild。
- 虚拟盘是否 Optimal。

如果阵列不是 Optimal，先不要继续做交付。

## 三、看 SMART 信息

如果系统能直接访问物理盘，可以用：

```bash
smartctl -a /dev/sda
smartctl -a /dev/nvme0n1
```

如果在 RAID 卡后面，可能需要指定设备类型，比如：

```bash
smartctl -a -d megaraid,0 /dev/sda
```

常看的字段：

- Reallocated Sector Count。
- Current Pending Sector。
- Offline Uncorrectable。
- Power On Hours。
- Media Error Count。
- Wear Leveling Count。

不同厂商字段不完全一样，不能只看一个值。最重要的是结合 RAID 卡告警、系统日志和压力测试结果一起判断。

## 四、看系统日志

硬盘问题经常会在日志里留下痕迹：

```bash
dmesg | grep -iE "error|fail|ata|scsi|nvme"
journalctl -k | grep -iE "error|fail|ata|scsi|nvme"
```

如果看到大量 I/O error、timeout、reset，要重点检查硬盘、背板、线缆、RAID 卡和供电。

## 五、fio 做压力测试

交付前可以用 `fio` 做简单测试。注意不要在已有数据盘上直接写破坏性测试。

测试临时文件：

```bash
fio --name=disk-test \
  --filename=/tmp/fio.test \
  --size=4G \
  --rw=readwrite \
  --bs=1M \
  --numjobs=1 \
  --direct=1 \
  --iodepth=16
```

如果要测试数据盘，先确认路径和数据风险：

```bash
fio --name=data-test \
  --directory=/data \
  --size=10G \
  --rw=randrw \
  --bs=4k \
  --numjobs=4 \
  --direct=1 \
  --iodepth=32
```

测试时同时看日志：

```bash
dmesg -w
```

如果压力测试期间出现 I/O error，基本不能忽略。

## 六、badblocks 适合什么时候用

`badblocks` 可以检查坏块，但耗时长。交付大量服务器时，不一定每块盘都跑完整坏块扫描。

只读检查：

```bash
badblocks -sv /dev/sdX
```

破坏性写入测试风险很高，除非确认盘里没有数据：

```bash
badblocks -wsv /dev/sdX
```

线上或有数据的盘不要随便跑破坏性测试。

## 七、坏盘处理思路

如果确认有坏盘：

1. 记录硬盘槽位、序列号和告警信息。
2. 确认 RAID 级别和当前阵列状态。
3. 判断是否允许热插拔。
4. 更换硬盘。
5. 观察 Rebuild 进度。
6. Rebuild 完成后再次检查虚拟盘状态。

RAID5、RAID6 重建期间风险较高。尤其是大容量机械盘，重建时间长，期间不要随意做高强度写入。

## 八、交付时记录什么

建议记录：

- 硬盘型号、容量和数量。
- RAID 级别。
- 虚拟盘容量。
- 物理盘状态。
- SMART 或 RAID 卡告警。
- fio 测试结果。
- 是否有 I/O error。

这些记录可以作为后续质保、返修和故障追溯依据。

## 九、小结

硬盘排查不能只看系统里有没有盘。RAID 卡状态、SMART、系统日志和压力测试要一起看。交付前发现问题及时处理，比上线后遇到阵列降级要稳得多。
