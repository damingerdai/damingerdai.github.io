---
title: Podman的hello world入门
date: 2024-01-21 19:34:22
tags: [podman, 容器]
categories: [软件]
---

# 前言

[Podnam](https://podman.io/)是一个符合OCI，用于在 Linux® 系统上开发、提供了与 Docker 等类似的功能来管理容器。管理和运行容器开源工具。 Podman 最初由 Red Hat® 工程师与开源社区一起开发。Podman使用 libpod 库管理整个容器生态系统。

# 安装

如果你使用Macos, 可以使用homebrew安装：

```bash
brew install podman
```

安装之后就可以创建和启动Podnam虚拟机：

```bash
podman machine init
podman machine start
```

如果你使用Debian 或者 ubuntu， 可以使用apt-get命令安装：

```bash
sudo apt-get install runc -y
sudo apt-get -y install podman
```

 其他系统可能参考[podman安装页面](https://podman.io/docs/installation)

> 注意：对于 Windows 和 Mac，podman 需要一个虚拟机来部署容器。

# 配置

默认情况下，Podman 配置有两个容器注册表。

- [quay.io](https://quay.io/)
- [docker.io](https://hub.docker.com/)

我们可以在·/etc/containers/registries.conf·看到

但是为了方便我们构建基于Dockerfile的景象，我们需要在没有明确容器注册表的时候默认拉取docer.io的注册表，我们可以在·/etc/containers/registries.conf·或者·$HOME/.config/containers/registries.conf·设置unqualified-search-registries

```bash
unqualified-search-registries = ["docker.io"]
```

# 存储

每个系统用户都有自己的容器存储地址。这意味着，如果您尝试从不同的用户登录中提取镜像，它将从远程注册表而不是本地映像中提取镜像。

# 容器管理

拉取容器：

```bash
podman pull docker.io/nginx
```

运行容器：

```bash
podman  run --name docker-nginx -p 8080:80 docker.io/nginx
```

如果你绑定1024一下的端口，你需要使用root权限：

```bash
sudo podman run --name docker-nginx -p 80:80 docker.io/nginx
```

对于root用户，容器将会存储在·/var/lib/containers/storag·的文件夹中。
对于非root用户，容器将会存储在·$HOME/.local/share/containers/storage·的文件夹中。


# 参考资料

1. [What is Podman?](https://www.redhat.com/en/topics/containers/what-is-podman)
2. [Podman 初学者指南（上）](https://zhuanlan.zhihu.com/p/582580502)
3. [Podman "Error: no registries found in registries.conf, a registry must be provided" while logging/pulling from docker.io](https://github.com/containers/podman/issues/16096)
4. [What is Podman?](https://devopscube.com/podman-tutorial-beginners/)