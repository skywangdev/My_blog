---
title: "iperf3 网络测速使用笔记"
author: "Sky Wang"
pubDatetime: 2025-03-12T12:00:00+08:00
featured: false
draft: false
tags:
  - Linux
  - 网络
  - iperf3
description: "整理 iperf3 的安装、服务端和客户端测速命令、上传/下载/UDP 测试，以及如何阅读 SUM、receiver 和 Retr 结果。"
---

<figure>
  <img src="https://data.skywangdev.com/blog/S-10.jpeg" alt="iperf3 网络测速封面图" />
  <figcaption class="text-center">iperf3 适合测试两台主机之间的网络吞吐，不等同于公网测速网站。</figcaption>
</figure>

iperf3 用来测试不同主机之间的网络连接速度。它需要一端作为服务端，另一端作为客户端发起测试。相比网页测速，iperf3 更适合排查服务器之间、机房之间、内网链路之间的吞吐问题。

## Table of contents

## 安装 iperf3

CentOS/RHEL：

```bash
yum -y install iperf3
dnf install -y iperf3
```

Ubuntu/Debian：

```bash
apt-get update
apt-get -y install iperf3
```

Alpine：

```bash
apk add iperf3
```

Arch Linux：

```bash
pacman -S iperf3
```

Windows 版可以参考：

- [https://files.budman.pw/](https://files.budman.pw/)
- [iperf3-win-builds](https://github.com/ar51an/iperf3-win-builds)

## 启动服务端

在测速服务端执行：

```bash
iperf3 -s -p 5201
```

参数说明：

- `-s`：以服务端模式运行
- `-p`：指定端口，默认是 `5201`
- `-D`：后台运行

后台运行示例：

```bash
iperf3 -s -p 5201 -D
```

记得放行防火墙：

```bash
firewall-cmd --add-port=5201/tcp --permanent
firewall-cmd --reload
```

或者 Ubuntu：

```bash
ufw allow 5201/tcp
```

## 客户端测速

在客户端执行：

```bash
iperf3 -c 1.1.1.31 -p 5201 -t 30 -P 5 -R
```

参数说明：

- `-c`：客户端模式，并指定服务端地址
- `-p`：服务端端口
- `-t`：测试时长，单位秒
- `-P`：并发连接数
- `-R`：反向测试，服务端向客户端发送数据，常用来测下载
- `-u`：使用 UDP 测试
- `-i`：输出间隔，单位秒

上传测试：

```bash
iperf3 -c 1.1.1.31 -p 5201 -t 30 -P 5
```

下载测试：

```bash
iperf3 -c 1.1.1.31 -p 5201 -t 30 -P 5 -R
```

UDP 测试：

```bash
iperf3 -c 1.1.1.31 -p 5201 -u -b 100M -t 30
```

## 怎么看测试结果

旧笔记里有一段测试命令：

```bash
iperf3 -c 1.1.1.31 -p 5201 -i 1 -t 10 -P 5 -R
```

其中 `-P 5` 表示 5 个并发流。输出中每个 `[  5]`、`[  7]`、`[  9]` 是不同连接流，真正看整体带宽时，重点看 `[SUM]` 行。

示例：

```txt
[SUM]   0.00-10.00  sec   995 MBytes   835 Mbits/sec  4593 sender
[SUM]   0.00-10.00  sec   984 MBytes   826 Mbits/sec       receiver
```

阅读重点：

- `Transfer`：传输了多少数据。
- `Bitrate`：平均吞吐量。
- `Retr`：TCP 重传次数。
- `sender`：发送端统计。
- `receiver`：接收端统计。

一般更关注 `receiver`，因为它代表实际接收到了多少。

## 如何理解 Retr 重传

示例里有：

```txt
[SUM]   0.00-10.00  sec   995 MBytes   835 Mbits/sec  4593 sender
```

`Retr` 是 TCP 重传次数。重传越多，说明链路中可能存在丢包、拥塞或质量抖动。

这不一定代表链路完全不可用，但如果重传很多，即使带宽数字看起来还行，实际体验也可能不好，比如：

- SSH 卡顿
- 文件传输速度波动
- 视频流不稳定
- 应用连接延迟变高

## 并发数怎么选

单连接可能跑不满带宽，所以常用 `-P` 增加并发。

```bash
iperf3 -c 1.1.1.31 -t 30 -P 1
iperf3 -c 1.1.1.31 -t 30 -P 5
iperf3 -c 1.1.1.31 -t 30 -P 10
```

建议从 `1`、`5`、`10` 逐步测试。并发不是越高越好，太高可能测试的是系统调度和拥塞控制，而不是链路真实能力。

## 常用测试组合

测试上传：

```bash
iperf3 -c SERVER_IP -p 5201 -t 30 -P 5
```

测试下载：

```bash
iperf3 -c SERVER_IP -p 5201 -t 30 -P 5 -R
```

每秒输出一次：

```bash
iperf3 -c SERVER_IP -p 5201 -t 30 -P 5 -i 1
```

测试 UDP 100M：

```bash
iperf3 -c SERVER_IP -p 5201 -u -b 100M -t 30
```

输出 JSON，方便脚本分析：

```bash
iperf3 -c SERVER_IP -p 5201 -t 30 -P 5 -J
```

## 小结

iperf3 的核心不是“跑一个最大数字”，而是看链路是否稳定：

- 看 `[SUM]` 行，不要只看单个流。
- 看 `receiver`，不要只看 `sender`。
- 看 `Retr`，重传高说明链路质量可能有问题。
- 上传和下载都要测。
- TCP 和 UDP 测出来的问题维度不同。

如果测速结果明显低于预期，下一步可以继续查防火墙、MTU、路由路径、运营商限速、云服务器带宽上限和 CPU 性能瓶颈。
