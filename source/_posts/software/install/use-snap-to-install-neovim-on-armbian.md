---
title: 在armbian上使用snap安装neovim
date: 2025-02-14 21:40:00
tags: [snap, armbian, neovim]
categories: [软件]
---

# 概述

最近搞了一台armbian的机子，想在上面安装neovim，但是发现apt上的nvim版本太低了，只有0.4.0好像。经过一番的搜索，发现Snaps可以满足我的要求。

# Snaps介绍

Snaps是由Canonical提供的跨分发包管理系统的工具.Snaps 基本上是一个与其依赖项和库一起编译的应用程序——为应用程序运行提供了一个沙盒环境。它们安装起来更容易、更快捷，可以接收最新更新，并且不受操作系统和其他应用程序的限制。

# 安装Snaps

使用apt安装Snap；

```bash
sudo apt install snapd
```

# 安装neovim

使用snap安装neovim:

```bash
 sudo snap install --classic nvim
```

检查一下noevim的版本：

```bash
nvim --version
```

输出如下：

```bash
NVIM v0.10.4
Build type: RelWithDebInfo
LuaJIT 2.1.1713484068
Run "nvim -V1 -v" for more info
```

# 参考资料

1. [如何在各种 Linux 发行版中安装和使用 Snap](https://www.cnblogs.com/pipci/p/16109561.html)
