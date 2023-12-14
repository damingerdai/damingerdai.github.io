---
title: 学习multipass笔记
date: 2023-12-14 20:35:27
tags: [multipass]
categories: [软件]
---


# 前言

Multipass 是一个轻量虚拟机管理器，是由 Ubuntu 运营公司 Canonical 所推出的开源项目。运行环境支持 Linux、Windows、macOS。在不同的操作系统上，使用的是不同的虚拟化技术。在 Linux 上使用的是 KVM、Window 上使用 Hyper-V、macOS 中使用 HyperKit 以最小开销运行 VM，支持在笔记本模拟小型云。

同时，Multipass 提供了一个命令行界面来启动和管理 Linux 实例。下载一个全新的镜像需要几秒钟的时间，并且在几分钟内就可以启动并运行 VM。

# 安装

在window环境下进行部署，下载最新安装包：[https://github.com/canonical/multipass/releases/](https://github.com/canonical/multipass/releases/)

# 创建vm

创建实例

```
multipass launch -n k3s-master -c 2 -m 4G -d 10G
```

查看实例

```
multipass list

Name                    State             IPv4             Image
k3s-master              Running           172.19.151.166   Ubuntu 22.04 LTS
```

查看实例状态

```
Name:           k3s-master
State:          Running
IPv4:           172.19.151.166
Release:        Ubuntu 22.04.3 LTS
Image hash:     6d6af17f28c8 (Ubuntu 22.04 LTS)
CPU(s):         2
Load:           0.00 0.00 0.00
Disk usage:     1.6GiB out of 9.6GiB
Memory usage:   213.8MiB out of 3.8GiB
Mounts:         --
```

进入shell环境

```bash
multipass shell k3s-maste

Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Dec 14 20:45:28 CST 2023

  System load:  0.0               Processes:             100
  Usage of /:   17.2% of 9.51GB   Users logged in:       1
  Memory usage: 6%                IPv4 address for eth0: 172.19.151.166
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Thu Dec 14 20:26:30 2023 from 172.19.144.1
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
```


# 引用

1. [【工具系列】轻量级虚拟机 Multipass 使用教程](https://www.mobaijun.com/posts/3701652676.html)
2. [轻量虚拟机 Multipass 的部署和使用](https://www.cnblogs.com/hewei-blogs/articles/17569105.html)
3. [Multipass - 如 Docker 般的虛擬機](https://jackkuo.org/post/multipass_tutorial/)