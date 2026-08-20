---
title: 使用 FluxCD 构建 Homelab GitOps：从私有 Git 仓库到 Hoteler 自动部署
date: 2026-08-20 20:22:29
tags: [Kubernetes, GitOps, FluxCD, Gitea, DevOps]
categories: [软件]
---

# 使用 FluxCD 构建 Homelab GitOps：从私有 Git 仓库到 Hoteler 自动部署

最近我重新整理了自己的 Homelab 部署方式。

过去 Kubernetes 应用虽然已经使用 Kustomize管理，但本质上仍然需要人为执行部署操作。既然集群已经运行FluxCD，更合理的方式应该是：

> Git 描述期望状态，FluxCD 负责让 Kubernetes 的实际状态持续向 Git
> 中的状态收敛。

这次以 `Hoteler` 为例，将现有 Kustomize 部署配置接入FluxCD，并将应用源码、应用部署配置和集群配置拆分。

## 1. 仓库职责划分

整个 GitOps 体系拆成三个部分：

### Application Repository

`hoteler` 是应用源码仓库，负责：

``` text
Source Code
Dockerfile
Tests
CI
Container Image
```

CI 最终产生类似：

``` text
ghcr.io/damingerdai/hoteler:v0.0.5
```

这样的应用镜像。

应用仓库本身不直接操作 Kubernetes。

### Application GitOps Repository

独立的 `hoteler-deployment` 仓库负责描述：

> Hoteler 应该以什么状态运行。

目录类似：

``` text
hoteler-deployment/
└── hoteler-api/
    ├── base/
    │   ├── application.yml
    │   ├── cronjob.yaml
    │   ├── deployment.yaml
    │   ├── hpa.yaml
    │   ├── kustomization.yaml
    │   └── service.yaml
    └── overlays/
        ├── homepage/
        ├── lan/
        └── prod/
```

这里使用 Kustomize 的 Base + Overlay 模式管理不同环境。

### Fleet Repository

`fleet-infra` 负责回答：

> 某个 Kubernetes Cluster 应该运行哪些应用？

例如：

``` text
fleet-infra/
└── clusters/
    └── 192.168.31.222/
        ├── kustomization.yaml
        ├── flux-system/
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        └── apps/
            ├── hoteler-source.yaml
            ├── hoteler-api.yaml
            └── kustomization.yaml
```

三个仓库的职责：

``` text
hoteler
    ↓
软件是什么

hoteler-deployment
    ↓
软件应该怎么运行

fleet-infra
    ↓
哪些集群应该运行这个软件
```

## 2. GitRepository 与 Kustomization

FluxCD 中的核心关系可以理解为：

``` text
GitRepository
      ↓
Kustomization
      ↓
Kubernetes Resources
```

`GitRepository` 解决：

> Flux 去哪里获取配置？

`Kustomization` 解决：

> 获取 Repository 后，应该部署其中哪个目录？

这里需要区分两个同名但不同的 `Kustomization`。

Flux CRD：

``` yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
```

负责告诉 Flux 持续 reconcile Git 中的某个目录。

Kustomize 自己的配置：

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
```

负责组合 Deployment、Service、ConfigMap 等 Kubernetes Manifest。

因此实际关系是：

``` text
Flux GitRepository
        ↓
Flux Kustomization
        ↓
Git Directory
        ↓
Kustomize kustomization.yaml
        ↓
Deployment / Service / ConfigMap / HPA / ...
```

## 3. 接入私有 GitOps Repository

`hoteler-deployment` 位于 Homelab 内部的私有 Git 服务：

``` text
ssh://git@192.168.31.220:1022/hoteler/hoteler-deployment.git
```

首先为 Flux 创建 SSH Credential：

``` bash
flux create secret git hoteler-deployment-auth \
  --url=ssh://git@192.168.31.220:1022/hoteler/hoteler-deployment.git \
  --namespace=flux-system
```

Flux 会生成 SSH Deploy Key，并创建：

``` text
Secret/hoteler-deployment-auth
```

将生成的 Public Key 添加到私有 Git Repository 的 Deploy Keys 中即可。

目前 Flux 只需要读取部署配置，因此只需要 Read 权限。

## 4. 创建 GitRepository

在 `fleet-infra` 中创建：

``` text
clusters/192.168.31.222/apps/hoteler-source.yaml
```

``` yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: hoteler-deployment
  namespace: flux-system
spec:
  interval: 1m
  url: ssh://git@192.168.31.220:1022/hoteler/hoteler-deployment.git
  ref:
    branch: master
  secretRef:
    name: hoteler-deployment-auth
```

形成：

``` text
GitRepository/hoteler-deployment
            │
            │ secretRef
            ▼
Secret/hoteler-deployment-auth
            │
            │ SSH
            ▼
hoteler-deployment.git
```

## 5. 让 Fleet Repository 加载应用

`clusters/192.168.31.222/apps/kustomization.yaml`：

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - hoteler-source.yaml
  - hoteler-api.yaml
```

Cluster 根目录：

``` text
clusters/192.168.31.222/kustomization.yaml
```

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - flux-system
  - apps
```

这样 Flux Bootstrap 管理 `clusters/192.168.31.222` 时，就能继续发现
`apps` 中声明的资源。

## 6. 验证私有 Git Repository

通过：

``` bash
flux get sources git -A
```

检查：

``` text
NAMESPACE     NAME                 REVISION                READY
flux-system   flux-system          main@sha1:a28e0e8c      True
flux-system   hoteler-deployment   master@sha1:333bb97b    True
```

`hoteler-deployment READY=True` 意味着：

``` text
Flux
 ↓
SSH Credential
 ↓
