---
title: Docker Compose 部署 Nginx 反向代理与 Cloudflare 建坑避坑指南
date: 2026-09-12 14:17:21
tags: [docker, docker compose, nginx, cloudflare]
categories: [软件]
---

# Docker Compose 部署 Nginx 反向代理与 Cloudflare 建坑避坑指南

在自建服务或部署 Docker 应用时，我们经常需要通过 Nginx 进行反向代理，将二级域名解析到具体的容器服务上。本文将以部署一个 **Docker 镜像代理服务** 为例，完整记录通过 **Docker Compose + Nginx** 实现域名解析、SSL 证书配置以及解决常见 **Cloudflare 301 重定向循环** 和 **404 响应** 的实战过程。

---

## 1. 架构与准备

* **域名**：`registry.example.com`（DNS 解析至服务器 IP，通过 Cloudflare 代理）
* **后端服务**：运行在宿主机 `18234` 端口的 Docker Registry 代理容器
* **Nginx**：通过 Docker / Docker Compose 部署，监听 80 和 443 端口

---

## 2. Nginx 配置文件

在宿主机的配置文件目录（例如 `./conf.d/registry.conf`）中创建对应的反向代理配置：

```nginx
# HTTP 80 端口：重定向至 HTTPS & ACME 证书校验
server {
    listen 80;
    server_name registry.example.com;

    # Certbot 自动续签路径
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # 强制跳转 HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS 443 端口：反向代理核心逻辑
server {
    listen 443 ssl;
    server_name registry.example.com;

    # SSL 证书配置
    ssl_certificate /etc/letsencrypt/live/registry.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/registry.example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        # 转发至宿主机具体端口
        proxy_pass http://host.docker.internal:18234;

        # 请求头透传
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}

```

---

## 3. Docker Compose 配置

创建 `docker-compose.yml` 文件：

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /var/www/certbot:/var/www/certbot
    # 关键配置：允许 Linux 容器内的 Nginx 通过 host.docker.internal 访问宿主机网络
    extra_hosts:
      - "host.docker.internal:host-gateway"

```

执行启动命令：

```bash
docker compose up -d

```

---

## 4. 踩坑与排查全记录

### 坑一：curl 报 `ERR_TOO_MANY_REDIRECTS` / 无限 301 循环

* **现象**：访问域名时，`curl -IL [https://registry.example.com](https://registry.example.com)` 提示重定向超过最大次数限制。
* **原因**：Cloudflare 的 **SSL/TLS 模式默认设为了 Flexible（灵活）**。
* 客户端使用 HTTPS 请求 Cloudflare。
* Cloudflare 强制使用 **HTTP (80端口)** 去连接源站 Nginx。
* Nginx 触发了 80 端口下的 `return 301 https://...`。
* Cloudflare 收到 301 后再次用 HTTP 请求源站，陷入无限死循环。


* **解决方案**：
* 登录 Cloudflare Dashboard -> **SSL/TLS** -> **Overview**。
* 将加密模式从 **Flexible** 改为 **Full** 或 **Full (strict)**。
* *注：改动为 Full 是 Cloudflare 的全局标准操作，不会损坏同域名下其他配置了 SSL 的服务。*



---

### 坑二：浏览器/Curl 访问根路径返回 404

* **现象**：解决 301 循环后，访问 `[https://registry.example.com](https://registry.example.com)` 直接返回 `HTTP 404`。
* **原因**：后端服务（如 Docker Registry 镜像代理）**仅实现了标准的 API 接口（如 `/v2/`），并没有配置根目录 `/` 的前端 Web 页面**。
* **验证方法**：
对后端专用的 API 接口发起请求测试：
```bash
curl -i https://registry.example.com/v2/

```


**响应结果**：
```http
HTTP/2 401 Unauthorized
www-authenticate: Bearer realm="/auth/token",service="registry.docker.io"
docker-distribution-api-version: registry/2.0

```


返回 `401 Unauthorized` 并带有 `docker-distribution-api-version: registry/2.0` 响应头，说明这完全符合 Docker Registry v2 的规范，反向代理已成功连通！

---

## 5. 服务验证与使用

配置完成后，可以直接使用该域名进行 Docker 镜像拉取：

```bash
# 直接拉取镜像
docker pull registry.example.com/library/alpine:latest

# 或配置为系统默认镜像加速器 (/etc/docker/daemon.json)
{
  "registry-mirrors": [
    "https://registry.example.com"
  ]
}

```

---

## 6. 总结

1. **Docker 容器反代宿主机端口**：Linux 环境下需要在 `docker-compose.yml` 中添加 `extra_hosts: ["host.docker.internal:host-gateway"]`。
2. **Cloudflare CDN + Nginx 重定向**：源站配置了 HTTP->HTTPS 强制跳转时，Cloudflare 的 SSL 模式必须设为 **Full** 或 **Full (strict)**。
3. **正确评估 404/401 状态码**：对于纯 API 服务（如 Docker Registry），根路径 404 或 `/v2/` 路径返回 401 均属符合规范的正常响应，无需过度怀疑 Nginx 转发失败。