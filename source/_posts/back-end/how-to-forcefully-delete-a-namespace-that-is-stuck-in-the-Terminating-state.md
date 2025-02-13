---
title: 在k8s中如何强制删除处于Terminating状态的namesapce
date: 2025-02-13 21:04:19
tags: [k3s, k8s]
categories: [后端]
---

## 概述

在k3s中删除一个namespace十分简单，就是一个命令的事儿：

```bash
kubectl delete ns ${namespace}
```

但是可能存在删除失败或者namespace一直处于Terminating状态的话，那么上面的命令可能行不通。
这里介绍两种实用的解决方案去帮助我们解决。

注意: 这两种方案可能存在错误删除的情况，请谨慎操作。

## 强制删除

kubectl提供force和grace-period=0两个参数帮助我们强制删除namespace：

```bash
kubectl delete ns ${namespace} --force --grace-period=0
```

说明：

* --force：强制删除资源，跳过正常的删除流程。

* --grace-period=0：立即删除资源，不等待任何清理操作。

## 使用Kubernetes API 删除

获取处于Terminating状态的namespace：

```bash
kubectl get ns |grep Terminating |awk {'print $1'}
```

调用Kubernetes API 删除指定的namespace

```bash
kubectl get ns ${namespace} -o json >${namespace}.json
sed -i '/"kubernetes"/d' ${namespace}.json
kubectl replace --raw "/api/v1/namespaces/${namespace}/finalize" -f ${namespace}.json
```

## 参考资料

1. [Kubernetes 的 NameSpace 无法删除应该怎么办](https://segmentfault.com/a/1190000042972476)
2. [kubernetes批量删除长期处于Terminating状态的namespace](https://www.cnblogs.com/sunshine99/p/17265300.html)