---
title: "Linux 安装 NVIDIA 驱动、CUDA 与 Fabric Manager 笔记"
author: "Sky Wang"
pubDatetime: 2021-04-13T12:00:00+08:00
featured: false
draft: false
tags:
  - Linux
  - NVIDIA
  - CUDA
description: "整理 Linux 服务器安装 NVIDIA 驱动、CUDA 工具包和 nvidia-fabricmanager 的步骤，以及常见验证命令。"
---

<figure>
  <img src="https://data.skywangdev.com/blog/S-3.jpeg" alt="Linux 安装 NVIDIA 驱动封面图" />
  <figcaption class="text-center">GPU 服务器装驱动前，最重要的是确认内核、驱动版本和 CUDA 版本匹配。</figcaption>
</figure>

这篇笔记整理 Linux 服务器上安装 NVIDIA 驱动、CUDA 工具包和 Fabric Manager 的过程。旧文标题写的是 Ubuntu，但内容里也包含 AlmaLinux/RockyLinux 的命令，所以这里改成更通用的 Linux GPU 服务器笔记。

## Table of contents

## 安装前先确认 GPU

先确认系统能看到 NVIDIA 设备：

```bash
lspci | grep -i nvidia
```

如果没有任何输出，需要先检查：

- GPU 是否正确安装。
- PCIe 槽位是否识别。
- BIOS 里相关选项是否启用。
- 虚拟化环境是否已做 GPU 直通。

## 安装编译环境和内核头文件

驱动安装经常需要编译内核模块，所以要安装编译工具和当前内核对应的头文件。

AlmaLinux/RockyLinux：

```bash
dnf install -y gcc dkms kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```

Ubuntu：

```bash
apt update
apt install -y build-essential dkms linux-headers-$(uname -r)
```

这里最容易踩坑的是内核版本不匹配。安装前确认：

```bash
uname -r
ls /usr/src
```

## 禁用 nouveau

Nouveau 是开源 NVIDIA 驱动，安装官方驱动前通常需要禁用。

查看是否加载：

```bash
lsmod | grep nouveau
```

创建禁用配置：

```bash
cat > /etc/modprobe.d/blacklist-nouveau.conf << EOF
blacklist nouveau
options nouveau modeset=0
EOF
```

更新 initramfs。

AlmaLinux/RockyLinux：

```bash
mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak
dracut /boot/initramfs-$(uname -r).img $(uname -r)
```

Ubuntu：

```bash
update-initramfs -u
```

完成后重启：

```bash
reboot
```

重启后再次确认：

```bash
lsmod | grep nouveau
```

没有输出再继续安装。

## 安装 NVIDIA 驱动

从 NVIDIA 驱动下载页面选择对应显卡和系统版本：

[NVIDIA 驱动程序下载](https://www.nvidia.cn/Download/index.aspx?lang=cn)

如果下载的是 `.run` 文件，执行：

```bash
bash NVIDIA-Linux-x86_64-470.256.02.run
```

如果需要指定内核源码路径：

```bash
bash NVIDIA-Linux-x86_64-470.256.02.run \
  --kernel-source-path=/usr/src/kernels/$(uname -r) \
  -k $(uname -r)
```

验证：

```bash
nvidia-smi
```

能看到 GPU、驱动版本、显存信息，就说明驱动基本可用。

## 安装 CUDA 工具包

CUDA 下载地址：

[CUDA Toolkit Archive](https://developer.nvidia.com/cuda-toolkit-archive)

安装前要确认 CUDA 和驱动版本兼容。官方说明：

[CUDA Toolkit Release Notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)

示例安装：

```bash
bash cuda_11.4.0_470.256.02_linux.run
```

如果已经单独安装过驱动，安装 CUDA 时建议取消驱动安装选项，否则可能覆盖已有驱动或安装失败。

验证：

```bash
/usr/local/cuda/bin/nvcc -V
```

必要时配置环境变量：

```bash
cat > /etc/profile.d/cuda.sh << EOF
export PATH=/usr/local/cuda/bin:\$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:\$LD_LIBRARY_PATH
EOF

source /etc/profile.d/cuda.sh
```

## 安装 nvidia-fabricmanager

Fabric Manager 常见于带 NVLink/NVSwitch 的多 GPU 服务器。它的版本要和驱动版本匹配。

### 添加软件源

AlmaLinux/RockyLinux 示例：

```bash
dnf config-manager --add-repo \
  http://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
```

Ubuntu 20.04 示例：

```bash
wget https://developer.download.nvidia.cn/compute/cuda/repos/ubuntu2004/x86_64/cuda-ubuntu2004.pin
mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600

wget https://developer.download.nvidia.cn/compute/cuda/repos/ubuntu2004/x86_64/7fa2af80.pub
apt-key add 7fa2af80.pub
rm 7fa2af80.pub

echo "deb http://developer.download.nvidia.cn/compute/cuda/repos/ubuntu2004/x86_64 /" \
  | tee /etc/apt/sources.list.d/cuda.list
```

### 安装服务

AlmaLinux/RockyLinux：

```bash
dnf module enable -y nvidia-driver:470
dnf install -y nvidia-fabric-manager:470.256.02 \
  nvidia-fabric-manager-devel-0:470.256.02
```

Ubuntu：

```bash
apt update
apt install -y nvidia-fabricmanager-470=470.256.02-1
```

启动并设置开机自启：

```bash
systemctl start nvidia-fabricmanager
systemctl status nvidia-fabricmanager
systemctl enable nvidia-fabricmanager
```

验证 NVLink 拓扑：

```bash
nvidia-smi topo -m
```

如果输出里有 `NV*` 字样，说明 GPU 之间存在 NVLink 连接。还需要结合服务器硬件拓扑确认是否符合预期。

## 常见问题

### nvidia-smi 无输出或报错

可以依次检查：

```bash
lsmod | grep nvidia
dmesg | grep -i nvidia
journalctl -xe | grep -i nvidia
```

常见原因：

- nouveau 没有禁用干净。
- 内核升级后驱动模块没有重新编译。
- 驱动版本不支持当前 GPU。
- Secure Boot 阻止内核模块加载。

### CUDA 可用但 nvcc 找不到

大概率是环境变量没配：

```bash
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
```

### Fabric Manager 启动失败

重点检查版本是否匹配：

```bash
nvidia-smi
dpkg -l | grep fabric
rpm -qa | grep fabric
```

## 小结

GPU 服务器装驱动时，不建议一上来就复制命令猛装。更稳的顺序是：

1. 确认 GPU 硬件可见。
2. 安装当前内核匹配的 headers/devel。
3. 禁用 nouveau 并重启。
4. 安装驱动并用 `nvidia-smi` 验证。
5. 再装 CUDA。
6. 多 GPU/NVLink 场景再处理 Fabric Manager。

驱动、CUDA、Fabric Manager 三者版本匹配，是这类问题的核心。
