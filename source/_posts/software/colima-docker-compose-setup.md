---
title: 无 Docker Desktop + Colima + Docker Compose v2 标准化方案
date: 2026-04-27 14:16:01
tags: [colima, docker, docker compose]
categories: [软件]
---

# 无 Docker Desktop + 纯 CLI + Colima + Docker Compose v2 标准化方案

## 一、背景

在 macOS 环境中，Docker Desktop 虽然开箱即用，但存在以下问题：

- 资源占用高（CPU / 内存）
- 启动慢
- 商业使用存在许可限制
- 与开发环境（如 k8s / 自定义 runtime）耦合较重

因此，采用以下组合替代：

> **Colima + Docker CLI + Compose v2 Plugin**

目标是实现：

- 轻量级 Docker 运行环境
- 完整 CLI 操作体验
- 原生支持 `docker compose`（v2）

---

## 二、整体架构

```bash
docker CLI
   ↓
docker context (colima)
   ↓
unix socket (~/.colima/default/docker.sock)
   ↓
Colima VM (container runtime)
````

---

## 三、环境安装

### 1. 安装基础工具

```bash
brew install docker colima docker-compose
```

说明：

* `docker`：CLI 工具
* `colima`：本地 container runtime（基于 Lima + containerd）
* `docker-compose`：Compose v2 实现（后续转换为 plugin）

---

### 2. 启动 Colima

```bash
colima start
```

可选配置：

```bash
colima start --cpu 4 --memory 8
```

---

### 3. 设置 Docker Context

```bash
docker context use colima
```

验证：

```bash
docker context ls
```

---

## 四、启用 Docker Compose v2（核心步骤）

Homebrew 不直接提供 `docker-compose-plugin`，需要手动转换：

### 1. 创建 plugin 目录

```bash
mkdir -p ~/.docker/cli-plugins
```

---

### 2. 软链接 compose

```bash
ln -s $(which docker-compose) ~/.docker/cli-plugins/docker-compose
```

---

### 3. 验证

```bash
docker compose version
```

期望输出：

```bash
Docker Compose version 5.x.x
```

### 4. 补充

Colima 默认只安装了底层的 Docker 守护进程，但在你的 Mac 宿主机上，缺少了 Docker 的 buildx 插件（也就是用来控制 BuildKit 的客户端组件）。

```bash
# 1. 安装 buildx 插件
brew install docker-buildx

# 2. 创建 Docker 插件目录（如果不存在的话）
mkdir -p ~/.docker/cli-plugins

# 3. 将刚刚安装的 buildx 软链接到 Docker 插件目录下
ln -sfn $(brew --prefix)/opt/docker-buildx/bin/docker-buildx ~/.docker/cli-plugins/docker-buildx
```

---

## 五、最终使用方式

统一使用：

```bash
docker compose up
docker compose down
docker compose ps
```

不再使用：

```bash
docker-compose  # deprecated
```

---

## 六、环境验证

### 1. Docker

```bash
docker ps
```

---

### 2. Compose

```bash
docker compose version
```

---

### 3. Socket

```bash
echo $DOCKER_HOST
```

应为空或由 context 管理（推荐）

---

## 七、常见问题

### 1. `docker compose` 不可用

原因：

* plugin 未配置

解决：

```bash
ls ~/.docker/cli-plugins
```

确认存在：

```bash
docker-compose
```

---

### 2. 连接错误

```bash
Cannot connect to the Docker daemon
```

排查：

```bash
docker context ls
```

---

### 3. 权限或路径问题

```bash
which docker
which docker-compose
```

确保均来自 Homebrew：

```bash
/opt/homebrew/bin/
```

---

## 八、最佳实践

### 1. 固定 context

在 shell 配置中加入：

```bash
export DOCKER_CONTEXT=colima
```

---

### 2. 避免 DOCKER_HOST 干扰

不要手动设置：

```bash
export DOCKER_HOST=...
```

交由 context 管理

---

### 3. 统一团队规范

建议团队统一：

* 使用 Colima
* 使用 `docker compose`
* 禁止 `docker-compose`

---

## 九、优缺点分析

### 优点

* 轻量（相比 Docker Desktop）
* 无商业授权限制
* 启动速度快
* 与 k8s / containerd 更贴近

---

### 缺点

* 需要手动配置 Compose plugin
* 初次上手成本略高
* 部分 GUI 工具不可用

---

## 十、总结

该方案的核心是：

> **用 Colima 替代 Docker Desktop，用 CLI + plugin 还原官方 Docker 体验**

最终效果：

```bash
docker compose up
```

与 Docker Desktop 环境保持一致，但更加轻量、可控。

---

## 十一、参考命令速查

```bash
# 启动环境
colima start

# 查看 context
docker context ls

# 切换 context
docker context use colima

# Compose
docker compose up
docker compose down

# 查看 plugin
ls ~/.docker/cli-plugins
```

---

## 十二、适用人群

* 后端工程师（Node.js / Go / Java）
* 前端工程化开发（需要容器化）
* 使用 k8s / 本地集群（k3s / kind）
* 追求轻量开发环境的工程师`
