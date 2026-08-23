---
title: 使用 FluxCD + GitHub + SOPS + age 构建完整 GitOps 部署流程
date: 2026-08-23 16:41:09
tags: [Kubernetes, GitOps, FluxCD, Github, DevOps]
categories: [软件]
---

# 使用 FluxCD + GitHub + SOPS + age 构建完整 GitOps 部署流程

最近我将自己的 Kubernetes 部署方式从手工 Helm 部署逐步迁移到了 FluxCD 管理的 GitOps 模式。

最终希望实现的目标是：

```text
GitHub
  │
  ├── fleet-infra
  │      └── 管理集群以及 FluxCD 配置
  │
  └── hoteler-deployment
         └── 管理 Hoteler 的 Kubernetes manifests
                  │
                  ▼
               FluxCD
                  │
                  ▼
                 K3s
```

其中两个 GitHub Repository 都是 Private Repository。

同时，由于 FluxCD 访问 `hoteler-deployment` 需要 SSH Deploy Key，因此又引出了一个新的问题：

> GitOps 仓库本身应该如何安全地管理 Secret？

最终我选择了：

```text
FluxCD + GitHub + SSH Deploy Key + SOPS + age
```

来完成整个部署流程。

本文记录完整配置过程，以及过程中遇到的一个比较典型的 SOPS Bootstrap “鸡生蛋”问题。

---

## 一、整体架构

最终的结构大致如下：

```text
GitHub
│
├── fleet-infra (Private)
│   │
│   ├── FluxCD 配置
│   ├── GitRepository
│   ├── Kustomization
│   └── SOPS 加密后的 Secret
│
└── hoteler-deployment (Private)
    │
    └── hoteler-api
        ├── base
        └── overlays
            └── 107.172.148.131
                    │
                    ▼
                  FluxCD
                    │
                    ▼
                   K3s
```

这里我把职责进行了拆分。

`fleet-infra` 负责：

```text
集群级 GitOps 配置
FluxCD
GitRepository
Kustomization
Secret
```

`hoteler-deployment` 负责：

```text
应用 Kubernetes manifests
```

这样应用部署配置和集群基础设施配置不会完全混在一起。

---

# 二、使用 Flux Bootstrap 接入 GitHub

首先准备 GitHub Personal Access Token。

我使用的是 GitHub Fine-grained Personal Access Token。

环境变量：

```bash
export GITHUB_TOKEN="github_pat_xxxxxxxxx"
```

然后执行：

```bash
flux bootstrap github \
  --owner=damingerdai \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/107.172.148.131 \
  --personal
```

这里有一个小坑。

一开始我使用：

```bash
--owner=$GITHUB_USER
```

但是没有正确设置：

```bash
GITHUB_USER
```

于是出现：

```text
failed to get Git repository "https://github.com//fleet-infra"

provider error:
validation error for UserRepositoryRef.UserLogin:
field is required
```

注意错误 URL：

```text
https://github.com//fleet-infra
                   ↑
```

这里实际上已经说明 owner 是空的。

如果不想额外维护 `GITHUB_USER`，直接写：

```bash
--owner=damingerdai
```

即可。

Bootstrap 成功之后，GitHub 中的：

```text
fleet-infra
```

就成为 FluxCD 的 Source of Truth。

目录类似：

```text
fleet-infra/
└── clusters/
    └── 107.172.148.131/
        └── flux-system/
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
```

检查：

```bash
flux get sources git
```

可以看到：

```text
NAME          REVISION          READY
flux-system   main@sha1:...     True
```

---

# 三、让 FluxCD 读取另一个 GitHub Private Repository

我的应用 Kubernetes 配置并不在 `fleet-infra`，而是在另一个私有仓库：

```text
damingerdai/hoteler-deployment
```

因此 FluxCD 还需要获得读取这个仓库的权限。

这里没有继续使用 GitHub PAT，而是使用：

```text
SSH Deploy Key
```

这样可以把权限严格限制在：

```text
hoteler-deployment
```

这一个 Repository。

## 生成 Deploy Key

```bash
mkdir -p ~/.ssh/flux

ssh-keygen \
  -t ed25519 \
  -C "flux-hoteler-deployment" \
  -f ~/.ssh/flux/hoteler-deployment \
  -N ""
```

得到：

```text
~/.ssh/flux/hoteler-deployment
~/.ssh/flux/hoteler-deployment.pub
```

其中：

