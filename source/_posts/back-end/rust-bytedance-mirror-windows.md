---
title: 在windows系统中如何给rust如何配置字节源
date: 2025-04-30 16:57:06
tags: [windows, rustl]
categories: [后端]
---

# rust中如何配置字节源

## 前言

本文的目的是在windows上安装或更新rust的时候通过配置字节的国内源来提高安装/更新速度。非windows系统就可以直接参考[rsproxy](https://rsproxy.cn/)

## 配置说明

在powershell中设置 Rustup 镜像

```powershell
$ENV:RUSTUP_DIST_SERVER='https://rsproxy.cn'
$ENV:RUSTUP_UPDATE_ROOT='https://rsproxy.cn/rustup'
```

安装或者更新Rust

如果是安装rust，直接点击`rust-init.exe`

如果是更新rust，直接运行rustup update命令

```powershell
rustup update
```

## 设置 crates.io 镜像

在当前用户的主目录（以我本地为例：D:\Users\daming）下的`.cargo`文件夹中创建`config`文件。

```toml
[source.crates-io]
replace-with = 'rsproxy-sparse'
[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"
[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"
[net]
git-fetch-with-cli = true
```


## 参考资料

1. [rsproxy](https://rsproxy.cn/)
2. [Windows 11 上通过国内源安装 Rust](https://www.sunzhongwei.com/windows-11-install-rust-with-china-mirror)
3. [rustup-init.exe 安装失败及其解决方案](https://www.cnblogs.com/manqing321/p/17026725.html)