---
title: 在 Minikube 上部署 Rancher：避坑指南与实战教程
date: 2026-01-20 20:34:37
tags: [kubernetes, minikube, rancher]
categories: [软件]
---


## 前言

在本地环境（如 Minikube）部署 Rancher 时，最常见的两个问题是**资源分配不足**和**证书签发卡死**。本文将记录如何通过宿主机 IP（192.168.31.133）和固定 NodePort（30081）成功搭建 Rancher。

# 一、 环境准备

* **宿主机 IP**: `192.168.31.133`
* **目标访问端口**: `30081`

## 1. 启动 Minikube (关键参数)

在 Docker 驱动下，Minikube 运行在容器内。为了让宿主机能直接访问 NodePort，必须在启动时通过 `--ports` 参数建立隧道映射。

```bash
# 建议先清理旧环境
minikube delete

# 启动并映射 30081 端口，分配至少 8GB 内存（Rancher 较重，建议给足）
minikube start --cpus 4 --memory 8192 --driver=docker --ports=30081:30081

```

## 2. 安装 Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

```


# 二、 部署 cert-manager (证书基础)

Rancher 依赖 cert-manager 来处理 SSL 证书。请确保组件完全运行后再进行下一步。

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

# 安装 CRDs
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.crds.yaml

# 安装组件
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.0

```

# 三、 安装 Rancher 与 sslip.io 的秘密

### 1. 执行安装命令

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.192.168.31.133.sslip.io \
  --set bootstrapPassword=admin \
  --set service.type=NodePort

```

## 2. 深度解析：为什么用 sslip.io？

在命令中我们设置了 `hostname=rancher.192.168.31.133.sslip.io`。

* **为什么填它？** Rancher 强制要求一个域名来生成证书和配置内部路由。`sslip.io` 是一个免费服务，它会自动将 `192.168.31.133.sslip.io` 解析到你的宿主机 IP。
* **为什么实际没用到？** 虽然证书是发给这个域名的，但我们通过 `NodePort` 直接访问 IP 也能打开页面。不过，**强烈建议保留此设置**，因为后续 Rancher 纳管其他集群时，会使用这个域名生成注册 URL。


# 四、 性能优化与网络微调 (避坑核心)

## 1. 缩减副本数：解决启动卡死

Rancher 默认启动 3 个副本，在单机 Minikube 上会造成严重的资源竞争，导致 Pod 长期处于 `0/1 Ready`。**初始化时请务必缩减为 1 个：**

```bash
kubectl scale deploy rancher -n cattle-system --replicas=1

```

## 2. 固定 NodePort 端口

默认端口是随机的，我们需要手动将其固定为映射好的 `30081`：

```bash
kubectl patch svc rancher -n cattle-system -p '{"spec":{"ports":[{"port":443,"targetPort":443,"nodePort":30081,"name":"https-internal"}]}}'

```

# 五、 状态排查

运行 `kubectl get certificate -n cattle-system`。
如果 **READY** 为 **True**，说明证书签发成功。此时只需静候 2 分钟，Pod 就会变为 `1/1 Ready`。


# 附录：Rancher 健康检查脚本

为了方便大家快速定位部署问题，可以保存以下脚本 `check_rancher.sh`：

```bash
#!/bin/bash

echo "===== 1. 检查 Cert-Manager 状态 ====="
kubectl get pods -n cert-manager

echo -e "\n===== 2. 检查证书签发状态 ====="
kubectl get certificate -n cattle-system

echo -e "\n===== 3. 检查 Rancher Pod 运行状态 ====="
kubectl get pods -n cattle-system -l app=rancher

echo -e "\n===== 4. 检查服务端口映射 ====="
kubectl get svc -n cattle-system rancher

echo -e "\n===== 5. 获取初始登录密码 ====="
kubectl get secret --namespace cattle-system bootstrap-secret -o jsonpath={.data.bootstrapPassword} | base64 --decode && echo ""

echo -e "\n[提示] 请访问: https://192.168.31.133:30081"

```

**使用方法：**
`chmod +x check_rancher.sh && ./check_rancher.sh`

---

**博主结语：**
在 Minikube 上跑 Rancher 是一件很酷的事，但一定要给够内存并手动控制副本数。希望这篇教程能让你少走弯路！