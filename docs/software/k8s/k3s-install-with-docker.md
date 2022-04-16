# K3S安装的时候默认使用docker

## 问题

k8s在1.20之后就弃用docker的运行时了，所以k3s也开始默认使用[containerd](https://containerd.io/)作为默认的运行时。
这导致一个问题。那就是k3s无法访问本地docker镜像。

## 解决方案

一种解决方案就是使用containerd的镜像，而不是docker。还有一种方式是在安装时就指定k3s使用docker作为运行环境。

```shell
/usr/local/bin/k3s-uninstall.sh
curl -sfL https://get.k3s.io | sh -s - server --docker
```

国内用户，可以使用以下方法加速安装：

```
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -s - server --docker
```
