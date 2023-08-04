---
title: 用k3s部署PostgreSQL用于开发
date: 2023-04-14 11:10:05
tags: [k8s, PostgreSQ]
categories: [后端]
---

## 前言

PostgreSQL是世界上最先进的开源数据库。
本文的目的是使用k3s本地部署PostgreSQL用于本地开发使用，不具备直接上生产的能力。


## 安装PostgreSQL

首先准备`config.yaml`用于定义PostgreSQL的配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: default
  labels:
    app: postgres
data:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: '123456'
  POSTGRES_DB: postgres
```

申请PersistentVolume用于数据持久化的存储：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-volume
  namespace: default
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

申请PersistentVolumeClaim用于数据持久化

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pv-claim
  namespace: default
  labels:
    type: local
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

数据库是一个有状态的服务，我认为StatefulSet比Pod更为合适，所以使用StatefulSet创建PostgreSQL实例：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: default
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15.0
          envFrom:
            - configMapRef:
                name: postgres-config
          ports:
            - containerPort: 5432
              name: postgres
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/PostgreSQL/data
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-pv-claim
```

定义一个Service让k3s里的其他pod可以访问数据库

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: default
  labels:
    app: postgres
spec:
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
  selector:
    app: postgres
```

现在我们可以执行kubectl apply命令去创建各种资源：

```bash
kubectl apply -f config.yaml
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f statefulset.yaml
```

> 请尽量按照顺序执行命令

查看一下效果:

```bash
kubectl get pods 

NAME         READY   STATUS    RESTARTS   AGE
postgres-0   1/1     Running   0          17h

----
kubectl get services

NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
postgres   ClusterIP   X.X.X.X   <none>        5432/TCP   17h
```

`X.X.X.X`是你k3s的集群内部网址，因人而异

## 配置PgBouncer

PostgreSQL是使用多进程的模式去创建连接，所以创建连接的成本比较高，因此推荐使用PgBouncer作为PostgreSQL的连接池用于提高数据库的并发性能。

将上文中的PostgreSQL代理到本地:

```bash
kubectl port-forward postgres-0 5432:5432
```

使用任意方式登录到PostgreSQL数据库，执行一下sql命令:

```sql
SELECT concat('"', usename, '" "', passwd, '"') FROM pg_shadow;
```

将获得的结果写入userlist.txt文件中

> 该步骤不可以省略，因为不同的数据库实例导出的passwd是不一样的！！！

准备pgbouncer.ini：

```ini
[databases]
postgres = host=postgres port=5432 dbname=postgres user=postgres password=123456

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 5432
admin_users = postgres
stats_users = postgres
auth_file = /etc/pgbouncer/userlist.txt
auth_type = scram-sha-256 
pool_mode = session
ignore_startup_parameters = extra_float_digits

# Log settings
# admin_users = postgres

# Connection sanity checks, timeouts
server_tls_sslmode = prefer

# TLS settings
client_tls_sslmode = disable
```

创建pgbouncer.yaml用于定义pgbouncer的配置信息

```bash
#!/bin/sh
KUBE_NAMESPACE="default"
cd `dirname $0`
for file in *.ini
do
  basename="$(basename $file)"
  echo $basname
  echo $file
  kubectl create secret generic "${basename%.*}"-config --namespace="$KUBE_NAMESPACE" --from-file="$file"  --from-file=userlist.txt -o yaml --dry-run=client | tee "${basename%.*}.yaml"
done
```

定义Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: default
  labels:
    app: pgbouncer
spec:
  revisionHistoryLimit: 10  # removes old replicasets for deployment rollbacks
  strategy:
    rollingUpdate:
      maxUnavailable: 0  # Avoid Terminating and ContainerCreating at the same time
  selector:
    matchLabels:
      app: pgbouncer
  template:
    metadata:
      labels:
        app: pgbouncer
    spec:
      containers:
        - name: pgbouncer
          image: rmccaffrey/pgbouncer:1.18.1
          #imagePullPolicy: Always
          resources:
            requests:
              memory: "100Mi"
              cpu: "0.5"
            limits:
              memory: "200Mi"
              cpu: "1"
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: configfiles
              mountPath: "/etc/pgbouncer"
              readOnly: true  # writes update the secret!
          livenessProbe:
            tcpSocket:
              port: 5432
            periodSeconds: 60
          lifecycle:
            preStop:
              exec:
                # Allow existing queries clients to complete within 120 seconds
                command: ["/bin/sh", "-c", "killall -INT pgbouncer && sleep 120"]
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ['all']
      volumes:
        - name: configfiles
          secret:
            secretName: pgbouncer-config
```

定义Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pgbouncer
  namespace: default
  labels:
    app: pgbouncer
spec:
  type: ClusterIP
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
      name: postgres
  selector:
    app: pgbouncer
```

创建资源：

```bash
kubectl apply -f pgbouncer.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

查看效果：

```bash
kubectl get services

---
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
postgres    ClusterIP   X.X.X.X        <none>        5432/TCP   17h
pgbouncer   ClusterIP   X.X.X.X        <none>        5432/TCP   50s
```

使用port-forward代理到本地进行测试pgbouncer是否能够正常使用：

```bash
kubectl port-forward service/pgbouncer 5432:5432
```

## 总结

本文实现了使用k3s部署PostgreSQL和PgBouncer，可以用于一般的本地开发坏境使用，也在docker destkop上测试通过，但是没有在minikube上测试过。

以上资源可以在[health-master-deployments-db](https://github.com/damingerdai/health-master/tree/main/deplyoments/db)和[health-master-deployments-pgbouncer](https://github.com/damingerdai/health-master/tree/main/deplyoments/pgbouncer)上获取。