```text
hoteler-deployment
```

是私钥。

```text
hoteler-deployment.pub
```

是公钥。

将公钥添加到：

```text
GitHub
→ hoteler-deployment
→ Settings
→ Deploy keys
→ Add deploy key
```

因为 FluxCD 目前只负责读取配置，所以：

```text
Allow write access
```

不需要开启。

测试：

```bash
ssh \
  -i ~/.ssh/flux/hoteler-deployment \
  -T git@github.com
```

GitHub 返回：

```text
Hi damingerdai/hoteler-deployment!
You've successfully authenticated,
but GitHub does not provide shell access.
```

说明 Deploy Key 已经正常工作。

---

# 四、为什么还需要 SOPS + age

现在出现了另一个问题。

FluxCD 要读取：

```text
hoteler-deployment
```

就必须持有：

```text
SSH Private Key
```

最简单的做法当然是：

```bash
kubectl create secret ...
```

但是这样做会导致 GitOps 不完整。

如果未来服务器重装：

```text
K3s 重建
   ↓
Flux Bootstrap
   ↓
fleet-infra 恢复
   ↓
缺少 SSH Secret
   ↓
无法读取 hoteler-deployment
```

还是需要人工重新创建 Secret。

因此我希望：

> Secret 也能够由 fleet-infra 管理。

但又绝对不能把：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
```

直接提交到 Git。

这就是 SOPS + age 要解决的问题。

---

# 五、SOPS 和 age 分别是什么

可以简单理解为：

```text
SOPS
负责：
如何加密 YAML / JSON 等配置

age
负责：
使用什么密钥进行加密和解密
```

组合起来：

```text
secret.yaml
    │
    ▼
   SOPS
    │
    │ age public key
    ▼
encrypted secret.yaml
    │
    ▼
GitHub
```

FluxCD 中保存：

```text
age private key
```

因此 Flux 可以：

```text
GitHub
   ↓
读取 SOPS Secret
   ↓
age private key
   ↓
解密
   ↓
Kubernetes Secret
```

---

# 六、安装 SOPS 和 age

我的服务器使用 Debian。

安装 age：

```bash
apt install age
```

确认：

```bash
age --version
```

例如：

```text
1.1.1
```

安装 SOPS：

```bash
curl -LO \
  https://github.com/getsops/sops/releases/download/v3.13.3/sops-v3.13.3.linux.amd64

mv sops-v3.13.3.linux.amd64 /usr/local/bin/sops

chmod +x /usr/local/bin/sops
```

确认：

```bash
sops --version
```

---

# 七、生成 age Key

创建：

```bash
mkdir -p ~/.config/sops/age

age-keygen \
  -o ~/.config/sops/age/keys.txt
```

里面包含：

```text
# public key: age1xxxxxxxxxxxx

AGE-SECRET-KEY-1XXXXXXXXXXXXXXXX
```

其中：

```text
age1xxxx
```

是公钥，可以公开。

但是：

```text
AGE-SECRET-KEY-1...
```

是私钥，不能提交 Git。

---

# 八、让 Flux 获得 age Private Key

创建 Kubernetes Secret：

```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey="$HOME/.config/sops/age/keys.txt"
```

检查：

```bash
kubectl get secret sops-age -n flux-system
```

例如：

```text
NAME       TYPE     DATA
sops-age   Opaque   1
```

这个 Secret 是整个 GitOps Bootstrap 中少数仍然需要在初始化阶段注入的 Secret。

---

# 九、配置 SOPS

在 `fleet-infra` 根目录创建：

```text
.sops.yaml
```

内容：

```yaml
creation_rules:
  - path_regex: .*\.secret\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1xxxxxxxxxxxxxxxx
```

这里的：

```text
age1xxxx
```

就是刚才生成的 age Public Key。

以后：

```text
*.secret.yaml
```

文件中的：

```yaml
data:
```

或者：

```yaml
stringData:
```

都会被 SOPS 加密。

---

# 十、让 Flux Kustomization 支持 SOPS

Flux Bootstrap 生成的：

```text
clusters/107.172.148.131/flux-system/gotk-sync.yaml
```

中存在：

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
```

需要增加：

```yaml
decryption:
  provider: sops
  secretRef:
    name: sops-age
```

最终：

