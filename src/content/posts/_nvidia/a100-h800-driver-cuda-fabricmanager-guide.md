---
title: "A100/H800 装机记录：NVIDIA 驱动、CUDA 和 Fabric Manager"
author: "Sky Wang"
pubDatetime: 2023-09-18T10:20:00+08:00
featured: true
draft: false
tags:
  - Linux
  - NVIDIA
  - CUDA
  - GPU
description: "记录 A100、H800 服务器安装 NVIDIA 驱动、CUDA 和 Fabric Manager 时的检查步骤和常见验证命令。"
---

这篇记录 GPU 服务器最基础的一步：把 A100、H800 节点装到 `nvidia-smi`、CUDA、Fabric Manager 都稳定可用。后面无论是跑训练、推理、容器集群，还是做压测，都要先把这一层打牢。

## Table of contents

## 一、安装前先确认硬件和系统

先确认系统能识别 GPU：

```bash
lspci | grep -i nvidia
```

常见输出里能看到 NVIDIA 设备，说明 PCIe 枚举至少是正常的。如果这里没有输出，先不要装驱动，应该先检查：

- GPU 是否正确插入。
- 服务器 BIOS 是否打开 Above 4G Decoding。
- PCIe 插槽和转接板是否正常。
- BMC/iDRAC/IPMI 里是否有硬件告警。
- 供电线和电源模块是否满足功耗需求。

再确认系统版本和内核：

```bash
cat /etc/os-release
uname -r
```

GPU 节点建议尽量统一系统版本、内核版本、驱动版本和 CUDA 版本。集群里不同节点混用版本，后面排查 NCCL、容器、训练任务会很麻烦。

## 二、安装基础依赖

RHEL、Rocky Linux、AlmaLinux 系列：

```bash
dnf install -y gcc make dkms pciutils kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```

Ubuntu 系列：

```bash
apt update
apt install -y build-essential dkms pciutils linux-headers-$(uname -r)
```

这里最重要的是 `kernel-devel` 或 `linux-headers` 必须和当前运行内核匹配。可以这样核对：

```bash
uname -r
ls /usr/src
```

如果刚升级过内核但没有重启，驱动安装经常失败。先重启到目标内核，再继续。

## 三、禁用 nouveau

安装 NVIDIA 官方驱动前，通常需要禁用 nouveau。

查看 nouveau 是否加载：

```bash
lsmod | grep nouveau
```

写入禁用配置：

```bash
cat > /etc/modprobe.d/blacklist-nouveau.conf << EOF
blacklist nouveau
options nouveau modeset=0
EOF
```

RHEL 系列更新 initramfs：

```bash
dracut --force
```

Ubuntu 更新 initramfs：

```bash
update-initramfs -u
```

重启：

```bash
reboot
```

重启后确认：

```bash
lsmod | grep nouveau
```

没有输出再安装官方驱动。

## 四、安装 NVIDIA 驱动

A100、H800 这类数据中心 GPU 建议选择数据中心驱动分支，并提前确认驱动和 CUDA 版本兼容。

注意，`nvidia-smi` 顶部显示的 `CUDA Version` 不是本机安装的 CUDA Toolkit 版本，而是当前驱动最高支持的 CUDA runtime 能力。Toolkit 版本看 `nvcc -V` 或包管理器，容器任务则看镜像里的 CUDA runtime。

如果使用 `.run` 包：

```bash
chmod +x NVIDIA-Linux-x86_64-*.run
./NVIDIA-Linux-x86_64-*.run --dkms
```

安装完成后验证：

```bash
nvidia-smi
```

重点看几项：

- GPU 型号是否正确。
- 每张卡是否都能显示。
- Driver Version 是否符合预期。
- 是否有 `ERR!`、`Unknown Error` 之类异常。

如果 `nvidia-smi` 报 `No devices were found`，但 `lspci` 能看到 GPU，通常是驱动模块加载、Secure Boot、内核头文件或 PCIe 资源分配问题。

## 五、安装 CUDA Toolkit

如果只是跑容器任务，宿主机不一定需要完整 CUDA Toolkit，驱动能正常工作即可。但如果要编译 NCCL、跑 CUDA samples 或做本机测试，建议安装 CUDA Toolkit。

示例：

```bash
sh cuda_12.1.1_530.30.02_linux.run
```

如果驱动已经单独安装过，CUDA 安装时不要重复安装驱动。

配置环境变量：

```bash
cat > /etc/profile.d/cuda.sh << EOF
export PATH=/usr/local/cuda/bin:\$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:\$LD_LIBRARY_PATH
EOF

source /etc/profile.d/cuda.sh
```

验证：

```bash
nvcc -V
```

## 六、安装 Fabric Manager

A100 SXM、H800 SXM 这类带 NVSwitch 的服务器需要 Fabric Manager。更准确地说，Fabric Manager 是给 NVSwitch/NVLink fabric 做初始化和管理的；普通 PCIe 卡形态机器通常不需要安装它。

Fabric Manager 版本要和驱动栈匹配，至少要在同一驱动分支内选择对应包。现场不要只看包名“差不多”，建议用驱动版本反查发行仓库里的 `nvidia-fabricmanager` 对应版本。

查看驱动版本：

```bash
nvidia-smi --query-gpu=driver_version --format=csv,noheader | head -n 1
```

安装对应版本的 Fabric Manager 后启动：

```bash
systemctl enable --now nvidia-fabricmanager
systemctl status nvidia-fabricmanager
```

如果 Fabric Manager 没启动，多 GPU 通信可能表现异常。尤其是 SXM 机器，不能只看 `nvidia-smi` 能显示卡就认为完成了。

## 七、检查 GPU 拓扑

查看 GPU 互联关系：

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

A100/H800 多卡服务器里，拓扑直接影响训练通信效率。后面做 NCCL 压测时，要结合拓扑一起看。

## 八、基础验证清单

建议每台 GPU 节点安装后都跑一遍：

```bash
nvidia-smi
nvidia-smi topo -m
systemctl status nvidia-fabricmanager
nvcc -V
dmesg | grep -i nvidia | tail -n 50
```

如果用于容器环境，再验证：

```bash
docker run --rm --gpus all nvidia/cuda:12.1.1-base-ubuntu22.04 nvidia-smi
```

## 小结

GPU 集群部署的第一层不是 Kubernetes，也不是训练框架，而是每台机器的驱动、CUDA、Fabric Manager 和拓扑状态。单机状态不稳定，后面所有集群问题都会被放大。

## 参考资料

- [NVIDIA Fabric Manager User Guide](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html)
- [CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
