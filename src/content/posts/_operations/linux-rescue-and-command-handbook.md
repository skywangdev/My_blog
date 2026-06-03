---
title: "Linux 故障处理速查：重置 root 密码和常用命令"
author: "Sky Wang"
pubDatetime: 2020-08-03T12:00:00+08:00
featured: false
draft: false
tags:
  - Linux
  - CentOS
  - 运维
description: "记录 CentOS 7 重置 root 密码的方法，以及排查磁盘、内存、进程、网络和日志时常用的 Linux 命令。"
---


旧博客里有一篇 CentOS 7 重置 root 密码，还有一篇 Linux 命令大全。单独看一个太短，一个太散。这里合并成一篇更实用的速查笔记：前半部分处理“进不去系统”的急救场景，后半部分整理日常运维常用命令。

## Table of contents

## CentOS 7 重置 root 密码

这个方法适用于你能接触到服务器控制台、虚拟机控制台或云厂商 VNC 的情况。核心思路是进入引导编辑界面，临时以可写方式挂载根分区，然后重设 root 密码。

### 进入 GRUB 编辑模式

重启服务器，进入系统加载页面后，按键盘 `e` 进入编辑模式。

<figure>
  <img src="https://data.skywangdev.com/centos7-root-reset-1.png" alt="CentOS 7 GRUB 页面按 e 进入编辑模式" />
  <figcaption class="text-center">在 GRUB 启动项界面按 e。</figcaption>
</figure>

找到 `linux16` 开头的那一行。

<figure>
  <img src="https://data.skywangdev.com/centos7-root-reset-2.png" alt="CentOS 7 linux16 启动参数" />
  <figcaption class="text-center">找到 linux16 开头的启动参数。</figcaption>
</figure>

将末尾的 `ro` 改成 `rw`，并追加 `rd.break`。修改前后类似这样：

```txt
修改前：
linux16 /vmlinuz-3.10.0-1160.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet LANG=en_US.UTF-8

修改后：
linux16 /vmlinuz-3.10.0-1160.el7.x86_64 root=/dev/mapper/centos-root rw rd.break
```

然后按 `Ctrl + X` 启动。

<figure>
  <img src="https://data.skywangdev.com/centos7-root-reset-3.png" alt="CentOS 7 修改启动参数为 rw rd.break" />
  <figcaption class="text-center">修改为 rw rd.break 后启动。</figcaption>
</figure>

### 重置密码

进入单用户环境后，执行：

```bash
mount -o remount,rw /sysroot
chroot /sysroot
echo 'Aa@123456' | passwd root --stdin
touch /.autorelabel
exit
reboot
```

这里的 `touch /.autorelabel` 很重要。CentOS/RHEL 系统如果开启了 SELinux，改完密码后需要让系统在下次启动时重新标记上下文，否则可能出现登录异常。

<figure>
  <img src="https://data.skywangdev.com/centos7-root-reset-5.png" alt="CentOS 7 单用户模式重置 root 密码命令" />
  <figcaption class="text-center">执行重置密码命令后重启。</figcaption>
</figure>

### 注意事项

- 密码示例 `Aa@123456` 只是演示，线上环境不要使用弱密码。
- 如果是云服务器，优先使用云厂商提供的重置密码功能。
- 如果机器开启了磁盘加密或安全启动，操作路径可能不同。
- 重置完成后建议检查登录日志和用户权限，确认没有异常账号。

## 基础信息查看

<figure>
  <img src="https://data.skywangdev.com/blog/S-7.jpeg" alt="Linux 常用命令封面图" />
  <figcaption class="text-center">Linux 命令很多，日常最常用的是信息查看、文件处理、权限、磁盘和包管理。</figcaption>
</figure>

```bash
uname -m          # 显示处理器架构
uname -r          # 显示内核版本
arch              # 显示机器架构
date              # 显示系统时间
cat /proc/cpuinfo # 查看 CPU 信息
cat /proc/meminfo # 查看内存信息
cat /proc/version # 查看内核版本
cat /proc/net/dev # 查看网卡统计
lspci -tv         # 查看 PCI 设备
lsusb -tv         # 查看 USB 设备
dmidecode -q      # 查看硬件 DMI 信息
```

如果要看磁盘基础信息：

```bash
lsblk
blkid
fdisk -l
df -h
du -sh /var/log
```

