---
title: 使用 Traefik、cert-manager 和 DNS-01 为内网 Kubernetes 服务配置 HTTPS
date: 2026-08-28 20:13:15
tags: [Kubernetes, K3s, Traefik, Let's Encrypt, FluxCD, Kustomize]
categories: [软件]

---

# 使用 Traefik、cert-manager 和 DNS-01 为内网 Kubernetes 服务配置 HTTPS

最近我给部署在内网 K3s 集群中的 Web 应用配置了一个浏览器可信的 HTTPS 入口。最终采用的方案是：

> 自有域名的内部子域名 + 本地 DNS 解析 + Traefik Ingress + cert-manager + Let's Encrypt DNS-01

这套方案最大的优点是，服务本身不需要暴露到公网。Let's Encrypt 只通过公网 DNS 的 TXT 记录验证域名所有权，签发完成后，用户仍然通过局域网 IP 访问服务。

本文记录完整的配置过程，并使用以下示例信息：

```text
公网域名：example.com
内部域名：health-master-dev.internal.example.com
K3s 节点：192.168.31.215
应用命名空间：health-master-dev-namespace
Ingress Controller：Traefik
DNS 服务商：Cloudflare
```

## 一、整体原理

访问服务和签发证书走的是两条不同的链路。

本地访问链路：

```text
本地 hosts 或内网 DNS
        ↓
health-master-dev.internal.example.com
        ↓
192.168.31.215
        ↓
Traefik Ingress
        ↓
Kubernetes Service
        ↓
Next.js Pod
```

证书签发链路：

```text
cert-manager
        ↓ Cloudflare API
创建 _acme-challenge TXT 记录
        ↓
Let's Encrypt 查询公网 DNS
        ↓
验证域名所有权并签发证书
        ↓
生成 kubernetes.io/tls Secret
        ↓
Traefik 加载证书
```

DNS-01 验证不需要 Let's Encrypt 访问 `192.168.31.215`，也不要求 Traefik 对公网开放。只要可以通过 Cloudflare 公网 DNS 创建和查询 TXT 记录，就能完成验证。

## 二、为什么不用 HTTP-01

HTTP-01 会要求 Let's Encrypt 访问类似下面的公网地址：

```text
http://health-master-dev.internal.example.com/.well-known/acme-challenge/...
```

但内网 K3s 服务没有公网入口，公网无法访问 `192.168.31.215`，因此 HTTP-01 不适合这个场景。

DNS-01 只验证 TXT 记录：

```text
_acme-challenge.health-master-dev.internal.example.com
```

所以它特别适合：

- 内网服务
- 家庭实验室和 Homelab
- 没有公网 IP 的集群
- 需要通配符证书的场景

## 三、安装 cert-manager

使用 Helm 安装 cert-manager：

```bash
helm install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.1 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

如果提示：

```text
cannot reuse a name that is still in use
```

说明名为 `cert-manager` 的 Helm Release 已经存在，先检查状态：

```bash
helm list -n cert-manager
helm status cert-manager -n cert-manager
kubectl get pods -n cert-manager
```

需要升级时使用：

```bash
helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.1 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

正常情况下会看到：

```text
cert-manager              Running
cert-manager-cainjector   Running
cert-manager-webhook      Running
```

确认 CRD：

```bash
kubectl get crd certificates.cert-manager.io
```

## 四、创建 Cloudflare API Token

在 Cloudflare 创建一个仅能操作目标 Zone 的 API Token，最小权限为：

| 类型 | 权限 |
| --- | --- |
| Zone | DNS → Edit |
| Zone | Zone → Read |
| Zone Resources | Specific zone → `example.com` |

`DNS:Read` 不够，因为 cert-manager 需要临时创建和删除 `_acme-challenge` TXT 记录。

不要使用权限覆盖整个账号的 Global API Key。使用限定到单个 Zone 的 API Token，影响范围更小。

## 五、创建 Cloudflare Token Secret

`ClusterIssuer` 使用的 Cloudflare Secret 应放在 cert-manager 的资源命名空间，默认是 `cert-manager`。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "REPLACE_WITH_CLOUDFLARE_API_TOKEN"
```

将其保存为：

```text
infrastructure/cert-manager/cloudflare-api-token-secret.yaml
```

生产环境更推荐使用 SOPS、Sealed Secrets 或 External Secrets 加密管理。即使是内网 Git 仓库，也需要注意仓库备份、克隆副本和历史提交都可能长期保留明文 Token。

## 六、同时配置 staging 和 production Issuer

建议保留两个 `ClusterIssuer`：

- `letsencrypt-staging`：用于验证配置，证书不受浏览器信任，但签发限制更宽松。
- `letsencrypt-production`：用于签发浏览器信任的正式证书。

### Staging Issuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: "admin@example.com"
    server: https://acme-staging-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-staging-account-key

    solvers:
      - selector:
          dnsZones:
            - example.com

        dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token-secret
              key: api-token
```

### Production Issuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: "admin@example.com"
    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-production-account-key

    solvers:
      - selector:
          dnsZones:
            - example.com

        dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token-secret
              key: api-token
```

两者必须使用不同的名称、ACME Server 和账户私钥 Secret。

基础设施目录可以组织为：

```text
infrastructure/cert-manager/
├── cloudflare-api-token-secret.yaml
├── cluster-issuer-staging.yaml
├── cluster-issuer-production.yaml
└── kustomization.yaml
```

`kustomization.yaml`：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - cloudflare-api-token-secret.yaml
  - cluster-issuer-staging.yaml
  - cluster-issuer-production.yaml
```

检查 Issuer：

```bash
kubectl get clusterissuer
```

目标状态：

```text
NAME                     READY
letsencrypt-production   True
letsencrypt-staging      True
```

## 七、为应用声明 Certificate

Ingress、Certificate 和具体域名都与部署环境相关，因此我把它们放在 Kustomize overlay，而不是通用 base。

目录示例：

```text
app/health-master/
├── base/
│   ├── deployment.yaml
│   ├── web-deployment.yaml
│   ├── service.yaml
│   ├── web-service.yaml
│   ├── hpa.yaml
│   └── kustomization.yaml
└── overlays/
    └── 192.168.31.215/
        ├── certificate.yaml
        ├── ingress.yaml
        ├── redirect-https-middleware.yaml
        └── kustomization.yaml
```

先用 staging Issuer 测试：

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: health-master-web
spec:
  secretName: health-master-web-tls

  dnsNames:
    - health-master-dev.internal.example.com

  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
```

cert-manager 签发成功后，会在相同命名空间生成：

```text
Secret/health-master-web-tls
```

Secret 类型为：

```text
kubernetes.io/tls
```

## 八、配置 Web Service

Next.js 默认监听 3000 端口，Service 可以统一对集群提供名为 `http` 的端口：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: health-master-web
  labels:
    app: health-master-web
spec:
  type: ClusterIP

  selector:
    app: health-master-web

  ports:
    - name: http
      port: 3000
      targetPort: http
      protocol: TCP
```

Deployment 中必须存在对应的 Pod 标签和命名端口：

```yaml
spec:
  template:
    metadata:
      labels:
        app: health-master-web
    spec:
      containers:
        - name: health-master-web
          ports:
            - name: http
              containerPort: 3000
```

Ingress 通过端口名称引用 Service：

```yaml
port:
  name: http
```

## 九、配置 Traefik HTTPS Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: health-master-web-https
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik

  rules:
    - host: health-master-dev.internal.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: health-master-web
                port:
                  name: http

  tls:
    - hosts:
        - health-master-dev.internal.example.com
      secretName: health-master-web-tls
```

这里的 `secretName` 必须和 `Certificate.spec.secretName` 完全一致，并且 Certificate、TLS Secret 和 Ingress 必须位于同一个 namespace。

## 十、配置 HTTP 自动跳转 HTTPS

如果 Ingress 只绑定 `websecure`：

```yaml
traefik.ingress.kubernetes.io/router.entrypoints: websecure
```

访问 HTTP 时会得到 Traefik 的 404，因为没有路由监听 `web` entrypoint。

先创建 Middleware：

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

再创建单独的 HTTP Ingress：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: health-master-web-http
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: health-master-dev-namespace-redirect-https@kubernetescrd
spec:
  ingressClassName: traefik

  rules:
    - host: health-master-dev.internal.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: health-master-web
                port:
                  name: http
```

Middleware 引用格式是：

```text
<namespace>-<middleware-name>@kubernetescrd
```

测试：

```bash
curl -I http://health-master-dev.internal.example.com/
```

正常响应可能是：

```text
HTTP/1.1 308 Permanent Redirect
Location: https://health-master-dev.internal.example.com/
```

`308` 不是错误。和 `301` 相比，它会在跳转时保留原请求方法和请求体，更适合可能包含 POST、API 或 Server Action 请求的 Web 应用。

## 十一、加入 Kustomize overlay

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: health-master-dev-namespace

resources:
  - ../../base
  - certificate.yaml
  - ingress.yaml
  - redirect-https-middleware.yaml
```

需要注意：Cloudflare Token Secret 不应该放在这个 overlay 中，因为这里统一指定了应用 namespace。它应由独立的 cert-manager 基础设施目录管理，并明确位于 `cert-manager` namespace。

## 十二、通过 FluxCD 发布

GitRepository 示例：

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: application-gitops
  namespace: flux-system
spec:
  interval: 5m
  url: ssh://git@192.168.31.220:1022/example/application-gitops.git

  ref:
    branch: main

  secretRef:
    name: application-gitops-auth
```

Kustomization 示例：

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: application
  namespace: flux-system
spec:
  interval: 1h
  retryInterval: 1m
  timeout: 5m

  sourceRef:
    kind: GitRepository
    name: application-gitops

  path: ./clusters/192.168.31.215
  prune: true
  wait: true
```

这几个时间参数分别表示：

| 配置 | 含义 |
| --- | --- |
| `GitRepository.interval: 5m` | 最长约每 5 分钟检查一次 Git 新提交 |
| `Kustomization.interval: 1h` | 每小时进行周期性协调和漂移修正 |
| `retryInterval: 1m` | 协调失败后每分钟重试 |
| `timeout: 5m` | 单次协调等待资源健康的最长时间 |

GitRepository 发现新 revision 后会触发 Kustomization，不需要等待一个完整的 Kustomization interval，所以两者不是简单相加。

调试时可以立即触发：

```bash
flux reconcile kustomization application \
  -n flux-system \
  --with-source
```

## 十三、检查证书申请过程

查看 Certificate：

```bash
kubectl get certificate \
  -n health-master-dev-namespace
```

查看完整 ACME 资源链：

```bash
kubectl get certificate,certificaterequest,order,challenge \
  -n health-master-dev-namespace
```

查看详细错误：

```bash
kubectl describe certificate health-master-web \
  -n health-master-dev-namespace

kubectl describe challenge \
  -n health-master-dev-namespace
```

确认 TLS Secret：

```bash
kubectl get secret health-master-web-tls \
  -n health-master-dev-namespace
```

正常状态类似：

```text
NAME                READY   SECRET                  AGE
health-master-web   True    health-master-web-tls   5m
```

## 十四、检查 Traefik 实际返回的证书

即使 Certificate 显示 `Ready=True`，也建议检查实际通过 443 端口返回的证书：

```bash
openssl s_client \
  -connect health-master-dev.internal.example.com:443 \
  -servername health-master-dev.internal.example.com \
  </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -dates
```

如果 issuer 中出现：

```text
(STAGING)
```

说明 Traefik 已正确加载 staging 证书，只是该证书不受浏览器信任。

如果显示：

```text
TRAEFIK DEFAULT CERT
```

则通常表示：

- TLS Secret 不存在；
- Ingress 和 Secret 不在同一个 namespace；
- `Ingress.spec.tls.secretName` 写错；
- Traefik 尚未加载最新配置。

## 十五、从 staging 切换到 production

staging 验证成功后，只需要修改 Certificate：

```yaml
issuerRef:
  name: letsencrypt-production
  kind: ClusterIssuer
```

`secretName` 可以保持不变：

```yaml
secretName: health-master-web-tls
```

cert-manager 会重新执行 DNS-01，并用生产证书更新 Secret。Traefik 会自动重新加载证书，Ingress 无需修改。

切换后再次检查：

```bash
openssl s_client \
  -connect health-master-dev.internal.example.com:443 \
  -servername health-master-dev.internal.example.com \
  </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -dates
```

只要 issuer 不再包含 `(STAGING)`，就说明已经切换到正式证书。

## 十六、配置本地解析

在 macOS 的 `/etc/hosts` 中添加：

```text
192.168.31.215 health-master-dev.internal.example.com
```

刷新 DNS 缓存：

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

验证 `/etc/hosts`：

```bash
dscacheutil -q host \
  -a name health-master-dev.internal.example.com
```

不建议只使用 `dig` 验证，因为 `dig` 通常直接查询 DNS Server，不一定经过系统的 `/etc/hosts` 解析流程。

最终测试：

```bash
curl -IL http://health-master-dev.internal.example.com/
```

应该先看到 HTTP 到 HTTPS 的永久跳转，再看到 HTTPS 响应。

## 十七、几个容易踩到的坑

### 1. 拥有的域名必须和证书域名一致

如果只拥有 `example.com`，就只能为 `example.com` 及其子域名签发证书，不能凭此为另一个未持有的域名签发。

推荐内部域名采用：

```text
<service>.<environment>.internal.example.com
```

### 2. 内网 DNS 和公网 DNS 可以分开

公网 Cloudflare DNS 负责 ACME TXT 验证；内网 DNS 或 `/etc/hosts` 负责把服务域名解析到私有 IP。公网不需要创建指向 `192.168.x.x` 的 A 记录。

### 3. staging 证书必然不受浏览器信任

出现自签名或不可信错误，不一定表示配置失败。先通过 `openssl` 检查 issuer；如果包含 `(STAGING)`，说明测试流程已经成功，应切换到 production。

### 4. Ingress 只监听 websecure 时，HTTP 会返回 404

需要额外创建监听 `web` 的 Ingress，并通过 Traefik Middleware 跳转到 HTTPS。

### 5. 308 是正常的 HTTPS 永久跳转

308 会保留原始 HTTP 方法和请求体，比可能将 POST 变为 GET 的 301 更适合现代 Web 应用。

### 6. Certificate、TLS Secret 和 Ingress 必须在同一个 namespace

否则 Certificate 可能已经签发成功，但 Traefik 仍返回默认证书。

### 7. Cloudflare Token 需要 DNS:Edit

`DNS:Read` 无法让 cert-manager 创建 TXT Challenge，必须使用 `DNS:Edit`，并把 Token 范围限制到目标 Zone。

## 十八、最终效果

完成配置后，实现了：

- 服务只在局域网中开放；
- 使用自己持有域名的内部子域名访问；
- HTTPS 证书由 Let's Encrypt 正式签发并被浏览器信任；
- cert-manager 自动签发和续期；
- Cloudflare DNS-01 自动完成域名验证；
- HTTP 自动以 308 跳转到 HTTPS；
- Ingress、Certificate 和环境域名由 Kustomize overlay 管理；
- FluxCD 自动将 Git 中的声明同步到 K3s 集群。

对于 Homelab、内部管理平台和不希望暴露到公网的服务，这是一套兼顾安全性、自动化和使用体验的 HTTPS 方案。

## 参考资料

- [cert-manager：ACME 配置](https://cert-manager.io/docs/configuration/acme/)
- [cert-manager：Cloudflare DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/)
- [Let's Encrypt：Challenge Types](https://letsencrypt.org/docs/challenge-types/)
- [Traefik：Kubernetes Ingress](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [FluxCD：Kustomization](https://fluxcd.io/flux/components/kustomize/kustomizations/)

