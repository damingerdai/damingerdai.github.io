---
title: 如何使用bazel去构建基于express和typescript的nodejs项目
date: 2023-08-10 22:10:03
tags: [nodejs, typescript, bazel]
categories: [前端]
cover: https://cn.bing.com/th?id=OHR.HokkaidoShida_ZH-CN0103354943_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp
---

## 前言

Bazel 是一款类似于 Make、Maven 和 Gradle的开源构建和测试工具。它使用可读的高级构建语言，支持多种变成语言编写的项目，并且能够为多个平台进行构建。Bazel 支持构建包含多个仓库、大量开发人员的大型代码库。

详细介绍可见[Bazel官网](https://bazel.build/)。

## 目的

本文的目的是使用bazel5去构建一个完整的nodejs后端项目，并不负责bazel相关知识的介绍。

## 配置

首先在`package.json`文件中·devDependencies·部分添加：

```json
    "@bazel/bazelisk": "^1.7.5",
    "@bazel/buildifier": "^6.0.0",
    "@bazel/ibazel": "^0.16.0",
    "@bazel/typescript": "^5.8.1",
```

然后再执行安装依赖命令。

创建`.bazelignore`文件并写入下面的内容:

```bazel
.git
node_modules
dist
```

创建`.bazelrc`文件并写入下面的内容:

```bazel
# Enable debugging tests with --config=debug
test:debug --test_arg=--node_options=--inspect-brk --test_output=streamed --test_strategy=exclusive --test_timeout=9999 --nocache_test_results

# Do not attempt to de-flake locally.
# On CI we might set this to `3` to run with deflaking.
test --flaky_test_attempts=1

###############################
# Filesystem interactions     #
###############################

# Create symlinks in the project:
# - dist/bin for outputs
# - dist/testlogs, dist/genfiles
# - bazel-out
# NB: bazel-out should be excluded from the editor configuration.
# The checked-in /.vscode/settings.json does this for VSCode.
# Other editors may require manual config to ignore this directory.
# In the past, we say a problem where VSCode traversed a massive tree, opening file handles and
# eventually a surprising failure with auto-discovery of the C++ toolchain in
# MacOS High Sierra.
# See https://github.com/bazelbuild/bazel/issues/4603
build --symlink_prefix=dist/

# Turn off legacy external runfiles
build --nolegacy_external_runfiles
run --nolegacy_external_runfiles
test --nolegacy_external_runfiles

# Turn on --incompatible_strict_action_env which was on by default
# in Bazel 0.21.0 but turned off again in 0.22.0. Follow
# https://github.com/bazelbuild/bazel/issues/7026 for more details.
# This flag is needed to so that the bazel cache is not invalidated
# when running bazel via `yarn bazel`.
# See https://github.com/angular/angular/issues/27514.
build --incompatible_strict_action_env
run --incompatible_strict_action_env
test --incompatible_strict_action_env

# Do not build runfile trees by default. If an execution strategy relies on runfile
# symlink teee, the tree is created on-demand. See: https://github.com/bazelbuild/bazel/issues/6627
# and https://github.com/bazelbuild/bazel/commit/03246077f948f2790a83520e7dccc2625650e6df
build --nobuild_runfile_links

build --enable_runfiles

###############################
# Release support             #
# Turn on these settings with #
#  --config=release           #
###############################

# Releases should always be stamped with version control info
# This command assumes node on the path and is a workaround for
# https://github.com/bazelbuild/bazel/issues/4802
build:release --workspace_status_command="yarn -s ng-dev release build-env-stamp --mode=release"
build:release --stamp

# Building AIO against local Angular deps requires stamping
# versions in Angular packages due to CLI version checks.
build:aio_local_deps --stamp
build:aio_local_deps --workspace_status_command="yarn -s --cwd aio local-workspace-status"

# Snapshots should also be stamped with version control information.
build:snapshot-build --workspace_status_command="yarn -s ng-dev release build-env-stamp --mode=snapshot"
build:snapshot-build --stamp

##########################################################
# AIO architect build configuration                      #
# See aio/angular.json for available configurations.     #
# To build with a partiular configuration:               #
#   bazel build //aio:build --aio_build_config=<config>  #
# Default config is `stable``.                           #
##########################################################
build --flag_alias=aio_build_config=//aio:flag_aio_build_config

####################################
# AIO first party dep substitution #
# Turn on with                     #
#  --config=aio_local_deps         #
####################################

build:aio_local_deps --//aio:flag_aio_local_deps

###############################
# Output                      #
###############################

# A more useful default output mode for bazel query
# Prints eg. "ng_module rule //foo:bar" rather than just "//foo:bar"
query --output=label_kind

# By default, failing tests don't print any output, it goes to the log file
test --test_output=errors

################################
# Settings for CircleCI        #
################################

# Bazel flags for CircleCI are in /.circleci/bazel.linux.rc and /.circleci/bazel.windows.rc

##################################
# Remote Build Execution support #
# Turn on these settings with    #
#  --config=remote               #
##################################

# The following --define=EXECUTOR=remote will be able to be removed
# once https://github.com/bazelbuild/bazel/issues/7254 is fixed
build:remote --define=EXECUTOR=remote

# Set a higher timeout value, just in case.
build:remote --remote_timeout=600

# Bazel detects maximum number of jobs based on host resources.
# Since we run remotely, we can increase this number significantly.
common:remote --jobs=200

build:remote --google_default_credentials

# Limit the number of test jobs for on an AIO local deps build. The example tests running
# concurrently pushes the circleci executor RAM usage to its limits.
test:aio_local_deps --jobs=24

# Force remote exeuctions to consider the entire run as linux
build:remote --cpu=k8
build:remote --host_cpu=k8

# Toolchain and platform related flags
build:remote --crosstool_top=@npm//@angular/build-tooling/bazel/remote-execution/cpp:cc_toolchain_suite
build:remote --extra_toolchains=@npm//@angular/build-tooling/bazel/remote-execution/cpp:cc_toolchain
build:remote --extra_execution_platforms=@npm//@angular/build-tooling/bazel/remote-execution:platform
build:remote --host_platform=@npm//@angular/build-tooling/bazel/remote-execution:platform
build:remote --platforms=@npm//@angular/build-tooling/bazel/remote-execution:platform

# Remote instance and caching
build:remote --remote_instance_name=projects/internal-200822/instances/primary_instance
build:remote --project_id=internal-200822
build:remote --remote_cache=remotebuildexecution.googleapis.com
build:remote --remote_executor=remotebuildexecution.googleapis.com

# Use HTTP remote cache
build:remote-cache --remote_cache=https://storage.googleapis.com/angular-team-cache
build:remote-cache --remote_accept_cached=true
build:remote-cache --remote_upload_local_results=true
build:remote-cache --google_default_credentials

# Ensure that tags like "no-remote-exec" get propagated to actions created by rules,
# even if the rule implementation does not explicitly pass them to the execution requirements.
# https://bazel.build/reference/command-line-reference#flag--experimental_allow_tags_propagation
common --experimental_allow_tags_propagation

# Disable network access in the sandbox by default. To enable network access
# for a particular target, use:
#
# load("@npm//@angular/build-tooling/bazel/remote-execution:index.bzl", "ENABLE_NETWORK")
# my_target(
#   ...,
#   exec_properties = ENABLE_NETWORK,   # Enables network in remote exec
#   tags = ["requires-network"]         # Enables network in sandbox
# )
build --nosandbox_default_allow_network

##################################
# Saucelabs tests settings       #
# Turn on these settings with    #
#  --config=saucelabs            #
##################################

# For saucelabs tests we don't want to enable flaky test attempts. Karma has its own integrated
# retry mechanism and we do not want to retry unnecessarily if Karma already tried multiple times.
test:saucelabs --flaky_test_attempts=1

################
# Flag Aliases #
################

# --ng_perf will ask the Ivy compiler to produce performance results for each build.
build --flag_alias=ng_perf=//packages/compiler-cli:ng_perf

####################################################
# User bazel configuration
# NOTE: This needs to be the *last* entry in the config.
####################################################

# Load any settings which are specific to the current user. Needs to be *last* statement
# in this config, as the user configuration should be able to overwrite flags from this file.
try-import %workspace%/.bazelrc.user
```

创建`.bazelversion`用于指定bazel版本：

```bazel
5.0.0
```

> 可以使用bazel5的任意版本，理论上都支持，但是不要使用bazel6的版本

新建`WORKSPACE`用于定义bazel的工作区：

```bazel
workspace(
   name = "express-postgres-ts-starter",
    managed_directories = {
        "@npm": ["node_modules"],
    },
)
```

导入`http_archive`用于获取bazel的库：

```bazel
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
```

如果使用了`.yarn`文件夹用于限制yarn的版本，可以通过创建`yarn.bzl`解决：

```
YARN_PATH = ".yarn/releases/yarn-1.22.19.cjs"
YARN_LABEL = "//:" + YARN_PATH
```

然后在`WORKSPACE`导入：

```
load("//:yarn.bzl", "YARN_LABEL")
```

引入bazel的nodesj依赖库：

```bazel
http_archive(
    name = "build_bazel_rules_nodejs",
    sha256 = "5dd1e5dea1322174c57d3ca7b899da381d516220793d0adef3ba03b9d23baa8e",
    urls = [
        "https://github.com/bazelbuild/rules_nodejs/releases/download/5.8.3/rules_nodejs-5.8.3.tar.gz",
        "https://ghproxy.com/https://github.com/bazelbuild/rules_nodejs/releases/download/5.8.3/rules_nodejs-5.8.3.tar.gz"
    ],
)

load("@build_bazel_rules_nodejs//:repositories.bzl", "build_bazel_rules_nodejs_dependencies")

build_bazel_rules_nodejs_dependencies()

http_archive(
    name = "rules_pkg",
    sha256 = "62eeb544ff1ef41d786e329e1536c1d541bb9bcad27ae984d57f18f314018e66",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_pkg/releases/download/0.6.0/rules_pkg-0.6.0.tar.gz",
        "https://github.com/bazelbuild/rules_pkg/releases/download/0.6.0/rules_pkg-0.6.0.tar.gz",
        "https://ghproxy.com/https://github.com/bazelbuild/rules_pkg/releases/download/0.6.0/rules_pkg-0.6.0.tar.gz",
    ],
)

