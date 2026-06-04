---
title: "A100 节点标准化记录：系统、驱动、容器运行时和巡检"
author: "Sky Wang"
pubDatetime: 2023-12-07T14:30:00+08:00
featured: true
draft: false
tags:
  - Linux
  - NVIDIA
  - Docker
  - GPU
description: "记录 A100 节点标准化时常做的系统基线、驱动确认、NVIDIA Container Toolkit、Docker 验证和巡检脚本。"
---

这篇记录 A100 GPU 节点的标准化部署。目标不是把某一台机器装通，而是让一批节点尽量保持一致：系统一致、驱动一致、容器运行时一致、巡检口径一致。

## Table of contents

## 一、为什么要做标准化

GPU 集群里最难排查的问题，很多不是单点故障，而是节点之间有细微差异。

常见差异包括：

- 内核版本不同。
- 驱动版本不同。
- CUDA 镜像版本不同。
- Docker 或 containerd 配置不同。
- 某些节点 Fabric Manager 没启动。
- 某些节点 BIOS 参数没统一。

这些差异在单机上可能不明显，但到多机训练、NCCL 通信、调度系统里就会变成随机问题。

## 二、系统基线

建议先记录每台节点的基础信息：

```bash
hostname
cat /etc/os-release
uname -r
lscpu | head
free -h
lsblk
```

GPU 信息：

```bash
lspci | grep -i nvidia
nvidia-smi
nvidia-smi topo -m
```

如果是 8 卡 A100 节点，最基本要求是每台机器都能稳定看到 8 张卡，并且拓扑符合机器型号。

## 三、驱动和 Fabric Manager

检查驱动：

```bash
nvidia-smi --query-gpu=index,name,driver_version,memory.total --format=csv
```

`nvidia-smi` 顶部的 `CUDA Version` 不等于宿主机安装的 CUDA Toolkit 版本。它表示当前驱动最高支持到哪个 CUDA runtime 能力；真正的 Toolkit 版本要看 `nvcc -V`、包管理器记录，或者容器镜像 tag。

检查 Fabric Manager：

```bash
systemctl is-active nvidia-fabricmanager
systemctl status nvidia-fabricmanager --no-pager
```

Fabric Manager 主要用于带 NVSwitch 的平台，比如 HGX A100、HGX H100/H800 这类 SXM 机器。PCIe 卡形态的服务器通常不需要它，验收时不要机械套用。需要 Fabric Manager 的机器，服务状态不正常就先不要加入训练池，多卡通信问题很容易在这里埋雷。

## 四、安装 Docker

以 Ubuntu 为例：

```bash
apt update
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

启动并设置开机自启：

```bash
systemctl enable --now docker
docker version
```

## 五、安装 NVIDIA Container Toolkit

安装 NVIDIA Container Toolkit 后，容器才能通过 `--gpus all` 使用 GPU。

Ubuntu 示例：

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  > /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt update
apt install -y nvidia-container-toolkit
```

这里用的是 NVIDIA 官方文档当前推荐的 `stable/deb` 仓库路径。老文章里常见的 `$distribution/libnvidia-container.list` 写法在一些新系统上容易失效，批量装机时建议固定一套经过验证的 Toolkit 版本。

配置 Docker 运行时：

```bash
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
```

`nvidia-ctk` 会修改宿主机的 `/etc/docker/daemon.json`。如果这台机器已有业务自定义 Docker 配置，先备份再改，避免把 registry mirror、日志驱动之类配置覆盖掉。

验证：

```bash
docker run --rm --gpus all nvidia/cuda:12.1.1-base-ubuntu22.04 nvidia-smi
```

## 六、容器镜像版本要统一

训练任务常见镜像变量：

- CUDA 版本。
- cuDNN 版本。
- PyTorch 或 TensorFlow 版本。
- NCCL 版本。
- Python 版本。

建议在集群里固定镜像 tag，不要用 `latest`：

```bash
docker pull nvidia/cuda:12.1.1-cudnn8-devel-ubuntu22.04
```

记录镜像摘要：

```bash
docker image inspect nvidia/cuda:12.1.1-cudnn8-devel-ubuntu22.04 \
  --format '{{index .RepoDigests 0}}'
```

## 七、基础巡检脚本

可以准备一个简单巡检脚本：

```bash file="gpu-node-check.sh"
#!/usr/bin/env bash
set -euo pipefail

echo "== Host =="
hostname
cat /etc/os-release | grep PRETTY_NAME
uname -r

echo "== GPU =="
nvidia-smi --query-gpu=index,name,driver_version,memory.total,temperature.gpu,power.draw --format=csv

echo "== Fabric Manager =="
systemctl is-active nvidia-fabricmanager || true

echo "== Topology =="
nvidia-smi topo -m

echo "== Docker GPU =="
docker run --rm --gpus all nvidia/cuda:12.1.1-base-ubuntu22.04 nvidia-smi
```

执行：

```bash
bash gpu-node-check.sh
```

## 八、加入集群前的判断

一台节点加入集群前，至少要满足：

- `nvidia-smi` 正常。
- GPU 数量正确。
- Fabric Manager 正常。
- Docker GPU 容器能看到全部 GPU。
- 拓扑符合预期。
- 巡检输出没有明显异常。

## 小结

A100 节点标准化的价值在于减少变量。驱动和容器运行时不是一次性安装动作，而是后续集群稳定性的基础。越早把节点状态统一，后面排查训练任务越轻松。

## 参考资料

- [NVIDIA Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- [NVIDIA Fabric Manager User Guide](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/index.html)
- [CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
