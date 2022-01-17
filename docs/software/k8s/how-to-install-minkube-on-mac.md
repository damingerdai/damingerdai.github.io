# Mac版minikube安装笔记

[minikube](https://minikube.sigs.k8s.io/docs/start/)是一个专注于让Kubernetes更加容易学习和开发的本地Kubernetes。
只需要[Docker](https://www.docker.com/)或者虚拟机环境，我们便可以通过`minikube start`就能快速启动Kubernetes。

## 要求

安装minikube需要以下要求：

1. 至少两个CPU
2. 至少2GB内容
3. 至少20GB的硬盘空间
4. 容器或者虚拟机， 比如[Docker](https://www.docker.com/, [Hyperkit](https://github.com/moby/hyperkit), [Hyper-V](https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/hyper-v-technology-overview), [KVM](https://www.linux-kvm.org/page/Main_Page), [Parallels](https://www.parallels.com/), [Podman](https://podman.io/), [VirtualBox](https://www.virtualbox.org/), or [VMware Fusion/Workstation](https://www.vmware.com/asean/products/fusion.html)

## 安装

针对macos的x86-64的平台的，可以使用homebrew进行安装：

```
brew install minikube
```

如果倾向于二进制文件安装，可以使用下面的命令：

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

如果是其他平台的操作系统，请看[minikube的官网教程](https://minikube.sigs.k8s.io/docs/start/)

## 启动

简单来说，可以直接使用`minikube start`可以直接启动了，但是在国内用于特殊的网络限制，请使用如下命令：

```
minikube start --registry-mirror=https://registry.docker-cn.com  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers
```

### 驱动（Drivers）

通过驱动，minikube可以支持不同的容器和虚拟机。

针对macos来说，推荐使用docker和hyperkit，我们只需要在上面的命令的基础上添加`--vm-driver=docker`或者`--vm-driver=hyperkit`。

其他平台可以访问[Drivers](https://minikube.sigs.k8s.io/docs/drivers/)。

## kubectl

[kubectl](https://kubernetes.io/docs/tasks/tools/)是Kubernetes的命令行工具，方便用户通过一些简单的命令去管理Kubernetes。

如果事先安装了[docker desktop](https://desktop.docker.com/mac/stable/Docker.dmg), 那么`kubectl`已经安装好了，
如果没有安装，那可以使用homebrew安装：

```
brew install kubectl 
```

其他平台可以访问[kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)

## 验证

新建终端，输入`kubectl get po -A`,显示以下类似命令就算启动成功了：

```
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-7d89d9b6b8-d462h           0/1     Running   0          20s
kube-system   etcd-minikube                      1/1     Running   0          32s
kube-system   kube-apiserver-minikube            1/1     Running   0          35s
kube-system   kube-controller-manager-minikube   1/1     Running   0          32s
kube-system   kube-proxy-gcbpj                   1/1     Running   0          20s
kube-system   kube-scheduler-minikube            1/1     Running   0          31s
kube-system   storage-provisioner                1/1     Running   0          30s
```


## 参考资料

以下是本文撰写过程中的参考资料：

1. [minikube start](https://minikube.sigs.k8s.io/docs/start/)
2. [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
3. [Minkube安装笔记](https://damingerdai.github.io/software/k8s/how-to-install-minkube/)
4. [Minikube - Kubernetes本地实验环境](https://developer.aliyun.com/article/221687)