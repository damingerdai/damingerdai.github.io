---
title: 从零开始：在 K3s 中部署 CloudNativePG 并成功实现外部直连（全流程实战）
date: 2026-06-10 15:27:59
tags: [kubernetes, k3s, CloudnativePg]
categories: [软件]
---

# 前言

本文是在我的Homelab 单节点 K3s（IP: 192.168.0.103）上部署 CloudNativePG (CNPG) 1.29 版本。

```bash
kubectl get nodes
NAME     STATUS   ROLES           AGE   VERSION
earzer   Ready    control-plane   53d   v1.35.4+k3s1
```

# 过程

## 安装 kubectl-cnpg 插件（强烈推荐）

CloudNativePG 提供了一个非常强大的 kubectl 插件，可以帮助你轻松查看数据库状态、进行主备切换、查看日志等。

在你的控制端（或 K3s 节点上）运行以下命令安装：

```bash
curl -sSfL https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-cnpg-plugin.sh | sudo sh -s -- -b /usr/local/bin
```

如果你是中国用户，你可以使用gh-proxy来加速

```bash
curl -sSfL https://gh-proxy.com/https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-cnpg-plugin.sh | sudo sh -s -- -b /usr/local/bin
```

验证：

```bash
kubectl cnpg version
Build: {Version:1.29.1 Commit:a4060c152 Date:2026-05-08}
```

## 部署 CNPG Operator

使用helm来部署CNPG Operator：

```bash
# 添加并更新仓库
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

# 安装 1.29 对应的 Operator
helm install cnpg-operator cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace
```

## 部署你的 PostgreSQL 实例

准备 postgres-cluster.yaml：

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres
  namespace: postgres
spec:
  # 实例数量：单节点 Homelab 设为 1
  instances: 1

  # 官方默认 PostgreSQL 16 镜像地址
  imageName: ghcr.io/cloudnative-pg/postgresql:18.4
  imagePullPolicy: IfNotPresent

  # 存储配置
  storage:
    size: 10Gi
    storageClass: local-path # 完美匹配 K3s 默认本地存储卷驱动(多节点推荐Longhorn)

  postgresql:
    pg_hba:
      - host all all 0.0.0.0/0 scram-sha-256
```

执行部署：

```bash
# 创建命名空间（如果之前没建过）
kubectl create namespace postgres

# 部署数据库
kubectl apply -f postgres-cluster.yaml
```

数据库已经稳稳地跑起来了，CloudNativePG 已经非常贴心地自动在后台为你创建好了完整的 Kubernetes Service（服务发现） 矩阵。

我们可以用一记命令把它们全部抓出来：

```bash
kubectl get svc -n postgres
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
postgres-r    ClusterIP      10.43.91.157   <none>          5432/TCP         3h50m
postgres-ro   ClusterIP      10.43.95.111   <none>          5432/TCP         3h50m
postgres-rw   ClusterIP      10.43.29.10    <none>          5432/TCP         3h50m
```

## 访问Postgres数据库

K3s 默认内置了一个极其轻量级的负载均衡器（Klipper LB）。它不需要你安装像 MetalLB 这样复杂的组件，就能直接把集群内的 5432 端口原封不动地映射到宿主机的 5432 端口上。

在postgres-cluster.yaml上追加

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-lb # 换个名字，避免和 Operator 自动生成的默认 svc 冲突
  namespace: postgres
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 5432 # 局域网访问的端口
      targetPort: 5432 # 容器内部端口
  selector:
    # 🎯 精准命中 CloudNativePG 自动生成的 Primary（读写主节点）标签
    cnpg.io/cluster: postgres
    cnpg.io/instanceRole: primary
```

保存退出后，直接执行这行命令。别担心，这不会导致你的数据库重启，Kubernetes 只会在线更新 postgres-rw 的服务类型：

```bash
kubectl apply -f postgres-cluster.yaml
```

应用成功后，挂上监控看一下成果：

