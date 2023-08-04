---
title: 两种在Minikube中运行本地Docker镜像的简单方式
date: 2022-03-23 23:20:25
tags: [minikube, docker, k8s]
categories: [后端]
---

# 两种在Minikube中运行本地Docker镜像的简单方式

## 前言

本文将分享两种在Minikube中运行本地Docker镜像的简单方式

![Kubernetes](https://miro.medium.com/max/1400/1*Q8RckeLvx7rVFNPTHzx21A.png)

## 要求

- 安装并运行[Docker](https://www.docker.com/get-started)
- 安装并运行[Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [安装kubectl](https://kubernetes.io/docs/tasks/tools/)

## 你将学到的知识

本文将教会大家如何在Minikube中使用本地的Docker镜像。因为Kubernetes默认从注册表中提取镜像，所以Kubernetes一般是不会使用本地镜像，并且在生产环境中也不应该使用本地镜像。但是，如果我们可以很轻松的使用本地镜像，而不是每次都需要将这些镜像推送到远程的注册表、登录远程注册表并在本地电脑中重新拉取，那么使用本地镜像将十分便利。

和往常一样，我在github中准备了一个[仓库](https://github.com/Abszissex/medium-local-docker-image-minikube), 方便大家查看完成的代码库，并且可以按照本文描述的步骤进行操作。

## Demo介绍

```
/
|- app/
  |- Dockerfile
  |- index.js
  |- package.json
|- deployment.yaml
```

在上面的文件夹结构中，我将重点介绍将在本文中使用的重点文件：

- **app/Dockerfile** 用于构建包含一个Node.js Web 服务器的本地Docker镜像的Dockerfile，我们将其部署到Minikube

- **app/index.js** Node.js Web 服务器的应用程序代码

- ***app/package.json** 我们Node.js Web 服务器的依赖。在本文中，只使用了**express**，一个用于搭建Web服务器的Nodejs库

- **deployment.yaml** 在Kubernetes中运行Node.js Web 服务器的Deployment配置

**app**文件夹中的实际内容和本文无关。 我仅仅是提供一个demo方便接下来的教程讲解，当然使用你自己的demo也是可以的。如果你想要使用**app**这个应用，请注意服务器将在容器内部中监听8080端口。

## Deployment配置

**deployment.yaml**的内容如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    name: myapp
spec:
  selector:
    matchLabels:
      name: myapp
  template:
    metadata:
      labels:
        name: myapp
    spec:
      containers:
        - name: myapp
          image: pz/demo
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
```

我们需要重点关注的是**imagePullPolicy**和**containerPort**这两个配置

通过**containerPort**，我们将暴露app正在监听的*8080*端口，因此我们稍后可以通过浏览器访问*http://127.0.0.1:8080*去验证是否满足我们预期那样工作。

更关键的是**imagePullPolicy**。如果你想使用本地Docker镜像，**imagePullPolicy**需要设置成**Never**, 否则， Kubernetes将在注册表中根据你提供的名字来搜素同名镜像。

## 构建Dcoker镜像

为了验证Docker镜像能够按照我们的预期在Kubernetes中运行，让我们来构建并运行。

首先，我们导航进入**app**目录，然后我们通过**docker build -t pz/demo .**构建Docker镜像，通过**-t**参数将镜像名字设置为**pz/demo**。

![docker images](https://miro.medium.com/max/1400/1*Q-XZFc-_TRrTfwS_-DtHQw.png)

当构建完成之后，我们可以通过**docker run -it --rm -p 8080:8080 pz/demo**命令来运行容器，并将Docker的8080端口映射的本地的8080端口。接下来，我们可以在浏览器中访问[localhost:8080](http://localhost:8080)。 如果我们可以在浏览器中看到*"Hello World!"*，那么说明我们的容器运行正常。

![localhost:8080](https://miro.medium.com/max/1224/1*BrzMRioHmZEaR1km1C0EWA.png)

## 在Minikube中运行本地Docker镜像

如果你想通过**kubectl apply -f deployment.yaml**命令部署上面的**deployment.yaml**到你的Minikube中，那启动的Pod将找不到你刚刚构建的Docker镜像。

你可以通过**kubectl logs deployment.apps/myapp**命令来检查日志去来验证这个错误的结果。

![kubectl logs deployment.apps/myapp](https://miro.medium.com/max/1400/1*qGB6vXA3rpRnubXQ9Qf3oQ.png)

上面的日志显示由于Kubernetes拉取不到镜像Pod一直等待重启。这其实是因为Minikube没有方法获取你本地Docker镜像。

但是幸运的是，有两个简单的命令可以帮助解决这个问题。

第一种方式是使用**image load**命令， 你可以使用下面的名利将本地Docker镜像从本地电脑中导入Minikube中：

```shell
# General
minikube image load <IMAGE_NAME>
# Example
minikube image load pz/demo
```

在导入完成之后，你可以重启pod。然后你可以发现pod可以正常工作了。

其实我们还可以变得更简单。以前的方法是你需呀先在本地构建Docker镜像，然后将其移动到Minikube容器中，虽然耗时不多，但是终究还是浪费了不少时间。

通过使用Minikube的**image build**， 我们可以在Minikube中直接构建镜像：

```shell
# General
minikube image build -t <IMAGE_NAME> .
# Example
minikube image build -t pz/demo .
```

使用**minikube image build**构建出来的镜像可以在Minikkube中直接访问，同时也不再需要在**minikube image load**中的第二步导入步骤。

使用以上两种之一的方法，我们可以重新检查一下Deployment的日志：

![kubectl logs deployment.apps/myapp](https://miro.medium.com/max/1400/1*-yXXk7reYwXm_w3folq6kQ.png)

为了进一步验证一切是否按照我们预期的工作，我们可以使用下面的命令将本地8080端口转发发Deployment的8080端口中：


```shell
kubectl port-forward deployment/myapp 8080:8080
```

重新检查浏览器，我们可以发现本地构建的应用在Minikube中运行正常，🎉🎉🎉

![localhost:8080](https://miro.medium.com/max/1224/1*BrzMRioHmZEaR1km1C0EWA.png)

## 总结

通过本文，你应该能够使用**minikube image load**和**minikube image buildcommand**命令在Minikube中使用本地镜像了.

更多信息请关注[LinkedIn](https://www.linkedin.com/in/pascal-zwikirsch-3a95a1177/)

## 后言

原文：[Two easy ways to use local Docker images in Minikube](https://medium.com/gitconnected/two-easy-ways-to-use-local-docker-images-in-minikube-cd4dcb1a5379)

译文： [两种在Minikube中运行本地Docker镜像的简单方式](https://damingerdai.github.io/back-end/two-easy-ways-to-use-local-docker-images-in-minikube/)

译者：[大明二代](https://damingerdai.github.io/)