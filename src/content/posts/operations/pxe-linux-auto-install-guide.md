---
title: "Ubuntu 上配置 PXE 自动安装 Linux 的实践笔记"
author: "Sky Wang"
pubDatetime: 2021-07-20T12:00:00+08:00
modDatetime: 2026-06-03T01:20:00+08:00
featured: false
draft: false
tags:
  - Linux
  - PXE
  - 自动化部署
description: "记录在 Ubuntu 20.04 上搭建 PXE 服务，用 DHCP、TFTP、Nginx 和 Kickstart 批量安装 RHEL、Kylin 等系统的过程。"
---

<figure>
  <img src="https://data.skywangdev.com/blog/S-4.jpeg" alt="PXE 自动安装 Linux 封面图" />
  <figcaption class="text-center">PXE 适合批量安装服务器，核心是 DHCP 指路、TFTP 拉启动文件、HTTP 提供系统源和 Kickstart。</figcaption>
</figure>

这篇笔记基于 Ubuntu 20.04 搭建 PXE 环境，目标是批量安装 RHEL 7.9、RHEL 8.x、Kylin Server V10 等系统。旧文偏命令堆叠，这里补一层结构说明，方便以后复用。

## Table of contents

## PXE 流程先理清

PXE 自动安装大致分四步：

1. 客户端开机，从网卡启动。
2. DHCP 分配 IP，并告诉客户端 TFTP 服务器地址和启动文件名。
3. 客户端通过 TFTP 下载启动文件、内核和 initrd。
4. 安装程序通过 HTTP 访问系统源和 Kickstart 文件，完成无人值守安装。

对应服务：

| 服务 | 用途 |
| --- | --- |
| DHCP | 给客户端分配 IP，并告诉它去哪下载启动文件 |
| TFTP | 提供 PXE 启动文件、内核、initrd |
| Nginx | 提供系统安装源和 Kickstart 文件 |
| Kickstart | 定义安装选项，比如分区、软件包、root 密码 |

## 安装所需软件

```bash
apt update
apt install -y tftpd-hpa isc-dhcp-server nginx
```

建议 PXE 服务器使用固定 IP。旧文使用：

```txt
10.0.0.1
```

后面的 DHCP、Kickstart、TFTP 配置都以这个地址为例。

## 准备系统 ISO

旧环境里准备过这些镜像：

```txt
rhel-server-7.9-x86_64-dvd.iso
rhel-8.3-x86_64-dvd.iso
Kylin-Server-10-SP1-Release-Build20-20210518-x86_64.iso
Kylin-Server-10-SP2-x86-Release-Build09-20210524.iso
```

镜像文件建议统一放到一个目录，例如：

```bash
mkdir -p /data/iso
```

## 制作本地 HTTP 源

以 RHEL 7.9 为例：

```bash
mkdir -p /mnt/rhel79
mount rhel-server-7.9-x86_64-dvd.iso /mnt/rhel79

mkdir -p /var/www/html/rhel79/x86_64/base
cp -r /mnt/rhel79/* /var/www/html/rhel79/x86_64/base/
```

配置 Nginx 打开目录浏览：

```nginx file="/etc/nginx/sites-enabled/default"
server {
  listen 80 default_server;
  root /var/www/html;
  autoindex on;
}
```

重启服务：

```bash
systemctl restart nginx
```

浏览器访问：

```txt
http://10.0.0.1/rhel79/x86_64/base/
```

如果能看到目录列表，说明 HTTP 源可用。

## 准备 Kickstart 文件

Kickstart 文件可以从一次手工安装后的 `/root/anaconda-ks.cfg` 改出来。

```bash
mkdir -p /var/www/html/kickstart
cp rhel79.cfg /var/www/html/kickstart/
```

示例目录结构：

```txt
/var/www/html
├── kickstart
│   ├── rhel79.cfg
│   ├── rhel83.cfg
│   ├── v10sp1.cfg
│   └── v10sp2.cfg
├── rhel79
│   └── x86_64
│       └── base
├── rhel83
│   └── x86_64
│       └── base
└── v10sp2
    └── x86_64
        └── base
```

验证：

```bash
curl http://10.0.0.1/kickstart/rhel79.cfg
```

## 配置 DHCP

编辑 `/etc/dhcp/dhcpd.conf`：

```ini file="/etc/dhcp/dhcpd.conf"
default-lease-time 600;
max-lease-time 7200;
log-facility local7;

option arch code 93 = unsigned integer 16;

subnet 10.0.0.0 netmask 255.255.255.0 {
  range 10.0.0.10 10.0.0.50;
  next-server 10.0.0.1;

  class "pxeclients" {
    match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";

    if option arch = 00:07 or option arch = 00:09 {
      filename "efi/x86_64/BOOTX64.EFI";
    } else if option arch = 00:0b {
      filename "efi/aarch64/BOOTAA64.EFI";
    } else {
      filename "bios/x86_64/pxelinux.0";
    }
  }
}
```

这里的关键是：

- `next-server` 指向 TFTP 服务器。
- `filename` 根据客户端启动模式返回不同启动文件。
- Legacy BIOS 用 `pxelinux.0`。
- UEFI x86_64 用 `BOOTX64.EFI`。