```bash
 kubectl get svc -n postgres
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
postgres-lb   LoadBalancer   10.43.32.3     192.168.0.103   5432:30900/TCP   3h47m
postgres-r    ClusterIP      10.43.91.157   <none>          5432/TCP         3h54m
postgres-ro   ClusterIP      10.43.95.111   <none>          5432/TCP         3h54m
postgres-rw   ClusterIP      10.43.29.10    <none>          5432/TCP         3h54m
```

你会看到 postgres-lb 的  type是 LoadBalancer，并且它的 EXTERNAL-IP 在两秒钟内就会被 Klipper 自动刷成你的宿主机局域网 IP（192.168.0.103）。

CloudNativePG 会默认把密码存放在一个名为 postgres-app 的 Secret 对象里。请直接在终端执行这行命令，它会帮你解密并显示在屏幕上：

```bash
kubectl get secret postgres-app -n postgres -o jsonpath="{.data.password}" | base64 --decode
```

如果想要临时测试修改密码可以使用：

```bash
kubectl exec -it postgres-1 -n postgres -c postgres -- psql -U postgres -d app -c "ALTER USER app WITH PASSWORD 'password123';"
```

当然最好是直接修改或创建一个对应的 Kubernetes Secret，然后让 CNPG 自动滚动更新

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-user-password
  namespace: postgres
type: kubernetes.io/basic-auth
stringData:
  username: app
  password: password123
```

然后在 Cluster 的 spec.managed.roles 中引用它：

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres
  namespace: postgres
spec:
  instances: 1
  imageName: ghcr.io/cloudnative-pg/postgresql:16.4
  imagePullPolicy: IfNotPresent

  # 📥 这里是声明式管理用户的核心配置
  managed:
    roles:
      - name: app              # 数据库中的用户名
        ensure: present        # 确保该用户存在
        login: true            # 允许该用户登录
        # 🔐 引用上面创建的密码 Secret
        passwordSecret:
          name: my-app-user-secret

  storage:
    size: 10Gi
    storageClass: local-path

  postgresql:
    pg_hba:
      - host all all 0.0.0.0/0 scram-sha-256
```

这里以go为例：

```golang
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/jackc/pgx/v5"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	connStr := "postgres://app:12345@192.168.0.103:5432/app?sslmode=require"

	conn, err := pgx.Connect(ctx, connStr)
	if err != nil {
		log.Fatalf("connect failed: %v", err)
	}
	defer conn.Close(ctx)

	var now time.Time
	err = conn.QueryRow(ctx, "select now()").Scan(&now)
	if err != nil {
		log.Fatalf("query failed: %v", err)
	}

	fmt.Println("OK:", now)
}
```

## 镜像加速

中国用户可以使用daocloud进行镜像加速。

在你的 K3s 服务器 earzer 上，编辑或创建 /etc/rancher/k3s/registries.yaml 文件：

```bash
sudo mkdir -p /etc/rancher/k3s
sudo vim /etc/rancher/k3s/registries.yaml
```

把下面这段配置贴进去：

```yaml
mirrors:
  "ghcr.io":
    endpoint:
      - "https://ghcr.m.daocloud.io"
```

保存退出。然后重启 K3s 控制平面以加载配置，此操作不会影响已在运行的容器：

```bash
sudo systemctl restart k3s
```

## 问题

如果使用jdbc访问就会出现：

```bash
026-06-10T21:57:02.099+08:00  INFO 16952 --- [demo] [nio-8080-exec-1] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2026-06-10T21:57:08.172+08:00 ERROR 16952 --- [demo] [nio-8080-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed: org.springframework.jdbc.CannotGetJdbcConnectionException: Failed to obtain JDBC Connection] with root cause

java.net.SocketTimeoutException: Read timed out
...
```

目前尚在研究中，有遇到类似问题的小伙伴可以告知一下为什么呢



# 引用

- [CloudNativePG Documentation](https://cloudnative-pg.io/docs/1.29/installation_upgrade)
- [CloudNativePG Chart documentation f](https://github.com/cloudnative-pg/charts/blob/main/charts/cloudnative-pg/README.md)
- [CloudNativePG Kubectl Plugin](https://cloudnative-pg.io/docs/devel/kubectl-plugin/)
- [Public Image Mirror](https://github.com/DaoCloud/public-image-mirror)