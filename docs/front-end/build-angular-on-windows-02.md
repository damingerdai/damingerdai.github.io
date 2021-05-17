# 在windows上构建angular项目 (下)

当完成bazel的安装之后，我们还需要安装nodejs就可以开始编译angular了。

## Nodejs

目前angular仅支持nodejs12和nodejs14这两个版本，推荐使用node14这个版本。

对于windows平台而言，nodejs可以直接从[官网](https://nodejs.org/en/)选择windows平台的二进制包下载，然后进行点击安装就可以了，但是我个人更推荐使用[nvm-windows](https://github.com/coreybutler/nvm-windows)。

### nvm-windows

nvm-windows是windows平台上常用的node版本管理工具，可以方便我们针对不同项目的要求切换不同的node版本。点击[该链接](https://github.com/coreybutler/nvm/releases)，下载最新安装包，然后点击安装。

#### 使用国内镜像

由于国家特殊的网络政策，我们需要使用淘宝镜像:

```
nvm node_mirror http://npm.taobao.org/mirrors/node/ // 注意结尾有斜杠
nvm npm_mirror https://npm.taobao.org/mirrors/npm/
```

#### 安装nodejs14

目前node14的最新版是14.17.0，现在我们可以使用nvm来进行安装:

```
nvm install v14.17.0
nvm use v14.17.0
```

现在我们可以验证一下node是否正确安装：

```
node --version

// 结果
v14.17.0
```

## Yarn

[Yarn](https://classic.yarnpkg.com/lang/en/)是Facebook开发的nodejs包管理工具，与早期npm相比，具有下载速度快，命令更加友好，提供本版本锁等优势。当然，虽然npm不断的更新迭代，yarn的优势也不再明显，但是angular开发团队更加推荐使用yarn。

在这里需要注意的是，[Yarn1.x](https://classic.yarnpkg.com/lang/en/)和[Yarn2.x](https://yarnpkg.com/)是两个不兼容的版本，而angular只支持yarnv1.22.4以上且不包括2的版本(即>=1.22.4 <2)。因此在下载的时候请选择正确的版本。

## Angular

当bazel、nodejs、yarn都安装完毕之后，我们终于可以在本地开始编译angular了。

### 下载angular仓库

在终端执行clone命令

```
git clone --depth=16 https://github.com/angular/angular.git
```

> `--depth=16`表示仅仅git只会拉去最新16个commit。

### 安装angular依赖

在安装依赖之前，我们需要先删除`sauce-connect`依赖，因为该依赖不支持windows，且也不影响我们的编译。

在终端执行yarn命令：

```
yarn
```

### 编译angular

在终端执行编译脚本

```
node scripts/build/build-packages-dist.js
```

该命令将会先后编译angular,angular-in-memory-web-api和zone.js三大模块。

> 如果编译过程中发生错误，可以在重新执行编译脚本之前，执行clean命令：
> ```
> bazel clean
> ```

### 结果

当上面的命令正确执行完成之后，检查一下dist目录下的编译后的angular代码了：

1. `dist\angular-in-memory-web-api-dist\angular-in-memory-web-api`目录下是[angular-in-memory-web-api](https://www.npmjs.com/package/angular-in-memory-web-api)；
2. `dist\packages-dist`目录下便是angular各个模块的代码了；
3. `dist\zone.js-dist`目录下的是[zone.js](https://www.npmjs.com/package/zone.js)。

## 总结

到此，windows上编译angular就算是完成了。angular使用bazel作为编译工具，确实导致编译的成本高了很多，尤其是在windows平台，但是不停探索的过程，是一个不断学习的过程。这个过程，不挣钱，就是交个朋友。
