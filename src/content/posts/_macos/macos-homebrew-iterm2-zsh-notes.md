---
title: "macOS 命令行环境记录：Homebrew、iTerm2 和 zsh"
author: "Sky Wang"
pubDatetime: 2024-05-18T10:30:00+08:00
featured: false
draft: false
tags:
  - macOS
  - Homebrew
  - iTerm2
  - zsh
description: "记录 macOS 上用 Homebrew 管理命令行工具，以及 iTerm2 和 zsh 的基础配置。"
---

这篇记录我在 macOS 上常用的命令行环境：Homebrew、iTerm2 和 zsh。它不追求花哨，重点是让 Mac 能舒服地做日常开发、远程登录服务器和处理一些运维类工作。

## Table of contents

## 一、为什么先装 Homebrew

macOS 自带不少命令，但很多版本偏旧，或者和 Linux 服务器上的习惯不太一样。Homebrew 更适合用来统一安装命令行工具，比如 `wget`、`tree`、`jq`、`htop`、`iperf3`、`tmux`。

安装命令以官网为准：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

安装后先检查：

```bash
brew --version
brew doctor
```

`brew doctor` 不一定每次都完全干净，但它能提示 PATH、权限、过期包等问题。刚装完 Homebrew 时建议看一眼。

## 二、常用 brew 命令

更新 Homebrew 自身索引：

```bash
brew update
```

安装软件：

```bash
brew install wget
brew install tree
brew install jq
brew install htop
```

查看已安装的包：

```bash
brew list
brew leaves
```

升级已安装的软件：

```bash
brew upgrade
```

清理旧版本缓存：

```bash
brew cleanup
```

查看某个包的信息：

```bash
brew info iperf3
```

如果不确定包名，可以先搜索：

```bash
brew search nmap
```

## 三、命令行工具清单

我常装的工具大致分几类。

基础查看：

```bash
brew install tree wget jq htop
```

网络排查：

```bash
brew install iperf3 nmap telnet mtr
```

文件和同步：

```bash
brew install rsync lftp
```

终端会话：

```bash
brew install tmux
```

这些工具不是每个人都需要一次装完。我的习惯是先装基础工具，后面遇到具体场景再补。

## 四、用 Brewfile 记录环境

如果经常换机器，或者想记录自己装过哪些工具，可以用 `brew bundle`。

导出当前安装列表：

```bash
brew bundle dump --file=~/Brewfile
```

新机器上按 Brewfile 安装：

```bash
brew bundle --file=~/Brewfile
```

Brewfile 适合做个人工具清单，不建议把它写得过满。真正长期不用的工具，留在清单里只会让新机器更乱。

## 五、iTerm2 基础配置

iTerm2 比 macOS 自带终端更适合长期使用。常用配置主要在 Profiles 里。

我一般会调整几项：

- 字体：选择自己看着舒服的等宽字体。
- 字号：不要太小，长时间看日志容易累。
- Colors：用浅色或深色都可以，重点是对比度足够。
- Working Directory：新窗口默认进入 Home 目录。
- Window：适当调大列数和行数。
- Keys：按自己的习惯设置分屏快捷键。

分屏是 iTerm2 很实用的地方。比如左边 SSH 到服务器，右边看本地命令或文档，排查问题时比来回切窗口舒服很多。

## 六、zsh 配置

macOS 默认 shell 已经是 zsh。先确认：

```bash
echo $SHELL
zsh --version
```

常用配置文件是：

```bash
~/.zshrc
```

Homebrew 装完后，要确认 PATH 里能找到 brew。Apple Silicon 机器通常是：

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Intel Mac 通常是：

```bash
eval "$(/usr/local/bin/brew shellenv)"
```

常用别名可以少量写一些：

```bash
alias ll='ls -lh'
alias la='ls -lha'
alias grep='grep --color=auto'
```

别名不要堆太多。尤其是经常登录 Linux 服务器的人，本地别名太多，到了服务器上反而容易不适应。

## 七、iTerm2 和 zsh 的 Shell Integration

iTerm2 有 Shell Integration，可以增强命令历史、目录跳转、远程文件下载等能力。新版 iTerm2 可以在 Profile 里开启自动加载，也可以按官方文档手动安装。

如果只是日常 SSH 和写命令，不装也可以。真正需要的时候再开，避免把 `.zshrc` 搞得太复杂。

## 八、维护习惯

Homebrew 不是装完就不用管。我的习惯是隔一段时间做一次：

```bash
brew update
brew upgrade
brew cleanup
brew doctor
```

如果某次升级后工具异常，先看：

```bash
brew info <package>
brew reinstall <package>
```

很多问题不是 macOS 本身坏了，而是 PATH、依赖版本或旧缓存没处理干净。

## 九、小结

macOS 命令行环境不用做得太复杂。Homebrew 负责装工具，iTerm2 负责好用的终端窗口，zsh 负责基本 shell 配置。把这三件事处理好，Mac 就可以很顺手地连接 Linux 服务器、查日志、跑网络测试和处理日常命令行工作。

## 参考

- [Homebrew Documentation](https://docs.brew.sh/)
- [Homebrew Installation](https://docs.brew.sh/Installation.html)
- [Homebrew Brewfile](https://docs.brew.sh/Brew-Bundle-and-Brewfile)
- [iTerm2 Shell Integration](https://iterm2.com/documentation-shell-integration.html)
