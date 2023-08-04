---
title: Angular和React构建体积简单对比
date: 2021-02-03 13:20:51
tags: [angular, react]
categories: [前端]
---

# Angular和React构建体积简单对比

## 前言

[Angular](https://angular.io/)是我主要使用的前端框架， 和[React](https://reactjs.org/)是我最近正在学习的前端框架。今天我想对比一下在各自默认的情况下，两者打包体积的对比。

## Angular

###  创建

我们使用[Angular CLI: 11.1.2](https://www.npmjs.com/package/@angular/cli/v/11.1.2)简单创建一个angular项目:

```
ng new daming-angular-app

# ? Do you want to enforce stricter type checking and stricter bundle budgets in the workspace? Y
# ? Would you like to add Angular routing? Y
# ? Which stylesheet format would you like to use? CSS
```

然后终端会提示我们输入一些必要的参数。对于Y或N的选择，我们统一选择Y，对拥有多个选择项的，我们统一选择第一个值：

```
# ? Do you want to enforce stricter type checking and stricter bundle budgets in the workspace? Y
# ? Would you like to add Angular routing? Y
# ? Which stylesheet format would you like to use? CSS
```

这样子我们就创建好了一个angular项目。

### 打包

在angular.json文件里，我们对*outputHashing*值从`all`改成`none`，目的在打包的时候生成js和css文件不是加上hash值，方便统计。

在终端执行构建脚本:

```
ng build --prod
```

### 统计

终端输出打包结果为：

```
Initial Chunk Files | Names         |      Size
main.js             | main          | 212.27 kB
polyfills.js        | polyfills     |  35.73 kB
runtime.js          | runtime       |   1.45 kB
styles.css          | styles        |   0 bytes

                    | Initial Total | 249.45 kB
```

通过Finder查看js和css文件的大小发现:

1. main.js大小为217kB;
2. polyfills.js大小为37kB;
3. runtime.js为1kB;
4. styles.css为0b。

因此从finder角度来说，打包的总体积为255kB。

此外，我们使用[gzip-size-cli](https://www.npmjs.com/package/gzip-size-cli)获取gzip的大小:

```
npx gzip-size-cli main.js       // 输出 64.0kB
npx gzip-size-cli polyfills.js  // 输出 12.4kB
npx gzip-size-cli runtime.js    // 输出 719B
npx gzip-size-cli styles.css    // 输出 20B
```

通过gzip压缩之后，打包的总体积为77.14kB.

## React

### 创建

我们使用[Create React App](https://create-react-app.dev/)创建一个react项目：

```
npx create-react-app daming-react-app
```

与angular不同，创建react项目的时候终端不会提示我们输入必要的参数。

### 打包

在终端执行构建脚本:

如果你使用npm

```
npm run build 
```

如果你使用yarn

```
yarn build 
```

### 统计

终端输出打包结果为：

```
File sizes after gzip:

  41.21 KB  build/static/js/2.6a046b6b.chunk.js
  1.4 KB    build/static/js/3.743bc3fe.chunk.js
  1.17 KB   build/static/js/runtime-main.073f5272.js
  597 B     build/static/js/main.2c9755c1.chunk.js
  531 B     build/static/css/main.8c8b27cf.chunk.css
```

经过统计， gzip总体积为44.91kB。

通过Finder查看js和css文件的大小发现:

1. main.2c9755c1.chunk.js大小为1kB;
2. runtime-main.073f5272.js大小为2kB;
3. 2.6a046b6b.chunk.js大小为131kB;
4. 3.743bc3fe.chunk.js大小为4kB;
5. main.8c8b27cf.chunk大小为804B。

由此可知，从finder角度来说，打包的总体积为138.80kB。

## 总结

通过上面简单的对比，在各自默认的创建、构建方式下，react在打包体积大小方面比angular更具优势。但是在实际开发中，我们都会使用大量的第三方的依赖，实际项目的打包体积大小还是需要因人而异的。

源代码已经上传到[github](https://github.com/damingerdai/imple-comparison-of-angular-and-react-build-size-code)。