# Fetch Aspect lib for utilities like write_source_files
# NOTE: We cannot move past version 1.23.2 of aspect_bazel_lib because it requires us to move to bazel 6.0.0 which
#       breaks our usage of managed_directories
http_archive(
    name = "aspect_bazel_lib",
    sha256 = "4b2e774387bae6242879820086b7b738d49bf3d0659522ea5d9363be01a27582",
    strip_prefix = "bazel-lib-1.23.2",
    urls = [
        "https://github.com/aspect-build/bazel-lib/archive/refs/tags/v1.23.2.tar.gz",
        "https://ghproxy.com/https://github.com/aspect-build/bazel-lib/archive/refs/tags/v1.23.2.tar.gz",
    ]
)
```

配置nodejs工具链：

```
load("@rules_nodejs//nodejs:repositories.bzl", "nodejs_register_toolchains")

nodejs_register_toolchains(
    name = "nodejs",
    node_version = "18.10.0",
)
```

> 在arm mac上不支持nodejs20的版本，不确定其他平台时候是否支持nodejs20

配置`yarn_install`下载依赖：

```
load("@build_bazel_rules_nodejs//:index.bzl", "yarn_install")

yarn_install(
    name = "npm",
		data = [
        YARN_LABEL,
        "//:.yarnrc",
				"//:patches/trim-newlines+5.0.0.patch"
		],
		exports_directories_only = False,
		symlink_node_modules = True,
    package_json = "//:package.json",
		yarn = YARN_LABEL,
    yarn_lock = "//:yarn.lock",
)
```

创建`BUILD.bazel`, 并定义为公开的:

```
package(default_visibility = ["//visibility:public"])
```

导入`ts_project`和`nodejs_binary`用于构建typescript和nodejs：

```
load("@npm//@bazel/typescript:index.bzl", "ts_project")
load("@build_bazel_rules_nodejs//:index.bzl", "nodejs_binary")
```

定义`SRCS`变量用于声明所有需要参与编译的ts文件：

```
SRCS = glob(
	[
		"src/**/*.ts",
	],
	exclude = [
		"src/**/*.spec.ts"
	]
)
```

使用`ts_project`将ts编译成js：

```
ts_project(
    name = "server-ts",
    srcs = ["server.ts" ] + SRCS,
    resolve_json_module = True,
    source_map = True,
    tsconfig = "//:tsconfig.json",
    visibility = ["//visibility:public"],
    deps = [
        "@npm//@apollo/server",
        "@npm//@graphql-tools/merge",
        "@npm//body-parser",
        "@npm//compression",
        "@npm//cookie-parser",
        "@npm//dotenv",
        "@npm//exceljs",
        "@npm//express",
        "@npm//express-fileupload",
        "@npm//express-joi-validation",
        "@npm//express-session",
        "@npm//graphql",
        "@npm//graphql-tag",
        "@npm//helmet",
        "@npm//joi",
        "@npm//jsonwebtoken",
        "@npm//knex",
        "@npm//lodash",
        "@npm//minio",
        "@npm//morgan",
        "@npm//pg",
        "@npm//pg-boss",
        "@npm//redis",
        "@npm//socket.io",
        "@npm//winston",

        "@npm//@types/express",
        "@npm//@types/express-fileupload",
        "@npm//@types/express-session",
        "@npm//@types/jest",
        "@npm//@types/node",
    ]
)
```

现在可以编译了：

```bash
yarn bazel build //... 
```

最后可以在`dist/bin`看到一个`server.js`, 我们可以通过执行`node dist/bin/server.js`运行我们的nodejs项目了。


## 参考

1. [Bazel官网](https://bazel.build)
2. [Bazel 学习笔记 (一) 快速开始](https://zhuanlan.zhihu.com/p/411563404)
3. [使用Bazel编译TypeScript](https://zhuanlan.zhihu.com/p/402311730)
4. [angular](https://github.com/angular/angular)
5. [feat: add bazel support](https://github.com/damingerdai/express-postgres-ts-starter/pull/462)