重启 DHCP：

```bash
systemctl restart isc-dhcp-server
systemctl status isc-dhcp-server
```

如果要单独记录 DHCP 日志：

```ini file="/etc/rsyslog.d/dhcp-relay.conf"
local7.* -/var/log/dhcp-relay.log
```

## 配置 TFTP

编辑 `/etc/default/tftpd-hpa`：

```ini file="/etc/default/tftpd-hpa"
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure -vvv"
```

创建目录：

```bash
mkdir -p /srv/tftp/bios/x86_64
mkdir -p /srv/tftp/efi/x86_64
```

重启 TFTP：

```bash
systemctl restart tftpd-hpa
systemctl status tftpd-hpa
```

## 配置 Legacy BIOS 启动

准备 `pxelinux.0` 和 `vesamenu.c32`：

```bash
cp /var/www/html/rhel79/x86_64/base/Packages/syslinux-4.05-15.el7.x86_64.rpm .
rpm2cpio syslinux-4.05-15.el7.x86_64.rpm | cpio -dimv

cp usr/share/syslinux/pxelinux.0 /srv/tftp/bios/x86_64/
cp usr/share/syslinux/vesamenu.c32 /srv/tftp/bios/x86_64/
```

准备内核和 initrd：

```bash
mkdir -p /srv/tftp/bios/x86_64/images/rhel79
cp /var/www/html/rhel79/x86_64/base/images/pxeboot/vmlinuz /srv/tftp/bios/x86_64/images/rhel79/
cp /var/www/html/rhel79/x86_64/base/images/pxeboot/initrd.img /srv/tftp/bios/x86_64/images/rhel79/
```

创建启动菜单：

```bash
mkdir -p /srv/tftp/bios/x86_64/pxelinux.cfg
vim /srv/tftp/bios/x86_64/pxelinux.cfg/default
```

示例：

```txt file="/srv/tftp/bios/x86_64/pxelinux.cfg/default"
default vesamenu.c32
timeout 600

label local
  menu label Boot from ^local drive
  menu default
  localboot 0xffff

label rhel79
  menu label ^Install Red Hat Enterprise Linux 7.9
  kernel images/rhel79/vmlinuz
  append initrd=images/rhel79/initrd.img inst.ks=http://10.0.0.1/kickstart/rhel79.cfg quiet
```

## 配置 UEFI 启动

准备 UEFI 启动文件：

```bash
cp /var/www/html/rhel79/x86_64/base/EFI/BOOT/BOOTX64.EFI /srv/tftp/efi/x86_64/
cp /var/www/html/rhel79/x86_64/base/EFI/BOOT/grubx64.efi /srv/tftp/efi/x86_64/
chmod 644 /srv/tftp/efi/x86_64/BOOTX64.EFI
chmod 644 /srv/tftp/efi/x86_64/grubx64.efi
```

准备内核和 initrd：

```bash
mkdir -p /srv/tftp/efi/x86_64/images/rhel79
cp /var/www/html/rhel79/x86_64/base/images/pxeboot/vmlinuz /srv/tftp/efi/x86_64/images/rhel79/
cp /var/www/html/rhel79/x86_64/base/images/pxeboot/initrd.img /srv/tftp/efi/x86_64/images/rhel79/
```

创建 `grub.cfg`：

```txt file="/srv/tftp/efi/x86_64/grub.cfg"
set timeout=5
set default=0

menuentry 'Install Red Hat Enterprise Linux 7.9' {
  linuxefi efi/x86_64/images/rhel79/vmlinuz ip=dhcp inst.ks=http://10.0.0.1/kickstart/rhel79.cfg
  initrdefi efi/x86_64/images/rhel79/initrd.img
}

menuentry 'Install Kylin Linux Advanced Server V10 SP1' {
  linuxefi efi/x86_64/images/v10sp1/vmlinuz ip=dhcp inst.ks=http://10.0.0.1/kickstart/v10sp1.cfg
  initrdefi efi/x86_64/images/v10sp1/initrd.img
}
```

## 最终 TFTP 目录结构

```txt
/srv/tftp
├── bios
│   └── x86_64
│       ├── images
│       ├── pxelinux.0
│       ├── pxelinux.cfg
│       │   └── default
│       └── vesamenu.c32
└── efi
    └── x86_64
        ├── BOOTX64.EFI
        ├── grub.cfg
        ├── grubx64.efi
        └── images
```

## 日志和排错

常用日志：

```bash
tail -f /var/log/syslog | grep dhcp
tail -f /var/log/syslog | grep tftp
tail -f /var/log/nginx/access.log
```

排错顺序：

1. 客户端有没有拿到 DHCP 地址。
2. DHCP 返回的 `filename` 是否符合启动模式。
3. TFTP 文件路径是否存在，权限是否正确。
4. Nginx 是否能访问 ISO 源和 Kickstart。
5. Kickstart URL 是否写错。
6. BIOS/UEFI 模式是否和菜单文件匹配。

## 小结

PXE 的配置不难，但细节很多。建议每次只增加一个系统版本，先把一个 RHEL 或 Kylin 跑通，再复制目录和菜单项扩展。真正上线前，最好在测试网段验证，避免 DHCP 影响生产网络。