```yaml
spec:
  interval: 10m0s

  path: ./clusters/107.172.148.131

  prune: true

  sourceRef:
    kind: GitRepository
    name: flux-system

  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

---

# 十一、一个非常重要的坑：SOPS Bootstrap 鸡生蛋问题

这里遇到了整个配置过程中最值得记录的问题。

我第一次配置的时候，在**同一个 Git revision** 中同时提交了：

```text
① flux-system 增加 SOPS decryption

② SOPS 加密的 hoteler-deployment-auth Secret
```

理论上看起来没问题。

但实际上产生了一个循环依赖。

当前 Kubernetes 中正在运行的是：

```text
旧 flux-system Kustomization
```

它还没有：

```yaml
decryption:
  provider: sops
```

然后它从 GitHub 拉到了新的 revision。

新的 revision 同时包含：

```text
新的 SOPS 配置
+
SOPS encrypted Secret
```

Flux 在 reconcile 时发现：

```text
Secret/flux-system/hoteler-deployment-auth
is SOPS encrypted
```

但是当前运行中的 Kustomization 还没有 SOPS 解密能力。

于是直接报错：

```text
Secret/flux-system/hoteler-deployment-auth is SOPS encrypted,
configuring decryption is required for this secret to be reconciled
```

问题就在这里。

Flux 想 apply：

```text
新的 decryption 配置
```

需要先 reconcile 这个 revision。

但是 reconcile 这个 revision 又需要：

```text
新的 decryption 配置
```

于是形成：

```text
需要 SOPS
   │
   ▼
才能 apply 新 revision
   │
   ▼
新 revision 才能启用 SOPS
   │
   └──────────────┐
                  │
                  ▼
               死循环
```

这就是一个典型的 Bootstrap 鸡生蛋问题。

---

# 十二、临时解决办法

当时直接手动 patch 当前 Kubernetes 中运行的 Kustomization：

```bash
kubectl patch kustomization flux-system \
  -n flux-system \
  --type=merge \
  -p '{
    "spec": {
      "decryption": {
        "provider": "sops",
        "secretRef": {
          "name": "sops-age"
        }
      }
    }
  }'
```

检查：

```bash
kubectl get kustomization flux-system \
  -n flux-system \
  -o yaml | grep -A5 decryption
```

看到：

```yaml
decryption:
  provider: sops
  secretRef:
    name: sops-age
```

然后：

```bash
flux reconcile kustomization flux-system
```

这一次成功：

```text
✔ applied revision main@sha1:ee907279...
```

---

# 十三、如何从一开始避免这个问题

正确做法其实非常简单：

> **不要在同一个 commit 中同时启用 SOPS 和添加第一个 SOPS Secret。**

应该分成两个阶段。

## Commit 1：只启用 SOPS

首先手动创建：

```bash
kubectl create secret generic sops-age \
  -n flux-system \
  --from-file=age.agekey="$HOME/.config/sops/age/keys.txt"
```

然后 Git 中只修改：

```text
gotk-sync.yaml
```

增加：

```yaml
decryption:
  provider: sops
  secretRef:
    name: sops-age
```

提交：

```bash
git commit -m "feat: enable SOPS decryption"
git push
```

然后：

```bash
flux reconcile kustomization flux-system
```

确认：

```bash
kubectl get kustomization flux-system \
  -n flux-system \
  -o yaml | grep -A5 decryption
```

确定 Kubernetes 中实际运行的 Kustomization 已经具有 SOPS 能力。

## Commit 2：再添加 SOPS Secret

这时候再提交：

```text
hoteler-deployment-auth.secret.yaml
```

Flux 已经具备解密能力，因此不会再产生循环依赖。

所以以后重新 Bootstrap 集群时，我会固定采用：

```text
1. flux bootstrap github

2. 创建 sops-age Secret

3. Git Commit：
   enable SOPS decryption

4. 等待 flux-system READY=True

5. Git Commit：
   encrypted secrets

6. Git Commit：
   workloads / external GitRepository
```

---

# 十四、使用 SOPS 加密 GitHub Deploy Key

首先生成 Kubernetes Secret YAML：

```bash
kubectl create secret generic hoteler-deployment-auth \
  --namespace=flux-system \
  --from-file=identity="$HOME/.ssh/flux/hoteler-deployment" \
  --from-file=identity.pub="$HOME/.ssh/flux/hoteler-deployment.pub" \
  --from-literal=known_hosts="$(ssh-keyscan github.com 2>/dev/null)" \
  --dry-run=client \
  -o yaml \
  > clusters/107.172.148.131/hoteler-deployment-auth.secret.yaml
