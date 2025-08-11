---
title: podman登录非https的私有仓库
date: 2025-08-11 22:33:06
tags: [podman]
categories: [后端]
---

# 前言

本文的目的是为了解决podman登陆非https的私有仓库。
这里使用192.168.31.220:5000威力

# 方案1：禁用 TLS 验证（临时解决方案）

对于测试环境或内部信任的网络，可以临时禁用 TLS 验证：

```bash
podman login --tls-verify=false 192.168.31.220:5000
```

> 注意：生产环境不建议禁用 TLS 验证，这会降低安全性。

# 方案2: 配置仓库为不安全仓库

编辑 /etc/containers/registries.conf文件，添加以下内容：

```bash
[[registry]]
location = "192.168.31.220:5000"
insecure = true
```

# 方案3：正确配置 HTTPS（生产环境推荐）

如果这是生产环境，建议为您的私有仓库配置正确的 HTTPS。