## 关机和重启

```bash
shutdown -h now        # 立即关机
shutdown -r now        # 立即重启
shutdown -h 23:30      # 定时关机
shutdown -c            # 取消定时关机
reboot                 # 重启
logout                 # 注销当前 shell
```

线上服务器操作前建议先确认当前登录用户：

```bash
who
w
uptime
```

## 文件和目录

```bash
pwd                  # 显示当前路径
cd /home             # 进入目录
cd ..                # 返回上一级
cd -                 # 返回上次目录
ls -lah              # 查看文件详情
mkdir -p dir/a/b     # 创建多级目录
cp file1 file2       # 复制文件
cp -a dir1 dir2      # 复制目录并保留属性
mv old new           # 移动或重命名
rm -f file           # 删除文件
rm -rf dir           # 删除目录，谨慎使用
touch file           # 创建空文件或更新时间
file file            # 查看文件类型
```

`rm -rf` 最好不要手快。删除前可以先 `ls` 一下目标路径：

```bash
ls -lah /path/to/delete
```

## 文件搜索

```bash
find / -name file1
find /home/user1 -name "*.log"
find /usr/bin -type f -mtime -10
find / -name "*.rpm" -exec chmod 755 '{}' \;
locate "*.ps"
whereis halt
which halt
```

如果系统没有 `locate`，需要先安装并更新数据库：

```bash
updatedb
```

## 挂载和磁盘空间

```bash
mount /dev/sda1 /mnt/data
umount /mnt/data
mount -o loop file.iso /mnt/iso
mount -t vfat /dev/sdb1 /mnt/usb
df -h
du -sh *
du -sk * | sort -rn
```

设备繁忙时，不建议直接强制卸载。先查谁在占用：

```bash
lsof +f -- /mnt/data
fuser -vm /mnt/data
```

## 用户和用户组

```bash
groupadd group_name
groupdel group_name
useradd user1
userdel -r user1
passwd user1
usermod -aG wheel user1
chage -l user1
```

查看用户和组：

```bash
id user1
cat /etc/passwd
cat /etc/group
```

## 权限和所有者

```bash
ls -lh
chmod u+x file
chmod 755 script.sh
chmod 644 file.txt
chown user1 file1
chown -R user1:group1 directory1
chgrp group1 file1
```

常见权限可以简单记：

| 权限 | 含义 | 常见用途 |
| --- | --- | --- |
| `644` | 所有者可写，其他人只读 | 普通文件 |
| `755` | 所有者可写，所有人可执行 | 目录、脚本 |
| `600` | 只有所有者可读写 | 私钥、敏感配置 |

特殊属性：

```bash
chattr +i file1 # 文件不可删除、不可修改、不可重命名
chattr -i file1 # 取消不可变属性
lsattr file1    # 查看特殊属性
```

## 压缩和解压

```bash
tar -cvf archive.tar dir1
tar -xvf archive.tar
tar -czvf archive.tar.gz dir1
tar -xzvf archive.tar.gz
tar -cjvf archive.tar.bz2 dir1
tar -xjvf archive.tar.bz2
zip -r file.zip dir1
unzip file.zip
gzip file1
gunzip file1.gz
```

解压到指定目录：

```bash
tar -xvf archive.tar -C /tmp
```

## RPM 和 YUM/DNF

RPM 包操作：

```bash
rpm -ivh package.rpm
rpm -Uvh package.rpm
rpm -e package_name
rpm -qa | grep httpd
rpm -qi package_name
rpm -ql package_name
rpm -qf /etc/httpd/conf/httpd.conf
```

YUM/DNF 常用：

```bash
yum install package_name
yum remove package_name
yum update
yum search package_name
yum clean all

dnf install package_name
dnf remove package_name
dnf update
dnf search package_name
```

## 小结

这篇不是完整 Linux 教程，更像故障处理时的速查卡。真正运维时，我会优先记住这些类别：

- 先看系统状态：`uptime`、`df -h`、`free -h`、`top`
- 再看日志：`journalctl`、`tail -f`
- 再定位文件和权限：`find`、`ls -lah`、`chmod`、`chown`
- 最后处理服务和软件包：`systemctl`、`yum`、`dnf`、`apt`

命令不一定要全部背下来，但要知道遇到问题时该往哪个方向查。