```

注意：

> 此时这个文件仍然包含可以还原的 Secret，不能提交 Git。

然后：

```bash
sops --encrypt --in-place \
  clusters/107.172.148.131/hoteler-deployment-auth.secret.yaml
```

检查：

```bash
grep "ENC\[AES256_GCM" \
  clusters/107.172.148.131/hoteler-deployment-auth.secret.yaml
```

可以看到：

```yaml
identity: ENC[AES256_GCM,data:...]
identity.pub: ENC[AES256_GCM,data:...]
known_hosts: ENC[AES256_GCM,data:...]
```

此时才可以提交 Git。

---

# 十五、配置 hoteler-deployment GitRepository

创建：

```text
clusters/107.172.148.131/hoteler-source.yaml
```

内容：

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: hoteler-deployment
  namespace: flux-system
spec:
  interval: 1m

  url: ssh://git@github.com/damingerdai/hoteler-deployment.git

  ref:
    branch: master

  secretRef:
    name: hoteler-deployment-auth
```

这里我又遇到了一个小问题。

最开始写的是：

```yaml
ref:
  branch: main
```

Flux 报错：

```text
couldn't find remote ref "refs/heads/main"
```

原因非常简单：

```text
hoteler-deployment
```

默认分支实际是：

```text
master
```

修改之后：

```bash
flux get sources git
```

成功：

```text
NAME                 REVISION               READY

flux-system          main@sha1:0c63ecb5     True

hoteler-deployment   master@sha1:9fdb65cb   True
```

至此：

```text
SOPS
→ Secret
→ SSH Deploy Key
→ GitHub Private Repository
```

整条链路已经打通。

---

# 十六、让 Flux 部署 hoteler-api

`hoteler-deployment` 使用的是比较标准的 Kustomize 目录结构：

```text
hoteler-deployment/
└── hoteler-api/
    ├── base/
    │
    └── overlays/
        └── 107.172.148.131/
```

其中：

```text
base
```

保存通用 Kubernetes 配置。

而：

```text
overlays/107.172.148.131
```

保存当前服务器真正需要部署的配置。

因此 Flux 不应该直接指向：

```text
base
```

而应该指向最终 Overlay。

在 `fleet-infra` 中创建：

```text
clusters/107.172.148.131/hoteler-api.yaml
```

内容：

```yaml
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

  path: ./hoteler-api/overlays/107.172.148.131

  prune: true

  wait: true

  timeout: 5m
```

于是形成：

```text
hoteler-deployment
        │
        ▼
GitRepository
        │
        ▼
hoteler-api Kustomization
        │
        ▼
hoteler-api/overlays/107.172.148.131
        │
        ▼
Kubernetes
```

---

# 十七、从 Helm 迁移到 FluxCD

这台 K3s 上原本已经通过 Helm 安装过 Hoteler。

检查：

```bash
helm list -A
```

可以看到：

```text
NAME      NAMESPACE    CHART

hoteler   hoteler-app  hoteler-0.0.4
```

原来的 Pod：

```text
hoteler-app
├── hoteler-api
└── hoteler-web
```

由于新的 Flux Kustomization 将接管 Hoteler，因此不应该继续让：

```text
Helm
```

和：

```text
Flux + Kustomize
```

同时管理同一套 Kubernetes Resource。

否则会产生：

```text
Helm
  ↓
Deployment

Flux
  ↓
同一个 Deployment
```

两个控制来源。

因此卸载旧 Helm Release：

```bash
helm uninstall hoteler -n hoteler-app
```

然后：

```bash
flux reconcile kustomization hoteler-api
```

Flux 返回：

```text
✔ applied revision master@sha1:9fdb65cb...
```

再次检查：

```bash
helm list -A
```

Hoteler 已经消失，只剩 K3s 自带的 Traefik：

```text
traefik
traefik-crd
```

检查 Pod：

```bash
kubectl get pods -A
```

新的 Hoteler 已经由 Flux 创建：

```text
NAMESPACE           NAME
hoteler-namespace   hoteler-api-7c4b4dd596-m2cck
hoteler-namespace   hoteler-web-89ccd7c8c-wl4c5
```

并且：

```text
READY
1/1
1/1
```

至此迁移完成。

---

# 十八、最终 GitOps 架构

整个系统最终形成：

