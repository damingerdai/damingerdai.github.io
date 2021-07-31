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

```
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.zshrc && source ~/.zshrc
```

## 参考资料

[教你一招搞定 Homebrew 下载加速](https://zhuanlan.zhihu.com/p/137464385)