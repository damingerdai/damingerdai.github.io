---
title: Debian 13 上安装最新版 Ruby：使用 rbenv 管理 Ruby 版本
date: 2026-09-29 22:04:38
tags: [Ruby, rbenv, debian]
categories: [软件]
---

# Debian 13 上安装最新版 Ruby

Ruby 是一门动态、面向对象的编程语言，以简洁、优雅的语法著称。对于需要维护 Ruby 项目或学习 Ruby on Rails 的开发者来说，配置合适的 Ruby 开发环境是第一步。

本文介绍如何在 Debian 13 上使用 **rbenv + ruby-build** 安装 Ruby、配置 Zsh 开发环境、管理多个 Ruby 版本，并解决安装过程中常见的问题。

## 一、为什么选择 rbenv？

Debian 自带 Ruby 软件包，可以直接执行：

```bash
sudo apt install ruby
```

但是，Debian 软件仓库提供的 Ruby 版本不一定是最新版本。如果需要同时维护多个项目，不同项目之间还可能存在 Ruby 版本不兼容的问题。

因此，更推荐使用 [rbenv](https://github.com/rbenv/rbenv)。

rbenv 的主要优势包括：

- **多版本管理**：支持同时安装多个 Ruby 版本。
- **项目级版本切换**：通过 `.ruby-version` 为不同项目指定 Ruby 版本。
- **不影响系统 Ruby**：无须覆盖 Debian 自带的 Ruby。
- **方便升级**：可以独立安装和切换新版本。

需要注意，rbenv 本身只负责 Ruby 版本管理，实际下载和编译 Ruby 的功能由 [ruby-build](https://github.com/rbenv/ruby-build) 提供。

## 二、安装编译依赖

首先更新 Debian 软件包索引：

```bash
sudo apt update
```

然后安装 Ruby 编译所需的依赖：

```bash
sudo apt install -y \
  git \
  curl \
  build-essential \
  autoconf \
  bison \
  rustc \
  libssl-dev \
  libyaml-dev \
  libreadline-dev \
  zlib1g-dev \
  libffi-dev \
  libgdbm-dev
```

其中：

- `build-essential`：提供 GCC、Make 等基础编译工具。
- `libssl-dev`：提供 OpenSSL 开发库。
- `libyaml-dev`：提供 YAML 解析库。
- `libreadline-dev`：支持交互式命令行编辑。
- `zlib1g-dev`：提供数据压缩支持。
- `rustc`：为需要 Rust 工具链的构建步骤提供支持。

## 三、安装 rbenv

### 3.1 克隆 rbenv

使用 Git 将 rbenv 安装到当前用户的 Home 目录：

```bash
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
```

如果已经通过其他方式安装了 rbenv，建议先检查当前安装路径，避免重复安装。

```bash
which rbenv
rbenv root
```

### 3.2 安装 ruby-build

ruby-build 是 rbenv 的插件，负责下载和编译 Ruby。

```bash
mkdir -p ~/.rbenv/plugins

git clone https://github.com/rbenv/ruby-build.git \
  ~/.rbenv/plugins/ruby-build
```

这里有一个容易忽略的问题：如果之前通过 APT 安装过 rbenv，实际的插件目录可能与 `~/.rbenv/plugins` 不一致。

因此，也可以使用以下命令，将 ruby-build 安装到当前 rbenv 实际使用的插件目录：

```bash
mkdir -p "$(rbenv root)/plugins"

git clone https://github.com/rbenv/ruby-build.git \
  "$(rbenv root)/plugins/ruby-build"
```

对于全新安装，建议统一使用 Git 安装的 rbenv，避免不同安装方式之间产生路径冲突。

注意：以上两组命令是不同安装场景下的替代方案，不需要重复执行。

## 四、配置 Shell 环境

本文使用 Zsh 作为默认 Shell。

编辑配置文件：

```bash
nano ~/.zshrc
```

添加以下内容：

```bash
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init - zsh)"
```

保存后重新加载配置：

```bash
source ~/.zshrc
```

检查 rbenv 是否正常：

```bash
rbenv --version
rbenv root
```

如果使用 Bash，则需要将配置写入 `~/.bashrc`：

```bash
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init - bash)"
```

然后执行：

```bash
source ~/.bashrc
```

## 五、安装最新版 Ruby

### 5.1 更新 ruby-build

ruby-build 会持续更新 Ruby 版本定义。

在安装新版本之前，建议先更新：

```bash
git -C "$(rbenv root)/plugins/ruby-build" pull
```

### 5.2 查看可安装版本

执行：

```bash
rbenv install -l
```

该命令会列出 ruby-build 当前支持的最新稳定版本。

如果需要查看所有可安装版本：

```bash
rbenv install -L
```

建议根据命令实际输出选择需要安装的 Ruby 版本。

### 5.3 编译安装 Ruby

例如，假设准备安装 Ruby 4.0.7：

```bash
rbenv install 4.0.7
rbenv rehash
```

ruby-build 会自动下载 Ruby 源码，并在本地完成编译和安装。

安装时间取决于 CPU 性能和网络环境。

如果安装过程中出现错误，可以查看终端输出中提示的构建日志，确定是否缺少编译依赖。

### 5.4 设置默认版本

安装完成后，执行：

```bash
rbenv global 4.0.7
```

然后更新 shims：

```bash
rbenv rehash
```

验证安装结果：

```bash
ruby --version
which ruby
```

如果配置正确，`which ruby` 通常会返回类似路径：

```text
/home/username/.rbenv/shims/ruby
```

这说明当前使用的 Ruby 已经由 rbenv 管理。

## 六、常见问题：rbenv install 命令不存在

在安装过程中，我遇到了以下错误：

```text
$ rbenv install -l

rbenv: no such command `install'
```

这是一个比较典型的问题。

`rbenv install` 并不是 rbenv 自身提供的命令，而是由 ruby-build 插件提供。

如果 ruby-build 没有正确安装，或者插件安装到了错误的目录，就可能出现这个错误。

### 6.1 检查 rbenv 安装位置

首先执行：

```bash
which rbenv
rbenv root
```

检查当前使用的 rbenv 及其根目录。

### 6.2 检查 ruby-build

执行：

```bash
ls -la "$(rbenv root)/plugins"
```

检查是否存在 `ruby-build` 目录。

如果不存在，则安装插件：

```bash
mkdir -p "$(rbenv root)/plugins"

git clone https://github.com/rbenv/ruby-build.git \
  "$(rbenv root)/plugins/ruby-build"
```

如果插件目录已经存在，则尝试更新：

```bash
git -C "$(rbenv root)/plugins/ruby-build" pull
```

### 6.3 验证安装命令

执行：

```bash
rbenv commands | grep install
```

正常情况下，输出中应该包含：

```text
install
```

然后再次执行：

```bash
rbenv install -l
```

如果能够正常显示 Ruby 版本列表，说明问题已经解决。

### 6.4 排查多个 rbenv 安装

如果问题依然存在，可以检查系统中是否安装了多个 rbenv：

```bash
type -a rbenv
```

例如，可能同时存在：

```text
/home/username/.rbenv/bin/rbenv
/usr/bin/rbenv
```

这种情况下，需要确认当前 Shell 使用的是哪个 rbenv，以及 ruby-build 是否安装在对应的插件目录中。

建议统一安装方式，避免系统版本和用户目录版本混用。

## 七、安装 Bundler

Ruby 安装完成后，可以继续安装 Bundler。

Bundler 是 Ruby 生态中的依赖管理工具，作用类似于 JavaScript 生态中的 npm。

安装 Bundler：

```bash
gem install bundler
```

验证安装：

```bash
bundle --version
```

对于已经存在的 Ruby 项目，可以在项目根目录执行：

```bash
bundle install
```

Bundler 会根据项目中的 `Gemfile` 和 `Gemfile.lock` 安装对应的依赖。

由于当前使用 rbenv 管理 Ruby，通常不需要通过 `sudo` 安装 Gems。

## 八、管理多个 Ruby 版本

rbenv 的另一个优势是能够为不同项目配置不同的 Ruby 版本。

### 8.1 查看已安装版本

```bash
rbenv versions
```

### 8.2 安装其他 Ruby 版本

例如：

```bash
rbenv install 3.4.8
```

### 8.3 设置全局版本

```bash
rbenv global 4.0.7
```

全局版本是默认使用的 Ruby 版本。

### 8.4 设置项目版本

假设某个旧项目需要 Ruby 3.4.8。

进入项目目录：

```bash
cd my-ruby-project
```

设置当前项目使用的版本：

```bash
rbenv local 3.4.8
```

rbenv 会在当前目录创建 `.ruby-version` 文件：

```text
3.4.8
```

当进入这个目录时，rbenv 就会自动切换到对应版本。

离开项目目录后，则会恢复使用全局版本，除非其他版本配置具有更高优先级。

这对于需要同时维护新旧 Ruby 项目的开发者非常方便。

## 九、升级 Ruby

以后需要升级 Ruby 时，无须重新安装 rbenv。

首先更新 ruby-build：

```bash
git -C "$(rbenv root)/plugins/ruby-build" pull
```

查看最新版本：

```bash
rbenv install -l
```

安装需要的新版本：

```bash
rbenv install <version>
```

这里的 `<version>` 是占位符，需要替换成实际版本号。

随后切换全局版本：

```bash
rbenv global <version>
```

最后检查：

```bash
ruby --version
```

需要注意，安装新 Ruby 版本后，之前安装的第三方 Gems 不一定会自动迁移。

对于已有项目，建议重新执行：

```bash
bundle install
```

同时，应当检查项目是否支持新的 Ruby 版本。

## 十、总结

通过 rbenv 和 ruby-build，我们可以在 Debian 13 上建立一套灵活的 Ruby 开发环境。

相比直接通过 APT 安装 Ruby，这种方式更加适合需要维护多个项目、尝试新版 Ruby 或学习 Ruby on Rails 的开发者。

整个安装过程中，最值得注意的是两个问题：

1. **rbenv 和 ruby-build 是两个独立项目**，安装 rbenv 后，还需要确保 ruby-build 已经正确安装。
2. **避免不同安装方式产生路径冲突**，尤其是在同时使用 APT 和 Git 安装 rbenv 的情况下。

最后，对于实际项目，建议优先遵循项目中的 `.ruby-version` 和 `Gemfile` 配置，而不是一味追求最新版本。

## 参考资料

- [Ruby 官方网站](https://www.ruby-lang.org/)
- [rbenv GitHub](https://github.com/rbenv/rbenv)
- [ruby-build GitHub](https://github.com/rbenv/ruby-build)
- [Bundler 官方文档](https://bundler.io/)
