---
title: 在内网 Kubernetes 集群上使用 Helm 安装 Rancher 并支持域名访问
date: 2025-08-17 14:42:31
tags: [Kubernetes, Helm,Rancher]
categories: [软件]
---

# 前言

本指南将帮助您在 192.168.31.222 的内网服务器上，使用 Helm 安装 Rancher，并通过域名 rancher.damingerdaiinternal.com 访问。  

## 1. 前提条件

✅ 已部署 Kubernetes 集群（单节点或多节点均可）  
✅ 已安装 kubectl 和 Helm 3  
✅ 已配置 DNS 或 Hosts 解析（确保 rancher.damingerdaiinternal.com 指向 192.168.31.222）  
✅ 服务器开放 80 和 443 端口  

## 2. 安装 cert-manager（用于证书管理）

Rancher 依赖 cert-manager 管理 TLS 证书。

### 2.1 安装 cert-manager CRD

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
```


### 2.2 添加 Helm 仓库并安装 cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0
```

### 2.3 验证 cert-manager 是否运行

```bash
kubectl -n cert-manager get pods

预期输出：

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-7dd5554b4-abc12               1/1     Running   0          1m
cert-manager-cainjector-64dc9f6bff-xyz45   1/1     Running   0          1m
cert-manager-webhook-5f4d8b7b8d-pqr78      1/1     Running   0          1m
```

## 3. 生成自签名证书（适用于内网）

由于是内网环境，我们使用自签名证书。

### 3.1 生成 CA 根证书

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout ca.key -out ca.crt -subj "/CN=damingerdaiinternal.com CA"
```

### 3.2 生成服务器证书

```bash
openssl req -newkey rsa:4096 -sha256 -nodes \
  -keyout tls.key -out tls.csr -subj "/CN=rancher.damingerdaiinternal.com"
```


### 3.3 使用 CA 签名服务器证书

```bash
openssl x509 -req -sha256 -days 3650 \
  -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out tls.crt -extfile <(echo "subjectAltName=DNS:rancher.damingerdaiinternal.com")
```

## 4. 安装 Rancher

### 4.1 创建命名空间

```bash
kubectl create namespace cattle-system
```

### 4.2 创建 Kubernetes Secret 存储证书

```bash
kubectl -n cattle-system create secret tls tls-rancher-ingress \
  --cert=tls.crt --key=tls.key
kubectl -n cattle-system create secret generic tls-ca \
  --from-file=ca.crt
```

### 4.3 添加 Rancher Helm 仓库

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

### 4.4 使用 Helm 安装 Rancher

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.damingerdaiinternal.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=secret \
  --set privateCA=true \
  --set replicas=1
```

输出：

```bash
NAME: rancher
LAST DEPLOYED: Sun Aug 17 14:37:32 2025
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.

Check out our docs at https://rancher.com/docs/

If you provided your own bootstrap password during installation, browse to https://rancher.damingerdaiinternal.com to get started.

If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:

 
echo https://rancher.damingerdaiinternal.com/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
 

To get just the bootstrap password on its own, run:

 
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

## 5. 配置 DNS / Hosts 解析

### 5.1 方案 A：内网 DNS 服务器

在 DNS 服务器（如 Bind、Windows DNS）中添加：

```txt
rancher.damingerdaiinternal.com. IN A 192.168.31.222
```

### 5.2 方案 B：客户端 Hosts 文件

在需要访问的电脑上修改 /etc/hosts（Linux/macOS）或 C:\Windows\System32\drivers\etc\hosts（Windows）：

```txt
192.168.31.222 rancher.damingerdaiinternal.com
```


## 6. 验证安装

### 6.1 检查 Rancher Pod 状态

```bash
kubectl -n cattle-system get pods -l app=rancher

预期输出：

NAME                      READY   STATUS    RESTARTS   AGE
rancher-5d8f6c9b8-abc12   1/1     Running   0          2m
```

### 6.2 查看 Ingress 配置

```bash
kubectl -n cattle-system get ingress

预期输出：

NAME      HOSTS                              ADDRESS        PORTS
rancher   rancher.damingerdaiinternal.com    192.168.31.222 80, 443
```


### 6.3 访问 Rancher UI

在浏览器打开：

https://rancher.damingerdaiinternal.com

• 首次登录：用户名 admin，密码 admin  

• 证书警告：由于是自签名证书，浏览器会提示不安全，手动信任即可  

## 7. 卸载 Rancher

```bash
helm uninstall rancher -n cattle-system
kubectl delete namespace cattle-system
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
```

## 8. 常见问题

### Q1: 访问时提示 "证书不受信任"

✅ 解决方案：  
• Windows：双击 ca.crt 并安装到 "受信任的根证书颁发机构"  

• Linux/macOS：
  sudo cp ca.crt /usr/local/share/ca-certificates/
  sudo update-ca-certificates
  

### Q2: Rancher Pod 无法启动

✅ 检查日志：
kubectl -n cattle-system logs -f deployment/rancher

✅ 检查资源是否足够（Rancher 至少需要 2GB 内存）：
kubectl describe nodes


### Q3: 外网无法访问

✅ 检查防火墙：
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

✅ 如果是云服务器，检查安全组规则  

# 总结

✅ 已完成：
• 使用 Helm 安装 Rancher  

• 配置自签名证书  

• 支持域名 rancher.damingerdaiinternal.com 访问  

• 验证 HTTPS 安全性  

现在，您可以通过 https://rancher.damingerdaiinternal.com 访问 Rancher 管理 Kubernetes 集群！ 🎉