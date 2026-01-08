---
title: 轻量化 K8s 管理方案：在 k3s 中部署 Skooner 可视化面板
date: 2026-01-08 22:27:09
tags: [kubernetes, k8s, k3s, skooner]
categories: [软件]
---


## 前言

管理 Kubernetes 集群不应该以牺牲系统资源为代价。虽然官方 Dashboard 功能强大，但对于小型集群或单节点 **k3s** 环境来说，它往往显得过于臃肿。如果你正在寻找一个极致轻快、响应迅速的管理界面，**Skooner** 是一个完美的平衡点。

Skooner（原名 K8dash）是一款开源、超轻量级的实时仪表盘，它在不增加集群负担的前提下，提供了核心的资源观测能力。


## 为什么选择 Skooner？

Skooner 的设计哲学与 k3s 的轻量化理念完美契合：

1. **极低的资源占用**：它的内存占用通常不到 **50MB**，在你的系统监控中几乎可以忽略不计。
2. **实时状态感知**：基于 WebSocket 技术，无需手动刷新页面即可实时查看 CPU、内存消耗以及 Pod 状态。
3. **移动端自适应**：UI 采用响应式设计，即使在手机或平板浏览器上也能轻松排查集群问题。


## 部署步骤

在本指南中，我们将通过 **NodePort (32007)** 的方式部署并暴露 Skooner，以便你直接通过物理机 IP 访问。

### 1. 部署核心组件

首先，一键安装 Skooner 的命名空间、服务和部署资源：

```bash
kubectl apply -f https://raw.githubusercontent.com/skooner-k8s/skooner/master/kubernetes-skooner.yaml

```

### 2. 配置自定义端口 (32007)

默认情况下，Kubernetes 会分配一个随机的 NodePort。我们通过以下命令将其固定为 **32007**：

```bash
kubectl patch svc skooner -n kube-system --type='json' -p='[{"op": "replace", "path": "/spec/type", "value":"NodePort"}, {"op": "replace", "path": "/spec/ports/0/nodePort", "value":32007}]'

```

### 3. 创建管理 Token

Skooner 使用 Kubernetes 原生的 RBAC 权限。为了获得完整的管理权限，我们需要创建一个 ServiceAccount 并生成登录 Token：

```bash
# 创建 ServiceAccount 和 集群角色绑定
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: skooner-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: skooner-admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: skooner-admin
  namespace: kube-system
EOF

# 生成登录 Token（有效期 24 小时）
kubectl create token skooner-admin -n kube-system --duration=24h

```


## 如何访问

1. 打开浏览器，访问：`http://<你的服务器IP>:32007`。
2. 在登录界面选择 **"Token"** 模式。
3. 粘贴上一步生成的长字符串即可进入面板。

这是我本地的效果：

![sknooer部署效果](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/deploy-sknooer-on-k3s.png)

## 结语

Skooner 在保持 k3s 轻量化特性的同时，为你提供了一个直观、现代的操作界面。对于那些既想要可视化观测、又不想被复杂管理平台拖慢系统性能的开发者来说，Skooner 是一个“即插即用”的最佳方案。

如需了解更多高级配置，可以参考官方文档：[skooner.io](https://skooner.io/)。

## 一键安装脚本

```bash
# 1. Deploy Skooner resources
kubectl apply -f https://raw.githubusercontent.com/skooner-k8s/skooner/master/kubernetes-skooner.yaml

# 2. Patch the Service to NodePort and set it specifically to 32007
kubectl patch svc skooner -n kube-system --type='json' -p='[{"op": "replace", "path": "/spec/type", "value":"NodePort"}, {"op": "replace", "path": "/spec/ports/0/nodePort", "value":32007}]'

# 3. Create a Cluster Admin ServiceAccount for login (if not already created)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: skooner-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: skooner-admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: skooner-admin
  namespace: kube-system
EOF

# 4. Generate Login Token
echo "-------------------------------------------------------"
echo "Skooner is now accessible at port: 32007"
echo "Access URL: http://$(hostname -I | awk '{print $1}'):32007"
echo ""
echo "Login Token:"
kubectl create token skooner-admin -n kube-system --duration=24h
echo "-------------------------------------------------------"
```
