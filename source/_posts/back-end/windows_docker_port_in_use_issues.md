---
title: 解决Windows下Docker启动容器时，端口被占用错误
date: 2023-02-26 19:54:58
tags: [windows, docker, docker desktop]
categories: [后端]
---

# 解决Windows下Docker启动容器时，端口被占用错误

## 问题描述

在使用docker-compose启动mysql的时候遇到一个问题:

```bash
bind: An attempt was made to access a socket in a way forbidden by its access permissions.
```

然后查了一下是否存在应用占用了3306的端口：

```
netstat -ano | findstr 3306
```

结果很尴尬，并没有。。。

百度了一下，发现是`Hyper-V`会保留部分tcp端口导致我们自己无法使用这些端口, 使用如下命令查看保留的端口：

```powershell
netsh interface ipv4 show excludedportrange protocol=tcp

协议 tcp 端口排除范围

开始端口    结束端口
----------    --------
      1026        1125
      1126        1225
      1226        1325
      1326        1425
      1426        1525
      1538        1637
      2327        2426
      2427        2526
      2527        2626
      2627        2726
      2727        2826
      2827        2926
      6344        6567
     50000       50059     *
     50070       50070     *

* - 管理的端口排除。

```

## 解决方案

解决方案很多，但是最推荐的该市修改永久排除保留端口：

首选关闭docker, 用管理员权限打开powershell， 输入以下命令用于关闭Hyper-V:

```bash
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V
```

> 该操作可能需要重启

然后将3306端口永久排除：

```bash
netsh int ipv4 add excludedportrange protocol=tcp startport=3306 numberofports=1 store=persistent
```

最后重启Hyper-V:

```
dism.exe /Online /Enable-Feature:Microsoft-Hyper-V /All
```

## 参考资料

1. [解决Windows下Docker启动容器时，端口被占用错误](https://www.cnblogs.com/uncmd/p/16056993.html)
2. [修改Hyper-V动态端口范围](https://www.cnblogs.com/fansys/articles/13375989.html)