---
title: 玩转 K8s 面板：轻量级 Kite 安装全记录 (K3s 篇)
date: 2026-01-25 21:49:06
tags: [kubernetes, kite, k3s]
categories: [软件]
---

# 前言

在 Kubernetes 运维中，官方的 Dashboard 虽然功能齐全但显得有些沉重。如果你正在寻找一款**轻量、现代化、且安装简单**的监控面板，**Kite** 绝对是一个惊喜。本文将详细记录如何在 K3s 环境下安装 Kite，并配置 Ingress 域名访问以及集成 Prometheus 历史监控。



# 一、 环境准备

* **集群环境**：K3s (自带 Traefik Ingress Controller)
* **节点 IP**：`192.168.31.222`
* **目标域名**：`kite.damingerdaiinternal.com`


# 二、 部署 Kite 核心组件

首先，我们直接使用官方提供的清单将 Kite 安装到 `kube-system` 命名空间下：

```bash
kubectl apply -f https://raw.githubusercontent.com/zxh326/kite/refs/heads/main/deploy/install.yaml

```

## 1. 修改服务为 NodePort (可选)

为了方便最初的调试，我们可以将 Service 类型修改为 NodePort 并指定端口 `32008`：

```bash
kubectl patch svc kite -n kube-system -p '{"spec":{"type":"NodePort","ports":[{"port":8080,"targetPort":8080,"nodePort":32008}]}}'

```

## 2. 重置管理员密码

Kite 默认通过环境变量管理账户。如果你需要自定义或重置密码，可以执行：

```bash
kubectl set env deployment/kite KITE_ADMIN_USER=admin KITE_ADMIN_PASSWORD=your_new_password -n kube-system

```

# 三、 配置 Ingress 域名访问 (HTTPS)

由于 K3s 默认使用 Traefik，我们使用现代化的 `ingressClassName` 语法进行配置。

## 1. 创建自签名证书 Secret

为了支持 HTTPS，我们先为域名创建一个证书：

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=kite.damingerdaiinternal.com"

kubectl create secret tls kite-tls --cert=tls.crt --key=tls.key -n kube-system

```

## 2. 应用 Ingress 规则

创建 `kite-ingress.yaml`：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kite-ingress
  namespace: kube-system
spec:
  ingressClassName: traefik
  tls:
    - hosts: ["kite.damingerdaiinternal.com"]
      secretName: kite-tls
  rules:
    - host: kite.damingerdaiinternal.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kite
                port:
                  number: 8080

```

应用配置：`kubectl apply -f kite-ingress.yaml`。随后在本地 `hosts` 文件中添加 `192.168.31.222 kite.damingerdaiinternal.com` 即可访问。

![kite部署效果](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/k3s-kite-prometheus-setup-001.png)



# 四、 集成 Prometheus 开启历史监控

Kite 默认只显示来自 `metrics-server` 的实时数据，无法看到历史趋势。我们需要接入 Prometheus。

## 1. 安装 kube-prometheus-stack

使用 Helm 快速部署全套监控栈：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace

```

## 2. 配置 Kite 连接 Prometheus

找到 Prometheus 在集群内的 DNS 地址，并将其注入 Kite 的环境变量：

```bash
# DNS地址通常为：http://[服务名].[命名空间].svc.cluster.local:9090
kubectl set env deployment/kite PROMETHEUS_URL=http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090 -n kube-system

```
![kite监控效果](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/k3s-kite-prometheus-setup-002.png)

# 五、 总结

稍等几分钟，待 Prometheus 完成指标抓取后，刷新 Kite 界面，你就会发现原先的 "Limited historical data" 提示消失了，取而代之的是精美的 CPU 和内存历史曲线图。

**Kite + K3s + Traefik + Prometheus** 的组合，既保证了监控的深度，又维持了极低的资源消耗，非常适合中小规模集群或边缘计算场景。


# 💡 小贴士

* 如果发现 Ingress 无法解析，请检查 `kubectl get ingress -n kube-system` 的 ADDRESS 列。
* 如果图表长时间不出数据，请确认 `kube-state-metrics` 是否在 `monitoring` 命名空间下正常运行。