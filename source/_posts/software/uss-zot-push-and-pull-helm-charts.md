---
title: 内网环境 K3s 部署实战：从 Zot OCI 托管到 HTTPS 域名解析全记录
date: 2026-03-01 08:35:27
tags: [kubernetes, kite, k3s, helm, ingress, traefik]
categories: [软件]
---

# 前言

在内网环境下搭建云原生基础设施时，镜像仓库的认证、OCI 协议的适配以及 Ingress 的路由转发通常是最容易卡壳的地方。本文记录了将应用部署至 K3s 集群，并通过自定义域名实现 HTTPS 访问的全过程。

# 环境信息
1. Zot 仓库 (OCI): 192.168.31.220:5000
2. K3s 节点: 192.168.31.222
3. 服务域名: health-master-dev.damingerdaiinternal.com

# 标准部署流程

## 镜像仓库与 Helm 认证

由于 Zot 采用 OCI 标准存储 Helm Chart，部署前需先在本地终端完成登录：

```bash
# 登录 OCI 注册表
echo "YOUR_PASSWORD" | helm registry login 192.168.31.220:5000 -u admin --password-stdin --insecure
```

## 配置 K3s 节点免密拉取

为了让 K3s 能够拉取私有仓库的镜像，需在 192.168.31.222 节点上配置`/etc/rancher/k3s/registries.yaml`：

```yaml
mirrors:
  "192.168.31.220:5000":
    endpoint:
      - "http://192.168.31.220:5000"
configs:
  "192.168.31.220:5000":
    auth:
      username: admin
      password: YOUR_PASSWORD
```

配置完成后重启服务：sudo systemctl restart k3s。

## Chart 的打包与推送（OCI 原始起点）

在执行安装之前，我们需要将本地的 Chart 目录打包并推送到 Zot 仓库。OCI 模式下的推送与传统的 chartmuseum 不同，它直接利用了容器镜像的存储逻辑。

### 打包 Chart

首先，进入你的 Chart 目录（包含 `Chart.yaml` 的地方），执行打包命令： 

```bash
helm package health-master
```

执行后会生成一个 `health-master-1.0.0.tgz` 文件。

### 推送至 Zot 仓库

使用 helm push 命令。注意，这里的协议头必须是 `oci://`：

```bash
# 格式：helm push [文件名] oci://[仓库地址]/[项目名]
helm push health-master-1.0.0.tgz oci://192.168.31.220:5000/health-master \
  --plain-http
```

### 验证推送（避坑点）

推送成功后，你会看到类似下方的输出：

```bash
Pushed: 192.168.31.220:5000/health-master/health-master:1.0.0
Digest: sha256:c2859e4f82d5248c59b2716762acc8929abe07694576d13636da1eabc114145d
```

特别注意:
> 在 Zot 或其他支持 OCI 的仓库中，你会发现 Chart 和 Docker 镜像并排存在。如果你使用的是 Zot 的 UI，你会看到该 Artifact 的媒体类型（Media Type）为 application/vnd.cncf.helm.config.v1+json

![health master charts in zot registery](/images/software/zot-health-master-charts.png)

## 执行 Helm 安装

使用 OCI 协议安装时，针对内网 HTTP 环境，必须携带 --plain-http 参数：

```bash
helm install health-master-dev oci://192.168.31.220:5000/health-master/health-master \
  --version 1.0.0 \
  -f values.yaml \
  --plain-http \
  --namespace health-master-namespace-dev --create-namespace
```

## 暴露 HTTPS 服务

创建 Middleware 资源处理 HTTP 到 HTTPS 的强制跳转，并在 Ingress 中引用：

```yaml
# 1. 定义跳转中间件 (API 版本需根据 kubectl api-resources 确定)
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: force-https
  namespace: health-master-namespace-dev
spec:
  redirectScheme:
    scheme: https
    permanent: true

# 2. 配置 Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: health-master-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web, websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.middlewares: health-master-namespace-dev-force-https@kubernetescrd
spec:
  tls:
    - hosts: ["health-master-dev.damingerdaiinternal.com"]
      secretName: health-master-tls
  rules:
    - host: health-master-dev.damingerdaiinternal.com
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: health-master-dev-web
                port:
                  number: 3000
```

# 部署过程中的“血泪史”（踩坑排查）

在实际操作中，我们并没有上述流程那么顺利，先后经历了四次重大“卡壳”：

## Helm 参数的“名不对题”

坑位：在尝试解决认证问题时，习惯性给 helm install 加了 --insecure。

真相：Helm 的命令行设计中，--insecure 仅用于 login；在 install/push/pull OCI 仓库时，如果是非 HTTPS 环境，必须使用 --plain-http。否则会报“协议不支持”或“认证找不到”的错误。

## 僵死的 Namespace (Terminating)

坑位：删除旧部署后，Namespace 状态卡在 Terminating 几十分钟不消失，导致无法重新部署。

对策：这是由于 finalizers 机制拦截了资源释放。通过 kubectl edit ns `<namespace>` 强制手动删除 finalizers 数组内容，Namespace 会瞬间被强制销毁，从而解除锁定。

## 关于推送时的常见报错：

报错：401 Unauthorized：如果在 push 时遇到这个错误，请检查是否执行了 helm registry login。

报错：server responded with status 405 Method Not Allowed：这通常是因为你没有使用 oci:// 前缀，或者仓库没有开启 OCI 支持。

## K3s 节点的 401 Unauthorized
坑位：本地 helm login 成功了，应用也部署上去了，但 Pod 状态一直是 ImagePullBackOff。

定位：这是一个经典误区——“我登了不代表 K8s 登了”。Helm 客户端的认证信息存在用户家目录，而节点拉取镜像是由 Containerd 负责的。

解决：必须在 K3s 的全局配置文件 registries.yaml 中显式声明私有仓库的账号密码

## Ingress 的 404 谜团与中间件失效

坑位：Ingress 状态正常，ADDRESS 也有 IP，但访问域名总是返回 404 page not found。

### 深度排查：

通过 kubectl logs -n kube-system -l app.kubernetes.io/name=traefik 查看实时日志。

关键发现：日志报错 middleware does not exist。

原因：引用了不存在的内置中间件，导致 Traefik 拒绝加载整个路由规则。

API 陷阱：旧版教程常用 traefik.containo.us/v1alpha1，新版 K3s 需改为 traefik.io/v1alpha1。只有 Middleware 成功创建并正确引用，路由才会生效。

# 总结

内网环境的“纯手工”部署是对运维基本功的最好打磨。每一个 404 背后，可能都是 API 版本不匹配或认证链路断裂的体现。理解了 OCI 协议规范、K8s 资源回收机制 以及 Traefik 路由匹配逻辑，这些坑就不再是阻碍，而是经验