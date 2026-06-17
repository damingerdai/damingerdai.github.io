---
title: 使用K3s 从零部署 CloudNativePG：打造 Homelab 统一 PostgreSQL 平台
date: 2026-06-12 14:08:53
tags: [kubernetes, k3s, CloudnativePg, HomeLab]
categories: [软件]
---

# K3s 从零部署 CloudNativePG：打造 Homelab 统一 PostgreSQL 平台

## 前言

在很多 CloudNativePG 教程中，通常只会介绍如何安装 Operator 并创建一个 PostgreSQL 集群。

但在实际 Homelab 场景中，我们更希望：

* 整个 K3s 集群共用一套 PostgreSQL 服务
* 支持 Spring Boot 项目
* 支持 Gitea
* 支持 Immich
* 支持未来新增应用
* 能够通过固定 IP 从局域网访问数据库
* 保留未来扩容多节点 K3s 的能力

因此本文不只是介绍 CloudNativePG 的安装，而是介绍如何将其建设为 Homelab 的统一 PostgreSQL 平台。

---

# 环境说明

## K3s 节点

```text
192.168.31.215
```

## 路由器

小米路由器

DHCP 配置：

```text
192.168.31.5 - 192.168.31.229
```

## MetalLB 地址规划

```text
192.168.31.230 postgres
192.168.31.231 ingress
192.168.31.232 gitea
192.168.31.233 argocd
192.168.31.234 grafana
```

本文中 PostgreSQL 固定使用：

```text
192.168.31.230
```

---

# 最终架构

```text
                    家庭局域网

 ┌──────────────┐
 │ MacBook      │
 ├──────────────┤
 │ ThinkPad     │
 ├──────────────┤
 │ DataGrip     │
 ├──────────────┤
 │ Spring Boot  │
 └───────┬──────┘
         │
         ▼
  postgres.home.lab
         │
         ▼
   192.168.31.230
         │
      MetalLB
         │
         ▼
   postgres-lb
         │
         ▼
   postgres-rw
         │
         ▼
 CloudNativePG
```

集群外通过 LoadBalancer 访问。

集群内通过 Service DNS 访问。

---

# 安装 MetalLB

添加 Helm Repo：

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
```

安装：

```bash
helm install metallb metallb/metallb \
  -n metallb-system \
  --create-namespace
```

确认：

```bash
kubectl get pods -n metallb-system
```

```text
NAME                                            READY   STATUS    RESTARTS   AGE
metallb-controller-55846b4849-t2dr7             1/1     Running   0          18h
metallb-frr-k8s-statuscleaner-8bf664555-52sgc   1/1     Running   0          18h
metallb-frr-k8s-w2n6j                           5/5     Running   0          18h
metallb-speaker-bd4fd                           1/1     Running   0          18h
```

安装完成后创建地址池：

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.31.230-192.168.31.239
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab
  namespace: metallb-system
```

---

# 安装 CloudNativePG

CloudNativePG 提供了一个非常强大的 kubectl 插件，可以帮助你轻松查看数据库状态、进行主备切换、查看日志等。

在你的控制端（或 K3s 节点上）运行以下命令安装：

```bash
curl -sSfL https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-cnpg-plugin.sh | sudo sh -s -- -b /usr/local/bin
```
如果你是中国用户，你可以使用gh-proxy来加速

```bash
curl -sSfL https://gh-proxy.com/https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-cnpg-plugin.sh | sudo sh -s -- -b /usr/local/bin
```

验证：

```bash
kubectl cnpg version
Build: {Version:1.29.1 Commit:a4060c152 Date:2026-05-08}
```

使用helm来部署CNPG Operator：

```bash
# 添加并更新仓库
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

# 安装 1.29 对应的 Operator
helm install cnpg-operator cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace
```

---

# 创建 PostgreSQL 集群

创建超级管理员密码：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-superuser
type: kubernetes.io/basic-auth
stringData:
  username: postgres
  password: StrongPassword123
```

创建 Cluster：

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres
spec:
  instances: 1

  storage:
    size: 20Gi

  superuserSecret:
    name: postgres-superuser

  bootstrap:
    initdb:
      database: postgres

  managed:
    services:
      additional:
        - selectorType: rw
          serviceTemplate:
            metadata:
              name: postgres-lb
            spec:
              type: LoadBalancer
              loadBalancerIP: 192.168.31.230
```

这里使用 CloudNativePG 官方推荐的 managed.services，而不是额外创建 Service。

这样未来主从切换时，LoadBalancer 会自动跟随新的 Primary。

查看超级用户密码：

```bash
kubectl get secret postgres-superuser \
-o jsonpath='{.data.password}' | base64 -d
```

---

# 验证 LoadBalancer

```bash
kubectl get svc
```

输出：

```text
postgres-rw
postgres-ro
postgres-r
postgres-lb
```

查看：

```bash
kubectl get svc postgres-lb
```

应看到：

```text
EXTERNAL-IP
192.168.31.230
```

本机测试：

```bash
psql \
-h 192.168.31.200 \
-U postgres
```

成功进入即可。

---

# 创建业务数据库

```sql
CREATE DATABASE hoteler;
CREATE DATABASE gitea;
CREATE DATABASE immich;
```

创建业务用户：

```sql
CREATE USER hoteler WITH PASSWORD 'password';

GRANT ALL PRIVILEGES
ON DATABASE hoteler
TO hoteler;
```

推荐一个应用一个数据库，一个用户。

---

# Spring Boot 访问 PostgreSQL

## 集群外访问

例如本地开发：

```yaml
spring:
  datasource:
    url: jdbc:postgresql://192.168.31.230:5432/hoteler
    username: hoteler
    password: password
```

推荐进一步配置 DNS：

```text
postgres.home.lab
    ↓
192.168.31.230
```

然后：

```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres.home.lab:5432/hoteler
```

---

## 集群内访问

如果 Spring Boot 本身运行在 K3s：

```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres-rw:5432/hoteler
```

不要通过 LoadBalancer 回流。

直接使用 Kubernetes Service。

---

# 为什么要同时保留 Service 和 LoadBalancer

很多教程会直接让所有应用连接 LoadBalancer。

实际上：

集群内部：

```text
Spring Boot
↓
postgres-rw
```

集群外部：

```text
DataGrip
↓
postgres.home.lab
↓
192.168.31.230
```

这样网络路径最短，也符合 Kubernetes 最佳实践。

---

# 后续规划

未来可以继续接入：

* Gitea
* Immich
* Grafana
* 自研 Spring Boot 项目

所有应用共用一套 CloudNativePG 集群。

对于单节点 Homelab 来说，这种模式资源占用最低，同时也保留了未来扩容到多节点 K3s 的能力。
