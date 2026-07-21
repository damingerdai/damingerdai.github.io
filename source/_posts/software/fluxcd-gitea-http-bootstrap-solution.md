---
title: 内网 GitOps 实践：FluxCD 与 Gitea (HTTP) 集成与 初始化 指南
date: 2026-07-21 13:43:22
tags: [Kubernetes, GitOps, FluxCD, Gitea, DevOps]
categories: [软件]
summary: 详细记录 FluxCD 在内网纯 HTTP 环境下集成 Gitea 的踩坑与解决全过程，重点解决协议校验与 Go-Git 凭证传输拦截问题。
---

FluxCD 在内网纯 HTTP 环境下与 Gitea 集成并完成初始化（Bootstrap）过程的技术总结

---

## 1. 环境背景与技术痛点

* **基础设施**：部署于内网的 Gitea 实例（通过 HTTP 协议暴露服务，地址格式为 `http://<IP>:<PORT>`）。
* **网络限制**：内网环境未配置 TLS/SSL 证书，不支持 HTTPS 通信。
* **核心拦截点**：
1. **协议强校验**：新版 Flux CLI 的 `flux bootstrap gitea` 在调用 Gitea API 时默认强制使用 HTTPS，抛出 `http: server gave HTTP response to HTTPS client` 错误。
2. **Go-Git 明文传输保护**：Flux 底层依赖的 Go-Git 库出于安全限制，禁止在未加密的明文 HTTP 链接中直接发送 Basic Auth 凭证（Token/密码），抛出 `basic auth cannot be sent over HTTP` 错误。



---

## 2. 核心解决方案（参考 Discussion #4972）

根据 FluxCD 社区讨论 [FluxCD Discussion #4972](https://github.com/fluxcd/flux2/discussions/4972)，最稳定且通用的解法是**跳过特定 Git 供应商（Gitea）的 API 专有引导逻辑，改用通用 `flux bootstrap git` 命令，并显式透传 `--allow-insecure-http=true` 参数**。

### 关键机制说明：

* **`flux bootstrap git`**：避开了针对 Gitea API 的 HTTPS 强制探活逻辑（如 `GetOrgByName` 等组织/个人账号鉴权 API）。
* **`--allow-insecure-http=true`**：向底层 Go-Git 显式下发参数，解除对纯 HTTP 链接发送鉴权凭证的限制。

---

## 3. 标准操作流程

### 步骤一：预先准备 Gitea 仓库与 Token

1. **服务部署**：Gitea 的容器化部署结构可参考 [home-labs-docker/gitea](https://github.com/damingerdai/home-labs-docker/tree/master/gitea)。
2. **凭证生成**：在 Gitea 用户设置中生成个人访问令牌（PAT），需包含仓库读写权限（`write:repository`）。
3. **创建仓库**：登录 Gitea 页面，手动创建一个空的私有 Git 仓库，例如 `fleet-infra`。

### 步骤二：执行 Bootstrap 引导命令

在终端中执行通用 Git 初始化命令（使用 PAT 作为 `--password`）：

```bash
flux bootstrap git \
  --url="http://<Gitea-IP>:<Port>/<Username>/fleet-infra.git" \
  --username="<Username>" \
  --password="<Gitea_PAT_Token>" \
  --allow-insecure-http=true \
  --token-auth=true \
  --path="./clusters/<Cluster-Name-or-IP>"

```

### 步骤三：验证安装结果

命令执行完成后，检查系统与组件状态：

1. **检查控制器状态**：
```bash
flux check

```


确认 `source-controller`、`kustomize-controller`、`helm-controller` 和 `notification-controller` 全部就绪。
2. **检查 Git 源同步状态**：
```bash
flux get sources git

```


确认 `flux-system` 源的 `READY` 状态为 `True`，且已存储最新的 Commit Hash。

```bash
➜  ~ flux get sources git
NAME       	REVISION          	SUSPENDED	READY	MESSAGE
flux-system	main@sha1:d1968847	False    	True 	stored artifact for revision 'main@sha1:d1968847'
```

---

## 4. GitOps 工作流闭环验证

初始化完成后，无需直接在本地运行 `kubectl apply`：

1. **配置提交**：将应用的 Kubernetes Manifests（如 `pod.yaml`）提交并 `git push` 至 Gitea 的 `fleet-infra` 仓库对应集群路径（`./clusters/<Cluster-Name-or-IP>/`）下。

```bash
cat <<EOF > ./clusters/192.168.31.215/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitops-pod-test
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
EOF
```
2. **自动调和**：集群内部的 Flux `source-controller` 与 `kustomize-controller` 会定期轮询 Gitea 仓库，自动检测版本差异并将其应用至集群，实现声明式部署与状态漂移修复。