Private Git Server
 ↓
hoteler-deployment
```

整条 Source 链路已经成功建立。

## 7. 使用 Flux Kustomization 部署 Hoteler

增加：

``` text
clusters/192.168.31.222/apps/hoteler-api.yaml
```

``` yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: hoteler-api
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: hoteler-deployment
  path: ./hoteler-api/overlays/lan
  prune: true
  wait: true
  timeout: 5m
```

组合起来表达的是：

> 使用 `hoteler-deployment` Repository 中的 `hoteler-api/overlays/lan`
> 作为当前集群中 Hoteler API 的期望状态。

``` text
fleet-infra
     ↓
Kustomization/hoteler-api
     ↓
GitRepository/hoteler-deployment
     ↓
hoteler-deployment.git
     ↓
hoteler-api/overlays/lan
     ↓
Kustomize
     ↓
Kubernetes
```

## 8. Base 与 Overlay 的 Namespace 设计

部署过程中遇到的一个问题是 Namespace。

最初 Base 中存在：

``` yaml
namespace: hoteler-namespace
```

但 LAN 环境希望部署到：

``` text
hoteler-dev-namespace
```

对于这个项目，更合理的职责是：

``` text
base
    ↓
描述 Hoteler 本身

overlay
    ↓
描述 Hoteler 在具体环境中的差异
```

既然 Namespace 属于环境差异，就由 Overlay 决定。

Base：

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml
  - cronjob.yaml

configMapGenerator:
  - name: hoteler-api-config
    files:
      - application.yml
```

LAN 环境增加：

``` text
overlays/lan/namespace.yaml
```

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hoteler-dev-namespace
```

然后：

``` text
overlays/lan/kustomization.yaml
```

``` yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: hoteler-dev-namespace

resources:
  - ../../base
  - namespace.yaml
```

最终生成：

``` text
Namespace/hoteler-dev-namespace

hoteler-dev-namespace
├── Deployment/hoteler-api
├── Service/hoteler-api
├── ConfigMap/hoteler-api-config-xxxxx
├── HPA
└── CronJob
```

## 9. 实际踩坑：Base 中固定 Namespace

最开始 Base 中存在：

``` yaml
namespace: hoteler-namespace
```

导致 `configMapGenerator` 生成的 ConfigMap 仍然属于：

``` text
hoteler-namespace
```

而 LAN Overlay 创建的是：

``` text
hoteler-dev-namespace
```

Flux 因此报告：

``` text
ConfigMap/hoteler-namespace/hoteler-api-config-td7c9kg7m5 not found:
namespaces "hoteler-namespace" not found
```

删除 Base 中固定的 Namespace，让 Overlay 统一决定环境 Namespace 后解决。

这里得到的经验是：

> Base 尽可能描述环境无关的应用结构；真正因环境不同而变化的内容交给
> Overlay。

## 10. Flux 不需要手动 Reconcile

调试时可以执行：

``` bash
flux reconcile kustomization flux-system --with-source
```

立即触发 reconciliation。

但它不是正常部署流程必须执行的命令。

正常流程应该是：

``` text
git push
    ↓
Flux Source Controller
    ↓
发现新的 Git Revision
    ↓
Kustomization Reconcile
    ↓
Kubernetes
```

所以日常修改 `hoteler-deployment` 后只需要：

``` bash
git add .
git commit
git push
```

Flux 会自动完成后续操作。

可以使用：

``` bash
flux get kustomizations -A --watch
```

观察 reconciliation。

GitOps 最终希望达到的效果就是：

``` text
git push
 ↓
结束
```

而不是：

``` text
git push
 ↓
SSH Server
 ↓
kubectl apply
```

## 11. 最终链路

修复 Namespace 后，Flux 成功创建 `hoteler-dev-namespace`，并创建 Hoteler
Deployment。

Pod 被 Kubernetes Scheduler 分配到节点后，kubelet 开始拉取：

``` text
ghcr.io/damingerdai/hoteler:v0.0.5
```

完整链路：

``` text
fleet-infra
      │
      ▼
FluxCD
      │
      ▼
GitRepository/hoteler-deployment
      │
      │ SSH
      ▼
Private Git Server
      │
      ▼
hoteler-deployment
      │
      ▼
hoteler-api/overlays/lan
      │
      ▼
Kustomize
      │
      ▼
Namespace / Deployment / Service
ConfigMap / CronJob / HPA
      │
      ▼
Kubernetes Pod
      │
      ▼
ghcr.io/damingerdai/hoteler:v0.0.5
```

## 12. 当前架构总结

最终三个 Repository 分别回答三个问题：

### `hoteler`

> 我要构建什么软件？

负责源码、测试、CI 和 Container Image。

### `hoteler-deployment`

> 这个软件应该怎么运行？

负责 Kustomize Base、Overlay、镜像版本和环境配置。

### `fleet-infra`

> 哪个 Cluster 应该运行这个软件？

负责 FluxCD、Cluster 和 Application GitRepository/Kustomization。

把三个职责拆开以后，整个部署模型更加清晰。

## 13. 下一步

下一步可以继续引入 Flux Image Automation：

``` text
ImageRepository
      ↓
ImagePolicy
      ↓
ImageUpdateAutomation
      ↓
更新 hoteler-deployment
      ↓
Flux Reconcile
      ↓
Rolling Update
```

最终实现：

``` text
Application Commit
      ↓
CI
      ↓
Build & Push Image
      ↓
Flux Image Automation
      ↓
Update GitOps Repository
      ↓
FluxCD
      ↓
Kubernetes Rolling Update
```

从而形成完整的 GitOps 发布闭环。
