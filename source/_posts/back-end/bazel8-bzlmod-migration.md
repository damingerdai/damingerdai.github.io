---
title: Bazel 8 Bzlmod 迁移记录：从 WORKSPACE 到 MODULE.bazel
date: 2025-11-03 20:20:10
tags: [spring boot, bazel, bzlmod]
categories: [后端]
---

# Bazel 8 Bzlmod 迁移记录：从 WORKSPACE 到 MODULE.bazel

本文档记录了项目 `hoteler` 从 Bazel 旧版 `WORKSPACE` 机制迁移到 Bazel 8 Bzlmod（Bazel 模块系统）的完整过程，重点记录了在迁移 Java 和 Maven 依赖时遇到的关键问题及其解决方案。

## 1\. 迁移背景与目标

  * **原 Bazel 版本:** Bazel 7.x
  * **目标 Bazel 版本:** Bazel 8.4.2 
  * **目标:** 完全移除 `WORKSPACE` 文件，将所有外部依赖（包括规则集和 Maven Artifacts）迁移到 `MODULE.bazel` 文件中进行管理。
  * **涉及的关键规则集:** `rules_java`、`rules_jvm_external`、`rules_spring`。

## 2\. 原始 WORKSPACE 结构概述

项目原先使用传统的 `WORKSPACE` 文件管理依赖，主要依赖 `http_archive` 引入规则集，并使用 `rules_jvm_external` 的 `maven_install` 函数定义所有 Maven 依赖。

```python
# WORKSPACE (简化结构)
workspace(name = "hoteler")
# 通过 http_archive 引入 rules_jvm_external 和 rules_spring
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
http_archive(...) # rules_jvm_external
http_archive(...) # rules_spring
# 使用 maven_install 定义所有 Maven 依赖，并将其暴露为 @maven 代码库
load("@rules_jvm_external//:defs.bzl", "maven_install")
maven_install(
    artifacts = [...], # 包含 Spring Boot, Mybatis, JWT 等所有依赖
    maven_install_json = "@//:maven_install.json",
)
load("@maven//:defs.bzl", "pinned_maven_install")
pinned_maven_install()
```

## 3\. Bzlmod 迁移与问题记录

迁移过程主要集中在 `MODULE.bazel` 文件的创建和调整上，遇到了 Bazel 8 Bzlmod 机制的多个严格限制。

### 3.1 遇到的第一个问题：`load` 语句限制

**问题描述:**
尝试在 `MODULE.bazel` 文件中直接使用 `load` 语句引入 `rules_jvm_external` 扩展和本地 `.bzl` 文件时，Bazel 报错 `load statements may not be used in MODULE.bazel files`。

**原因分析:**
`MODULE.bazel` 文件只允许使用声明性语句如 `module()`, `bazel_dep()`, 和 `use_extension()`。任何形式的 `load` 语句（包括加载模块扩展）都必须被重写为 `use_extension` 模式。

**初步解决方案 (失败)：**
最初尝试将 `load` 用于模块扩展也失败，因为它与 `use_extension` 的语法要求不符。

### 3.2 遇到的第二个问题：核心依赖缺失 (`rules_java`)

**问题描述:**
在成功引入 `rules_jvm_external` 后，构建失败，报错 `No repository visible as '@rules_java' from main repository`。

**原因分析:**
在 Bzlmod 中，即使是 Java 项目的核心规则集 `rules_java` 也需要被显式声明。`rules_jvm_external` 和 `rules_spring` 都依赖于它。

**解决方案:**
在 `MODULE.bazel` 中显式添加 `rules_java` 依赖，并根据最终构建需求将其版本设置为 `8.16.1`。

```python
bazel_dep(
    name = "rules_java",
    version = "8.16.1", 
)
```

### 3.3 遇到的第三个问题：`@maven` 仓库可见性问题

**问题描述:**
在 `BUILD.bazel` 文件中引用 Maven 依赖时，持续收到错误 `No repository visible as '@maven' from main repository` 或 `No repository visible as '@unpinned_maven' from main repository`。

**原因分析:**
在 Bzlmod 中，模块扩展（如 `rules_jvm_external` 的 `maven` 扩展）创建的代码库（如 `@maven`）不会自动对主模块可见。需要一个额外的步骤来将该代码库导入到主模块的命名空间。

**解决方案 (最终):**
使用 `use_repo` 函数将模块扩展生成的 `maven` 仓库导入。这是 `rules_jvm_external` 在 Bazel 8 中推荐的 Bzlmod 导入方式:

```python
# MODULE.bazel (最终结构的关键变动)
maven = use_extension("@rules_jvm_external//:extensions.bzl", "maven")

# 使用 use_repo 将扩展生成的 'maven' 仓库导入主模块
use_repo(maven, "maven")

maven.install(
    lock_file = "//:maven_install.json", # 修正了参数名
    artifacts = [...],
    # ...
)
```

## 4\. 最终 Bzlmod 配置与 BUILD 文件

以下是完成迁移后项目的最终依赖管理文件。

### 4.1 最终的 MODULE.bazel 文件

该文件集成了所有规则集依赖、版本修正和 Maven 扩展配置。

```python
module(
    name = "hoteler",
    compatibility_level = 1,
)

bazel_dep(
    name = "rules_java",
    version = "8.16.1", 
)

bazel_dep(
    name = "rules_jvm_external",
    version = "6.7", # 解决版本冲突
)

bazel_dep(
    name = "rules_spring",
    version = "2.6.3",
)

# 使用 use_extension 激活 Maven 扩展
maven = use_extension("@rules_jvm_external//:extensions.bzl", "maven")

# 导入 Maven 仓库，使其在 BUILD 文件中可见为 @maven
use_repo(maven, "maven")

# 配置 Maven 依赖
maven.install(
    lock_file = "//:maven_install.json",
    artifacts = [
        "org.springframework.boot:spring-boot-starter-web:3.5.5",
        # ... (所有 Maven 依赖项，包括补充的 cn.hutool:hutool-crypto 等)
    ],
    fetch_sources = True,
    repositories = [
        "https://maven.aliyun.com/repository/central",
        # ... (所有仓库)
    ],
)
```

### 4.2 最终的 BUILD.bazel 引用

在 `BUILD.bazel` 文件中，所有对 Maven 依赖的引用都保持使用 `@maven//:...` 别名，因为 `MODULE.bazel` 文件已通过 `use_repo(maven, "maven")` 语句确保了 `maven` 代码库的可见性。

```python
# BUILD.bazel (部分内容)
java_library(
    name = "java-maven-lib",
    srcs = glob(["src/main/java/org/daming/hoteler/*.java"]),
    deps = [
        "@maven//:org_springframework_boot_spring_boot_starter_web",
        "@maven//:io_jsonwebtoken_jjwt_api",
        # ... (所有其他 @maven 引用)
    ],
    # ...
)
```

## 5\. 结论

通过将规则集定义从 `http_archive` 迁移到 `bazel_dep`，并将 `maven_install` 逻辑封装到 `use_extension` 和 `use_repo` 组合中，项目成功实现了到 Bazel 8 Bzlmod 的完全迁移，并移除了 `WORKSPACE` 文件。迁移的关键在于理解 Bazel 8 对依赖导入和名称空间可见性的严格要求。