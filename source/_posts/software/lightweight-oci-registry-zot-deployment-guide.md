---
title: 轻量级私有镜像仓库选型——Zot 部署与多工具兼容性实测
date: 2026-02-21 23:08:53
tags: [k3s, docker registry, docker compose, docker-compose, docker, podman, zot]
categories: [软件]
---

# 技术博客：轻量级私有镜像仓库选型——Zot 部署与多工具兼容性实测

### 1. 选型背景

在私有化环境构建容器镜像仓库时，传统的 Docker Registry (V2) 缺乏原生 UI，而 Harbor 架构过于臃肿。经过调研，**Zot** 表现出了极佳的工业属性：单二进制文件、资源占用极低、完全兼容 OCI 规范。

### 2. 环境说明

* **部署主机**：`192.168.0.113`
* **软件版本**：`zot-linux-amd64:v2.1.14`
* **客户端**：Docker (24.0.x), Podman (4.x)

### 3. 部署要点：权限与配置

Zot 的权限管理通过 `.htpasswd` 实现。实测发现，v2.x 版本对加密算法有特定要求，若使用默认的 MD5 格式，日志会抛出 `unsupported hash type` 警告。

#### 3.1 权限文件初始化

必须使用 **Bcrypt** 算法生成加密文件，以确保 v2.1.14 能够正确解析。

```bash
# 安装工具链
sudo apt-get install apache2-utils -y

# 生成 Bcrypt 加密文件 (-B 是核心参数)
# 账号：admin，密码：123456
htpasswd -bc -B .htpasswd admin 123456

```

#### 3.2 核心配置 (`config.json`)

需开启 UI 扩展并正确指向权限文件路径。

```json
{
  "distSpecVersion": "1.1.0",
  "storage": {
    "rootDirectory": "/var/lib/zot"
  },
  "http": {
    "address": "0.0.0.0",
    "port": "5000",
    "realm": "zot",
    "auth": {
      "htpasswd": { "path": "/etc/zot/htpasswd" }
    }
  },
  "extensions": {
    "ui": { "enable": true },
    "search": { "enable": true }
  }
}

```

#### 3.3 服务编排 (`docker-compose.yml`)

```yaml
services:
  zot:
    image: ghcr.io/project-zot/zot-linux-amd64:v2.1.14
    container_name: zot-registry
    ports:
      - "5000:5000"
    volumes:
      - ./data:/var/lib/zot
      - ./config.json:/etc/zot/config.json:ro
      - ./.htpasswd:/etc/zot/htpasswd:ro
    extra_hosts:
      - host.docker.internal:host-gateway
    restart: always

```

### 4. 进阶维护：多用户管理

针对多团队或国外客户场景，可通过以下方式维护用户清单：

```bash
# 添加新用户并强制 Bcrypt 加密 (注意：去掉 -c 以免覆盖已有用户)
htpasswd -b -B .htpasswd foreigner_user client_password

# 重启容器使配置生效
docker compose restart zot

```

### 5. 跨工具兼容性实测 (核心过程)

#### 5.1 Docker 链路验证

首先修改 `/etc/docker/daemon.json` 加入 `insecure-registries` 并重启 Docker 服务。

**执行登录与推送：**

```bash
➜  docker login 192.168.0.113:5000 -u admin -p 123456
Login Succeeded

# 标签与推送
➜  docker tag hello-world 192.168.0.113:5000/my-hello:v1
➜  docker push 192.168.0.113:5000/my-hello:v1

# 输出结果
The push refers to repository [192.168.0.113:5000/my-hello]
17eec7bbc9d7: Pushed 
v1: digest: sha256:2771e37a12... size: 1035

```

#### 5.2 Podman 链路验证

Podman 与 Docker 共享层存储逻辑，但需要显式跳过 TLS 校验（或修改 `registries.conf`）。

**执行拉取与重定向推送：**

```bash
# Podman 登录
➜  podman login 192.168.0.113:5000 -u admin -p 123456 --tls-verify=false
Login Succeeded!

# 实测 Podman 标签推送
➜  podman pull hello-world
➜  podman tag hello-world 192.168.0.113:5000/podman-hello:v1
➜  podman push 192.168.0.113:5000/podman-hello:v1 --tls-verify=false

# 关键输出
Getting image source signatures
Copying blob 17eec7bbc9d7 skipped: already exists  # 触发去重逻辑
Writing manifest to image destination

```

### 6. 避坑指南：日志诊断

调试过程中若返回 `401 Unauthorized`，应检查容器后端反馈：

```bash
docker logs -f zot-registry

{"time":"2026-02-21T14:52:57.50973439Z","level":"warn","message":"htpasswd entry has unsupported hash type","username":"admin","caller":"zotregistry.dev/zot/v2/pkg/api/htpasswd.go:119","func":"zotregistry.dev/zot/v2/pkg/api.(*HTPasswd).Authenticate","goroutine":75}
```

* **报错信息**：`"message":"htpasswd entry has unsupported hash type"`
* **根因**：使用了 MD5 或其他不支持的加密格式。
* **修复**：重做 `htpasswd -B` 流程。

### 总结

Zot 成功通过了 Docker 与 Podman 的交叉测试。其**存储去重 (Deduplication)** 表现优秀：在处理相同 Layer 的镜像时（如测试中的 `hello-world` 层），后端仅需存储一份物理文件。对于追求极致轻量、标准化的团队，Zot 是 Harbor 之外的最佳选择。
