---
title: 从 Demo 到生产级：Kite 在 K3s 上的 PostgreSQL 与 Ingress 部署实战
date: 2026-04-05 17:27:48
tags: [kubernetes, kite, k3s]
categories: [软件]
---


# 从 Demo 到生产级：Kite 在 K3s 上的高可用数据库与 HTTPS 部署实践

### 引言

在之前的文章 [《玩转 K8s 面板：轻量级 Kite 安装全记录 (K3s 篇)》](https://blog.damingerdai.com/2026/01/25/software/setup-kite-with-prometheus-on-k3s/) 中，我们快速搭建了测试环境。但随着使用深入，原方案的局限性开始显现。今天我们将进行一次彻头彻尾的“生产级”改造。

-----

## 1\. 原方案的痛点：为什么不能用于生产？

原方案最大的问题在于**数据持久化**。

  * **默认 SQLite 的风险**：原方案依赖 Pod 内部或本地路径的 SQLite 数据库。在 Kubernetes 环境下，Pod 是随毁随建的（Stateless）。
  * **数据丢失**：一旦节点重启、Pod 漂移或进行版本升级，所有用户配置、AI 工作流数据将**全部丢失**。
  * **不可扩展性**：SQLite 不支持多实例并发写入，限制了未来的水平扩展。

-----

## 2\. 外部数据库集成：计算与存储分离

为了实现数据持久化，我们将数据库移出 K3s 集群，部署在独立的服务器（`192.168.31.220`）上。

### 2.1 数据库初始化

在 PG 服务器上执行：

```sql
CREATE DATABASE kite_db;
CREATE USER kite_user WITH PASSWORD '20260405';
-- CREATE DATABASE kite_db OWNER kite_user; 同步创建数据库
ALTER DATABASE kite_db OWNER TO kite_user;
```

### 2.2 神奇的“桥梁”：Service & EndpointSlice

为了让集群内的 Pod 能通过 `external-pg` 这个域名访问外部 IP，我们利用 K8s 的服务发现机制：

```yaml
# external-db.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-pg
  namespace: kite-system
spec:
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-pg-slice
  namespace: kite-system
  labels:
    kubernetes.io/service-name: external-pg
addressType: IPv4
ports:
  - protocol: TCP
    port: 5432
endpoints:
  - addresses: ["192.168.31.220"]
```
> Kubernetes 正在逐步废弃旧的 Endpoints 资源，转而推荐使用更高效的 EndpointSlice。虽然旧的 Endpoints 目前依然能工作，但为了符合生产环境的长期维护标准（以及消除那个烦人的 Warning），我们直接按照新标准来。

-----

## 3\. 安全访问：Ingress 与 TLS 证书

生产环境离不开 HTTPS。我们通过 Traefik Ingress 配置自签名证书。

### 3.1 创建 TLS Secret

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=kite.damingerdailocal.com"

kubectl create secret tls kite-tls-secret --cert=tls.crt --key=tls.key -n kite-system
```

### 3.2 刷新本地 DNS (macOS)

在 `/etc/hosts` 添加 `192.168.31.98 kite.damingerdailocal.com` 后，别忘了执行刷新：

```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

-----

## 4\. 终极部署：Helm 一键点火

通过自定义 `values.yaml` 覆盖默认配置，禁用 SQLite 并启用 PG 和 Ingress。

```yaml
db:
  type: postgres
  dsn: "postgres://kite_user:20260405@external-pg:5432/kite_db?sslmode=disable"

sqlite:
  persistence:
    enabled: false

ingress:
  enabled: true
  className: "traefik"
  tls:
    - secretName: kite-tls-secret
      hosts: ["kite.damingerdailocal.com"]
```

执行部署：

```bash
helm upgrade --install kite kite/kite --namespace kite-system --create-namespace -f values.yaml
```

![kite效果图](/images/kite-production-setup-with-external-postgres-and-ingress/kite-production-setup-with-external-postgres-and-ingress-1.png)

> 在 Helm 的设计中，upgrade --install 是生产环境最常用的命令, 如果 Kite 还没安装：它会执行 安装 (Install) 动作，创建一个新的 Release;如果 Kite 已经安装：它会对比 my-values.yaml 和当前的配置，执行 更新 (Upgrade) 动作。

-----

## 5\. 避坑总结与经验谈

在这次迁移中，我遇到了几个典型问题，希望能帮到大家：

  * **Namespace 依赖陷阱**：直接 `apply` 带有 namespace 属性的资源时，若命名空间不存在会报错。**经验：** 在 YAML 头部包含 Namespace 定义，或先执行 `kubectl create ns`。
  * **残留资源冲突**：清理 `default` 命名空间的旧 Service 时，`Endpoints` 资源通常会随之自动删除。如果遇到 `NotFound` 报错，说明 K8s 已经帮你处理好了。
  * **Readiness Probe 假死**：Pod 启动后显示 `0/1 READY`，通常是正在执行数据库 Migration。**经验：** 通过 `kubectl logs` 观察是否有 `CREATE TABLE` 动作，不要急着重启。
  * **外部访问阻碍**：如果 `ping` 通了但 `curl` 不通，大概率是目标机器（192.168.31.220）的防火墙没放行 5432 端口。

### 结语

通过这次改造，Kite 已经具备了应对生产环境的基础能力。数据有了保障，访问有了加密。
