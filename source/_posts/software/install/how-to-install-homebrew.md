---
title: Homebrew安装笔记
date: 2021-07-31 22:18:03
tags: [mac, homebew]
categories: [软件]
---

# Homebrew安装笔记

简单记录一下Homebrew安装

## 下载

新建终端，以下命令安装：

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## 配置国内镜像

替换 brew.git

```
cd "$(brew --repo)" && git remote set-url origin https://mirrors.ustc.edu.cn/brew.git
```

替换 homebrew-core.git

```
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core" && git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git
```

Homebrew Bottles 配置镜像

以zsh为例

```
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.zshrc && source ~/.zshrc
```

## 恢复默认官方源

重置brew.git:

```shell
cd "$(brew --repo)" && git remote set-url origin https://github.com/Homebrew/brew.git
```

重置homebrew-core.git:

```
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core" && git remote set-url origin https://github.com/Homebrew/homebrew-core.git
```

注释掉 zsh 配置文件里的有关 Homebrew Bottles 即可恢复官方源。 重启 zsh 或让 zsh 重读配置文件。

## 参考资料

[教你一招搞定 Homebrew 下载加速](https://zhuanlan.zhihu.com/p/137464385)
[替换及重置 Homebrew 默认源](https://lug.ustc.edu.cn/wiki/mirrors/help/brew/)