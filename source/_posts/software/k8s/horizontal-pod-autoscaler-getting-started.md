---
title: HorizontalPodAutoscaler入门实践
date: 2023-09-27 19:58:21
tags: [Docker Desktop, docker, kubernetes, k8s]
categories: [软件]
---

# HorizontalPodAutoscaler

在Kubernetes中，HorizontalPodAutoscaler 自动更新工作负载资源（例如Deployment或者StatefulSet），目的是自动扩缩工作负载以满足需求。水平扩缩意味着对增加的负载的响应是部署更多的 Pod。

本文目的是通过Docker Desktop上的Kubernetes实例去实践pod的水平扩展。

本文默认Docker Desktop上的Kubernetes已经安装完成。如果需要帮助，可以阅读[Docker Desktop自带k8s安装笔记](https://damingerdai.github.io/2021/01/14/software/k8s/how-to-install-docker-desktop-k8s/)。

# 安装 Metrics server

由于Docker Desktop上的Kubernetes默认并没有安装Metrics server，而HorizontalPodAutoscaler依赖通过Metrics server获取到的数据， 因此需要提前安装。

```bash
kubectl top node 
error: Metrics API not available
```

从[Metrics server的release页面](https://github.com/kubernetes-sigs/metrics-server/releases)获取最新的`components.yaml`文件，

然后执行：

```bash
kubectl apply -f components.yaml
```

镜像的拉取和运行需要一点时间。我们可以通过以下命令查看运行情况：

```bash
kubectl  get pod -n kube-system | grep metrics

# 结果
metrics-server-75f45b4dd4-fbxd2          1/1     Running   0               3h25m
```

查看node资源情况：

```bash
kubectl top node

# 结果
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
docker-desktop   1901m        47%    6250Mi          80% 
```

查看pod资源情况

```bash
kubectl top pods -A

# 结果
NAMESPACE           NAME                                     CPU(cores)   MEMORY(bytes)             
kube-system         coredns-5d78c9869d-dq2s7                 4m           16Mi            
kube-system         coredns-5d78c9869d-g5xhx                 2m           21Mi            
kube-system         etcd-docker-desktop                      20m          72Mi            
kube-system         kube-apiserver-docker-desktop            32m          271Mi           
kube-system         kube-controller-manager-docker-desktop   11m          64Mi            
kube-system         kube-proxy-qdc55                         1m           25Mi            
kube-system         kube-scheduler-docker-desktop            5m           32Mi            
kube-system         metrics-server-75f45b4dd4-fbxd2          6m           23Mi            
kube-system         storage-provisioner                      4m           15Mi            
kube-system         vpnkit-controller                        1m           14Mi  
```

# 配置HorizontalPodAutoscaler

这里以我的个人开源项目[hoteler](https://github.com/damingerdai/hoteler)为例, 将下面的内容写入`hpa.yaml`文件中：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hoteler-api-hpa
  namespace: hoteler-namespace
  labels:
    app: hoteler-api-hpa
spec:
  maxReplicas: 5
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hoteler-api
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 50
```
`minReplicas: 1`表示最小一个pod实例，`maxReplicas: 5`则说明最多扩展到5个pod实例。

`metrics`下面定义两个·Resource·， 第一个Resource规定当cpu资源占用率超过50%就会进行扩容，第二个Resource则规定了当内存使用率超过50%之后才会行将扩容。 这两个Resource是独立计算所需要的副本数量，取最大值。


# 参考

1. [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
2. [k8s的HPA安装测试](https://srerun.com/article/2021/5/6/39.html)
3. [安装 Metrics server](https://juejin.cn/post/7104907264779583501)
4. [Enable Kubernetes Metrics Server on Docker Desktop](https://dev.to/docker/enable-kubernetes-metrics-server-on-docker-desktop-5434)
5. [Kubernetes HPA 使用详解](https://www.qikqiak.com/post/k8s-hpa-usage/)
6. [How kubernetes HPA with 2 or more metrics behaves - especially the no.of replicas calculation?](https://stackoverflow.com/questions/54302592/how-kubernetes-hpa-with-2-or-more-metrics-behaves-especially-the-no-of-replica)
7. [feat: 使用k8s的HorizontalPodAutoscaler进行水平的资源缩放](https://github.com/damingerdai/hoteler/pull/749)