---
title: "在windows上构建angular项目(上)"
date: 2021-05-13 22:50:19
tags: [angular,bazel]
categories: [前端]
---
# 在windows上构建angular项目 (上)

前端三大框架( [angular](https://angular.io/), [reac](https://reactjs.org/), [vue](https://vuejs.org/) )中，angular一直都是一个很独特的存在。首先，angular的概念很多，服务、依赖注入、模块，指令等，都是在前端圈不是很常用的，此外，angular使用了[bazel](https://www.bazel.build/)作为构建工具，而react和vue都是使用了[rollup](https://rollupjs.org/guide/en/)，因此在本地编译构建angular将会远远超过react和vue，如果你是用的windows平台，那么一个个坑需要自己慢慢来填。。。

# Bazel是什么？

根据官网的定义，Bazel是类似于Make，Maven和Gradle的开源构建和测试工具。它使用人类可读的高级构建语言[Starlark](https://github.com/bazelbuild/starlark)(一种基于python的方言)。 Bazel支持多种语言的项目，并为多种平台构建输出。 

从我个人角度来看，bazel是一个强大且复杂的构建系统，通过`build rule`的概念，支持多种语言、不同平台，支持构建C/C++,Java,Android,IOS,Golang,Nodejs,Docker项目

## 安装

Bazel官方支持Windows，macOS, Ubuntu Linux三大平台，这也是开发人员比较常用的本地开发平台。

在社区的支持下，bazel还支持其他平台，具体信息可以看一下[官网](https://docs.bazel.build/versions/4.0.0/install.html)。这里我仅仅介绍如何在windows上安装。

### 安装要求

在安装前，请确保你windows系统符合要求：
1. 推荐使用64位的windows10，版本号不能低于1703
2. 64位的Windows 7以上
3. 64位的Windows Server 2008 R2以上


此外，请事先安装[Visual C++ Redistributable for Visual Studio 2015](https://www.microsoft.com/en-us/download/details.aspx?id=48145)

### 下载安装Bazel

我们需要在[github](https://github.com/bazelbuild/bazel/releases)选择下载`bazel-<version>-windows-x86_64.exe`，然后将该文件路径放在path环境变量中。

在终端中执行
```
$ bazel version
```

显示类似效果就是可以了

```
Build label: 4.0.0
Build target: bazel-out/x64_windows-opt/bin/src/main/java/com/google/devtools/build/lib/bazel/BazelServer_deploy.jar
Build time: Thu Jan 21 07:39:16 2021 (1611214756)
Build timestamp: 1611214756
Build timestamp as int: 1611214756
```

## 安装编译和语言运行时

根据不同语言的编译需要，我们可能需要以下的工具：

1. [MSYS2 x86_64](https://www.msys2.org/)
2. [Build Tools for Visual Studio 2019](https://aka.ms/buildtools)
3. [Java SE Development Kit 11 (JDK) for Windows x64](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)
4. [Python 3.6 for Windows x86-64](https://www.python.org/downloads/windows/)

如果你想在window平台编译构建angular，后三者必须的，JDK16和Python 3.9.5也能满足自己的需求。

# 总结

当完成以上步骤的时候，我们基本上就可以开始尝试在windows本地进行编译angular。