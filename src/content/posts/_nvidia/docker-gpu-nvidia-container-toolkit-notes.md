---
title: "Docker GPU 环境记录：NVIDIA Container Toolkit 和容器验证"
author: "Sky Wang"
pubDatetime: 2024-03-09T16:00:00+08:00
featured: false
draft: false
tags:
  - Docker
  - NVIDIA
  - GPU
  - Linux
description: "记录 Linux 服务器上配置 Docker GPU 环境的步骤，包括 NVIDIA Container Toolkit 安装、Docker 运行时配置和容器内 nvidia-smi 验证。"
---

GPU 服务器如果要跑容器任务，宿主机只有 NVIDIA 驱动还不够。Docker 需要通过 NVIDIA Container Toolkit 把 GPU 暴露给容器。这里记录一套常见配置和验证方法。

## Table of contents

## 一、先确认宿主机驱动

先看宿主机：

```bash
nvidia-smi
```

如果宿主机 `nvidia-smi` 都不正常，先不要配置容器。容器 GPU 能力依赖宿主机驱动。

再看 Docker：

```bash
docker version
docker info
```

确认 Docker 本身能正常运行。

## 二、安装 NVIDIA Container Toolkit

安装方式以 NVIDIA 官方文档为准。Ubuntu、Debian、RHEL 系列的仓库配置略有差异，但核心包一般是：

```bash
nvidia-container-toolkit
```

Ubuntu/Debian 当前官方示例使用 `stable/deb` 仓库路径：

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
```

如果是生产集群，建议不要“今天装到什么版本就是什么版本”。先在一台测试节点验证，再固定 Toolkit 版本批量铺开。

安装后确认命令存在：

```bash
which nvidia-ctk
which nvidia-container-runtime
```

## 三、配置 Docker 运行时

NVIDIA 官方推荐用 `nvidia-ctk` 配置 Docker：

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

这个命令会改宿主机的 `/etc/docker/daemon.json`。如果机器上已有 registry mirror、日志驱动、默认 runtime 等配置，先备份再执行。

配置后可以看 Docker 信息：

```bash
docker info | grep -i runtime
```

如果项目要求把 NVIDIA runtime 设成默认值，要确认这样不会影响普通容器。一般情况下，直接使用 `--gpus all` 更清楚。

## 四、容器内验证

最直接的验证命令：

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

如果只想指定某张卡：

```bash
docker run --rm --gpus '"device=0"' nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

正常情况下，容器内会显示和宿主机一致的 GPU 信息。

## 五、常见问题

### 1. 容器里找不到 nvidia-smi

有些镜像本身没有 `nvidia-smi`，不是 GPU 没透传。建议先用 NVIDIA CUDA 官方镜像验证。

### 2. 报找不到 GPU

先确认：

```bash
nvidia-smi
docker info
which nvidia-container-runtime
```

再重新配置：

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 3. 驱动和镜像 CUDA 版本怎么看

容器里的 CUDA runtime 版本不能高于宿主机驱动能支持的范围。`nvidia-smi` 顶部显示的 `CUDA Version` 表示驱动最高支持的 CUDA runtime 能力，不代表宿主机装了同版本 CUDA Toolkit。实际项目里建议统一驱动版本、CUDA 镜像版本和应用要求。

### 4. 容器共享内存不够

跑训练或 NCCL Tests 时，Docker 默认共享内存可能不够。可以加：

```bash
docker run --rm --gpus all \
  --shm-size=1g \
  --ulimit memlock=-1 \
  nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

## 六、交付时记录什么

建议记录：

- 宿主机驱动版本。
- Docker 版本。
- NVIDIA Container Toolkit 版本。
- 测试镜像名称。
- `docker run --gpus all ... nvidia-smi` 输出。
- 是否设置默认 runtime。

这些信息能帮助后续判断问题是在宿主机、Docker、镜像还是应用。

## 七、小结

Docker GPU 环境的检查顺序很简单：宿主机 `nvidia-smi` 先正常，再装 NVIDIA Container Toolkit，配置 Docker 运行时，最后用 CUDA 镜像验证容器内 GPU。不要跳过宿主机检查，也不要只凭应用能不能启动来判断。

## 参考

- [NVIDIA Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- [CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [Docker GPU access](https://docs.docker.com/engine/containers/gpu/)
