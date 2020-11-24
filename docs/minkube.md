# Minkube 安装笔记

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
