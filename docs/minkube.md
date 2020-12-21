## 在Ubuntu 18.04.5 LTS上安装minkube

### 要求

1. 2 CPUs or more
2. 2GB内存
3. 20G空间
4. 无线网络连接
5. 容器或者虚拟机， 比如: Docker, Hyperkit, Hyper-V, KVM, Parallels, Podman, VirtualBox, or VMWare

### 下载

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

### 启动

```bash
sudo minikube start --registry-mirror=https://registry.docker-cn.com --vm-driver=none --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers
```

### 注意点

#### none driver integration tests: k8s 1.18 needs conntrack installed

解决方案
```
sudo apt-get install conntrack
## https://github.com/kubernetes/minikube/issues/7179
```


## 在Windows上安装k8s

### 安装Docker

安装[Docker Desktop](https://desktop.docker.com/win/stable/Docker%20Desktop%20Installer.exe)

### 配置docker的国内镜像

```json
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn",
    "http://hub-mirror.c.163.com"
  ],
  "insecure-registries": [],
  "debug": true,
  "experimental": false
}
```

### 使用阿里云的镜像

为了更快的完成一些安装，我们先通过一个阿里云的批处理，提前把Kubernetes需要的Images拉取下来

```bash
git clone https://github.com/AliyunContainerService/k8s-for-docker-desktop.git

cd k8s-for-docker-desktop

.\load_images.ps1
```

### 启动Kubernetes
在Docker仪表盘上在Settings切到Kubernetes上启动Enabled Kubernetes

### 安装Dashboard（可选）
使用[recommended.yaml](https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml)进行安装

```bash
$ kubectl apply -f recommended.yaml 
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

#### 启动

前台启动
```
$ kubectl proxy
```

后台启动
```
nohup kubectl proxy >/dev/null &
```

#### 登录

登录需要获取token

```
kubectl -n kube-system describe secret default| awk '$1=="token:"{print $2}'
```

这是我的返回结果
```
eyJhbGciOiJSUzI1NiIsImtpZCI6Imk0TWpNeGM3SWVrMHllMVphM0FPVFZIZ2RIaXZIbll2UzZObkJSZTZ5MUEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLXR2bWJ0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiZDI3YzljZS0wZWY2LTQ0YTAtYThmNC0xYTg2ZWMxN2JmNTQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.UjlNOPi95jsxtbGXVu6t3LK-1kOjlcLk7_qVPhDEmYD9so5BLnosS6Z_nBfpO2aU5xxMZMMvkTIydMKVTgftzeFpUZ7_ANsqjZ17Z2EnzUxhzkBU9USU3294APU4Gxep1yb4uyetRtIozdsd39-TlMwoCkHb4aGbluZiT64AkbDS6v7PhONaaCIKTT6hxvo4PEiyau_fEKCfI6rsWdcoOWlKLeXOwqGW1tHgIZEPR7Eln8NA52fAOvHyPp5DSKgD3L2qGDAlQNXCFCrB2bc7-xBEEBjeDXOhTIl1sUX6gmhEzp0XFH20JZaSJysvW1ZQGsv_AXj-4PX8Egv1kq1txA
```