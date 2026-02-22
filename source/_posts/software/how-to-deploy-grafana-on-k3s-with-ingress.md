---
title: Grafana UI 部署与技术复盘报告
date: 2026-02-22 21:17:43
tags: [k3s, k8s, Kubernetes, Grafana, Ingress, Gateway Api]
categories: [软件]
---

# Grafana UI 部署与技术复盘报告

## 1. 基础环境

* **集群环境**: K3s (192.168.0.113)
* **监控套件**: `kube-prometheus-stack` (Helm 部署)
* **目标**: 通过正式域名 `grafana.damingerdaint.com` 暴露 Grafana UI 页面。

---

## 2. 成功方案：Ingress 部署实践

由于 K3s 内置的 Traefik 默认已占用 80 端口，采用 Ingress 方案避开了端口冲突，实现了快速上线。

### 配置实现 (v1 标准写法)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ui-ingress
  namespace: monitoring
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik  # 修复 Deprecated 警告的现代写法
  rules:
  - host: "grafana.damingerdaint.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80

```

### 运维操作指南

* **获取初始密码**:
```bash
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

```


* **密码修改途径**:
* **UI 方式**: 登录后进入 `User Profile` -> `Change Password` 修改。

## 3. 失败复盘：Gateway API 尝试记录

### 环境初始化 (K3s 开启支持)

在 K3s 上，我通过以下命令手动安装了标准 CRD 并配置了 Traefik 实验性开关：

```bash
# 1. 安装 Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# 2. 修改 Traefik 配置开启支持
# 编辑或创建 /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    providers:
      kubernetesGateway:
        enabled: true

```

### 尝试部署的 Bash 命令

```bash
# 1. 创建 GatewayClass
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
EOF

# 2. 创建 Gateway (监听 80 端口)
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: monitoring-gateway
  namespace: monitoring
spec:
  gatewayClassName: traefik
  listeners:
  - name: web
    protocol: HTTP
    port: 80
    hostname: "grafana.damingerdaint.com" #可能是不需要的
    allowedRoutes:
      namespaces:
        from: Same
EOF

# 3. 创建 HTTPRoute 尝试绑定
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana-route
  namespace: monitoring
spec:
  parentRefs: [{name: monitoring-gateway}]
  hostnames: ["grafana.damingerdaint.com"]
  rules:
  - matches: [{path: {type: PathPrefix, value: /}}]
    backendRefs: [{name: prometheus-grafana, port: 80}]
EOF

```

### 核心报错分析

* **错误状态**: `Status: False`, `Reason: NotAllowedByListeners`, `Message: PortUnavailable`。
* **原因排查**:
* **端口独占**: K3s 默认的 Traefik 服务在启动时已经通过其内部的 `entryPoint` 锁定了主机的 80 端口。
* **资源竞争**: Gateway API 作为一个新的控制平面，在没有特殊配置的情况下，无法与已有的 Ingress 共享同一个 `entryPoint`。


## 4. 后续研究课题 (New K3s Node Plan)

在下一步全新部署的 K3s 环境中，重点研究方向：

1. **解耦 Traefik**: 安装 K3s 时通过 `--disable traefik` 彻底解放 80/443 端口。
2. **共享 Entrypoints**: 探索如何通过 `HelmChartConfig` 映射 Gateway 到现有的 Traefik 端口。
