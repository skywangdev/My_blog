---
title: "服务器 RAID 基础与 Dell 阵列配置实践"
author: "Sky Wang"
pubDatetime: 2020-07-23T12:00:00+08:00
featured: false
draft: false
tags:
  - 服务器
  - RAID
  - Dell
description: "把 RAID 基本概念、常见级别选择，以及 Dell 服务器上创建 RAID1 和 RAID5 的操作整理成一篇可查的实践笔记。"
---


这篇是旧博客里两篇 RAID 笔记的合并版。一篇讲概念，一篇记录 Dell 服务器上的实际配置。单独看都偏碎，合在一起更适合日后查阅：先判断该用哪种 RAID，再进入 Dell 配置界面创建虚拟磁盘。

## Table of contents

## RAID 是什么

RAID，全称是 Redundant Array of Independent Disks，中文通常叫“独立磁盘冗余阵列”。它把多块物理硬盘组合成一个或多个逻辑磁盘，让操作系统看起来像是在使用一块盘。

RAID 通常有三个目标：

- **提升性能：** 多块盘同时读写，吞吐能力高于单盘。
- **增加可用性：** 通过镜像或校验，即使坏一块盘，也可以继续运行。
- **扩大容量：** 把多块盘组合成一个更大的逻辑空间，便于管理。

但有一句话必须放在最前面：

> RAID 不是备份。RAID 解决的是单盘故障时服务不中断，不解决误删除、勒索病毒、火灾、机房事故和人为操作失误。

如果数据重要，正确策略应该是：

```txt
RAID + 定期备份 + 异地副本
```

## 常见 RAID 级别怎么选

### RAID 0：只要性能，不要安全

RAID 0 会把数据分块后写入多块硬盘。

- 最少硬盘：2 块
- 可用容量：N 块盘容量之和
- 优点：读写性能高
- 缺点：任意一块盘坏，全部数据丢失
- 适合：缓存盘、临时素材盘、可重建的数据

### RAID 1：两块盘互为镜像

RAID 1 会把同一份数据写入两块盘。

- 最少硬盘：2 块
- 可用容量：约 50%
- 优点：安全性高，适合系统盘
- 缺点：空间利用率低
- 适合：操作系统盘、关键启动盘、小容量重要数据盘

### RAID 5：容量和安全的折中

RAID 5 使用分布式奇偶校验，可以容忍任意一块盘故障。

- 最少硬盘：3 块
- 可用容量：`(N - 1) * 单盘容量`
- 优点：容量利用率较好，读性能不错
- 缺点：写入有校验开销，重建期间风险高
- 适合：中小型文件服务器、一般数据存储

如果硬盘容量很大，RAID 5 重建时间可能很长，期间再坏一块盘就会丢数据。现在大容量机械盘场景下，要谨慎使用 RAID 5。

### RAID 6：能同时坏两块盘

RAID 6 类似 RAID 5，但有双重校验。

- 最少硬盘：4 块
- 可用容量：`(N - 2) * 单盘容量`
- 优点：可容忍两块盘故障
- 缺点：写入性能更差，容量利用率低于 RAID 5
- 适合：大容量硬盘阵列、归档存储、对安全性要求更高的 NAS

### RAID 10：性能和可靠性都要

RAID 10 是先做 RAID 1 镜像，再做 RAID 0 条带。

- 最少硬盘：4 块，且通常为偶数
- 可用容量：约 50%
- 优点：性能好，重建快，可靠性高
- 缺点：成本高
- 适合：数据库、虚拟化主机、高负载业务

## 常见场景建议

| 场景 | 推荐方案 | 原因 |
| --- | --- | --- |
| 系统盘 | RAID 1 | 容量需求不大，可靠性优先 |
| 小型文件服务器 | RAID 5 | 容量利用率和安全性较平衡 |
| 大容量 NAS | RAID 6 | 重建时间长，双校验更稳 |
| 数据库或虚拟化 | RAID 10 | 性能、可靠性和重建速度都更好 |
| 临时缓存 | RAID 0 | 只追求速度，数据可丢 |

## Dell 服务器配置目标

旧文中的需求是：

