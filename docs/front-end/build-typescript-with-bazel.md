# 使用bazel编译TypeScript

## 准备

请事先安装[Nodejs](https://nodejs.org/dist/v14.17.3/),[Yarn 1.x](https://classic.yarnpkg.com/en/docs/install)和[Bazel](https://docs.bazel.build/versions/4.2.0/install.html)

我使用的版本为:

1. Nodejs: v14.17.3
2. Yarn:    1.22.5
3. Bzel:    4.1.0

## 创建一个Typescript项目

选择指定目录，创建一个名为`ts-bazel`(其他名字也可以)的文件夹，使用终端进入该文件夹，然后执行`npm init`，一路选择默认。

安装Typescipt:

```
yarn add typescipt -D
```

创建Typescript配置文件

```
npx tsc --init
```

创建`src`文件夹，在该文件夹里新建`index.ts`文件，并写入一下内容：

```Typescript
function sayHello(name: string) {
    console.log(`helle ${name}`);
}

sayHello('daming');
```

## 配置Bazel

安装bazel等相关依赖：

```shell
yarn add @bazel/bazelisk @bazel/ibazel @bazel/typescript -D 
```

在根目录里创建`WORKSPACE`, 并写入以下内容：

```Bazel
workspace(
    name = "ts-bazel",
    managed_directories = {"@npm": ["node_modules"]},
)

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "build_bazel_rules_nodejs",
    sha256 = "275744d287af4c3a78d7c9891f2d970b7bc7eca8cfc0e9a671fe6258d09ff217",
    urls = ["https://github.com/bazelbuild/rules_nodejs/releases/download/4.0.0-rc.1/rules_nodejs-4.0.0-rc.1.tar.gz"],
)

load("@build_bazel_rules_nodejs//:index.bzl", "check_rules_nodejs_version", "node_repositories", "yarn_install")

check_rules_nodejs_version(minimum_version_string = "2.2.0")

# Setup the Node.js toolchain
node_repositories(
    node_version = "14.17.3",
    package_json = ["//:package.json"],
)

yarn_install(
    name = "npm",
    package_json = "//:package.json",
    yarn_lock = "//:yarn.lock",
)
```

在根目录中创建`BUILD.bazel`文件，并写入以下内容：

```Bazel
package(default_visibility = ["//visibility:public"])

exports_files(["tsconfig.json"])
```

在`src`文件夹中创建`BUILD.bazel`文件，并写入以下内容：

```Bazel
package(default_visibility = ["//visibility:public"])

load("@npm//@bazel/typescript:index.bzl", "ts_project")


ts_project(
  name = "index",
  srcs = ["index.ts"],
  tsconfig = "//:tsconfig.json",
  visibility = ["//visibility:public"],
)
```

## 编译

现在可以使用`bazel`编译项目了！

```
bazel build //src:index
```

检查一下结果

```
node  bazel-bin/src/index.js
```

输出结果为：

```
helle daming
```