```text
                  GitHub
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
     fleet-infra        hoteler-deployment
          │                     │
          │                     │
 Flux Bootstrap          Kustomize
          │             base + overlays
          │                     │
          ├──── GitRepository ──┘
          │
          ▼
       FluxCD
          │
          ├── source-controller
          ├── kustomize-controller
          ├── helm-controller
          └── notification-controller
          │
          ▼
         K3s
          │
          ▼
   hoteler-namespace
          │
     ┌────┴────┐
     ▼         ▼
hoteler-api hoteler-web
```

Secret 的链路则是：

```text
SSH Private Key
       │
       ▼
Kubernetes Secret YAML
       │
       ▼
      SOPS
       │
       │ age public key
       ▼
Encrypted Secret
       │
       ▼
fleet-infra
       │
       ▼
     FluxCD
       │
       │ age private key
       ▼
     decrypt
       │
       ▼
Kubernetes Secret
       │
       ▼
GitHub SSH Deploy Key
       │
       ▼
hoteler-deployment
```

---

# 十九、最终目录结构

`fleet-infra` 大致变成：

```text
fleet-infra/
│
├── .sops.yaml
│
└── clusters/
    └── 107.172.148.131/
        │
        ├── kustomization.yaml
        │
        ├── hoteler-api.yaml
        ├── hoteler-source.yaml
        ├── hoteler-deployment-auth.secret.yaml
        │
        └── flux-system/
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
```

而：

```text
hoteler-deployment/
```

负责应用自己的 Kubernetes 配置：

```text
hoteler-deployment/
└── hoteler-api/
    ├── base/
    └── overlays/
        └── 107.172.148.131/
```

两个仓库的职责比较清晰：

```text
fleet-infra
→ 集群应该部署什么

hoteler-deployment
→ Hoteler 应该如何部署
```

---

# 二十、一些实践后的经验

这次迁移下来，有几个点值得记录。

### 1. Private Repository 不等于 Secret Store

即使：

```text
fleet-infra
```

本身就是 Private Repository，也不应该直接提交：

```text
SSH Private Key
API Key
Database Password
Token
```

Private Repository 解决的是仓库访问控制。

SOPS 解决的是：

```text
Secret at rest
```

两者不是同一个安全层次。

### 2. GitRepository 尽量使用 Deploy Key

对于：

```text
Flux → GitHub Private Repository
```

使用 SSH Deploy Key 的好处是权限天然限定在单个 Repository。

而且如果 Flux 只读：

```text
Allow write access = false
```

即可。

### 3. SOPS Bootstrap 一定要分阶段

这是这次最重要的经验：

```text
先让 Flux 会解密
        ↓
再给 Flux 加密文件
```

不要反过来。

### 4. Flux Kustomization 应该指向 Overlay

对于：

```text
base + overlays
```

结构：

```text
base
```

只是公共模板。

真正的环境入口应该是：

```text
overlays/<environment>
```

所以 Flux：

```yaml
path: ./hoteler-api/overlays/107.172.148.131
```

而不是：

```yaml
path: ./hoteler-api/base
```

### 5. 一个 Resource 最好只有一个管理者

原来的：

```text
Helm
```

和新的：

```text
Flux + Kustomize
```

不应该长期同时管理同一个应用。

迁移完成后，应该明确：

```text
Git
 ↓
Flux
 ↓
Kubernetes
```

是唯一部署路径。

---

# 总结

完成这次迁移之后，我的 Hoteler 部署已经从：

```text
人工执行 Helm
      ↓
Kubernetes
```

变成：

```text
GitHub
   ↓
FluxCD
   ↓
Kustomize
   ↓
Kubernetes
```

同时通过：

```text
SOPS + age
```

解决 GitOps 中 Secret 的存储问题，通过：

```text
SSH Deploy Key
```

解决 FluxCD 访问 GitHub Private Repository 的权限问题。

最终实现：

```text
Infrastructure as Code
+
GitOps
+
Encrypted Secrets
+
Declarative Deployment
```

以后对 Hoteler 部署配置的修改，不再需要直接登录服务器执行：

```bash
kubectl apply
```

或者：

```bash
helm upgrade
```

而是修改 Git：

```text
hoteler-deployment
       ↓
git push
       ↓
Flux detects change
       ↓
reconcile
       ↓
K3s
```

这也是这次迁移最大的意义：

> Kubernetes 集群不再依赖“我上次在服务器上执行过什么命令”，而是逐渐变成一个可以通过 Git 中的声明状态重新构建和恢复的系统。