- 前 2 块 480GB SSD 做 RAID 1，用来安装系统。
- 后 4 块 4TB 机械盘做 RAID 5，用来存数据。

这个方案很常见：系统盘追求稳定，数据盘追求容量和一定容错。

<figure>
  <img src="https://data.skywangdev.com/blog/S-8.jpeg" alt="Dell 服务器 RAID 配置封面图" />
  <figcaption class="text-center">Dell 服务器 RAID 配置记录：2 块 SSD 做 RAID1，4 块机械盘做 RAID5。</figcaption>
</figure>

## 进入 Dell RAID 配置界面

开机后看到 Dell 启动画面，按 `F2` 进入 `System Setup`。

<figure>
  <img src="https://yan-jian.com/file/20231208/1.png" alt="Dell 开机按 F2 进入 System Setup" />
  <figcaption class="text-center">开机后按 F2 进入 System Setup。</figcaption>
</figure>

进入后选择 `Device Settings`。

<figure>
  <img src="https://yan-jian.com/file/20231208/2.png" alt="Dell System Setup 中的 Device Settings" />
  <figcaption class="text-center">进入 Device Settings。</figcaption>
</figure>

找到类似 `Integrated RAID Controller 1: Dell Configuration Utility` 的入口。

<figure>
  <img src="https://yan-jian.com/file/20231208/3.png" alt="Dell Integrated RAID Controller 配置入口" />
  <figcaption class="text-center">这里就是 RAID 阵列卡配置入口。</figcaption>
</figure>

旧文里使用的是 H740P Mini 阵列卡。它带有缓存，用于提升读写效率。

<figure>
  <img src="https://yan-jian.com/file/20231208/4.png" alt="Dell H740P Mini RAID 阵列卡" />
  <figcaption class="text-center">H740P Mini RAID 阵列卡。</figcaption>
</figure>

## 检查物理硬盘状态

进入 `Main Menu` 后，先看物理盘是否都处于 `Ready` 状态。

<figure>
  <img src="https://yan-jian.com/file/20231208/6.png" alt="Dell RAID Main Menu" />
  <figcaption class="text-center">进入 RAID 配置主菜单。</figcaption>
</figure>

选择 `Physical Disk Management`。

<figure>
  <img src="https://yan-jian.com/file/20231208/7.png" alt="Physical Disk Management 入口" />
  <figcaption class="text-center">先确认物理盘状态，不要急着创建阵列。</figcaption>
</figure>

确认 2 块 SSD 和 4 块机械盘都已经 Ready。

<figure>
  <img src="https://yan-jian.com/file/20231208/8.png" alt="Dell RAID 物理磁盘 Ready 状态" />
  <figcaption class="text-center">所有硬盘处于 Ready 状态后，再创建虚拟磁盘。</figcaption>
</figure>

## 创建 RAID 1 系统盘

返回主菜单，选择 `Configuration Management`。

<figure>
  <img src="https://yan-jian.com/file/20231208/9.png" alt="Dell Configuration Management" />
  <figcaption class="text-center">进入 Configuration Management。</figcaption>
</figure>

选择 `Create Virtual Disk`。

<figure>
  <img src="https://yan-jian.com/file/20231208/10.png" alt="Create Virtual Disk 创建虚拟磁盘" />
  <figcaption class="text-center">创建新的虚拟磁盘。</figcaption>
</figure>

`Select RAID Level` 选择 `RAID1`。

<figure>
  <img src="https://yan-jian.com/file/20231208/11.png" alt="选择 RAID1" />
  <figcaption class="text-center">系统盘选择 RAID1。</figcaption>
</figure>

选择前两块 480GB SSD。通常使用方向键选中硬盘，按空格勾选，然后 `Apply Changes`。

<figure>
  <img src="https://yan-jian.com/file/20231208/14.png" alt="选择两块 SSD 创建 RAID1" />
  <figcaption class="text-center">选择两块 SSD 作为 RAID1 成员盘。</figcaption>
</figure>

## RAID 参数怎么理解

创建虚拟磁盘时会看到一些参数，常见选择如下。

| 参数 | 建议 | 说明 |
| --- | --- | --- |
| Virtual Disk Name | 自定义 | 建议命名为 `OS_RAID1`、`DATA_RAID5` 这类可读名称 |
| Virtual Disk Size | 默认全部空间 | 一般直接使用全部容量 |
| Strip Element Size | 默认或按业务调整 | 大文件可用较大条带，小文件/数据库可用较小条带 |
| Read Policy | Read Ahead | 顺序读较多时可提升性能 |
| Write Policy | Write Back | 有电池/缓存保护时性能更好 |
| Disk Cache | Default | 通常保持默认 |
| Initialization | Fast | 系统安装场景可快速初始化 |

关于 `Write Back` 要注意：如果 RAID 卡电池或超级电容异常，策略可能自动降级为 `Write Through`，性能会下降。不要轻易选择 `Force Write Back`，否则异常断电时有数据风险。

<figure>
  <img src="https://yan-jian.com/file/20231208/16.png" alt="Dell RAID 虚拟磁盘参数配置" />
  <figcaption class="text-center">RAID 虚拟磁盘参数配置界面。</figcaption>
</figure>

确认参数后勾选 `Confirm`，点击 `Yes` 创建。

<figure>
  <img src="https://yan-jian.com/file/20231208/17.png" alt="确认创建 RAID1" />
  <figcaption class="text-center">确认创建 RAID1。</figcaption>
</figure>

## 创建 RAID 5 数据盘

RAID1 创建完成后，继续创建 RAID5。`Select RAID Level` 选择 `RAID5`。

<figure>
  <img src="https://yan-jian.com/file/20231208/19.png" alt="选择 RAID5" />
  <figcaption class="text-center">数据盘选择 RAID5。</figcaption>
</figure>

选择剩下 4 块 4TB 机械盘，然后应用。

<figure>
  <img src="https://yan-jian.com/file/20231208/21.png" alt="选择 4 块机械盘创建 RAID5" />
  <figcaption class="text-center">选择 4 块机械盘作为 RAID5 成员盘。</figcaption>
</figure>

确认参数后创建虚拟磁盘。

<figure>
  <img src="https://yan-jian.com/file/20231208/24.png" alt="确认创建 RAID5" />
  <figcaption class="text-center">确认创建 RAID5。</figcaption>
</figure>

## 创建完成后怎么检查

回到主菜单，进入 `Configuration Management`，选择 `View Disk Group Properties`。

<figure>
  <img src="https://yan-jian.com/file/20231208/28.png" alt="View Disk Group Properties 查看磁盘组属性" />
  <figcaption class="text-center">查看磁盘组属性。</figcaption>
</figure>

可以看到两个 Disk Group：一个 RAID1，一个 RAID5。

<figure>
  <img src="https://yan-jian.com/file/20231208/29.png" alt="RAID1 和 RAID5 创建完成" />
  <figcaption class="text-center">两个 RAID 组创建完成。</figcaption>
</figure>

如果要看初始化进度，进入 `Virtual Disk Management`。

<figure>
  <img src="https://yan-jian.com/file/20231208/31.png" alt="查看 RAID 初始化进度" />
  <figcaption class="text-center">如果选择 Full Initialization，需要等待初始化完成。</figcaption>
</figure>

## 实操检查清单

创建 RAID 前，建议按这个清单走一遍：

1. 确认硬盘数量、容量、槽位和用途。
2. 确认哪些盘做系统盘，哪些盘做数据盘。
3. 确认所有物理盘状态是 Ready。
4. 创建 RAID 后检查 Disk Group 和 Virtual Disk。
5. 记录 RAID 级别、成员盘、容量和用途。
6. 上系统前确认启动顺序和安装目标盘。
7. 配置备份策略，不要把 RAID 当备份。

## 小结

这套配置的思路是：

- 2 块 SSD 做 RAID1，保证系统盘可靠。
- 4 块机械盘做 RAID5，兼顾容量和单盘容错。
- 如果机械盘容量更大、重建时间更长，数据盘更建议考虑 RAID6。

RAID 配置本身并不复杂，真正容易出错的是：没确认盘位、选错成员盘、误以为 RAID 等于备份。尤其是线上服务器，创建或重建阵列前一定要确认数据和备份状态。
