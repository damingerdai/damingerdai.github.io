---
title: Gradle 强制使用指定版本的 Spring Web JAR
date: 2025-09-01 20:10:01
tags: [gradle, spring boot]
categories: [后端]
---

# Gradle 强制使用指定版本的 Spring Web JAR

## 场景说明

由于公司Artifactory已禁止下载Spring Web 5.x相关依赖，必须通过本地JAR包spring-web-5.3.31.jar，强制让所有依赖Spring Web的其他库使用这个指定版本。

## 方法一：使用强制版本解析

这种方法通过 Gradle 的强制版本功能，强制所有依赖使用指定版本。

### Gradle 配置

在 `build.gradle` 文件中添加以下配置：

```groovy
configurations.all {
    // 强制所有 spring-web 依赖使用 5.3.31 版本
    resolutionStrategy {
        force 'org.springframework:spring-web:5.3.31'
    }
}

dependencies {
    // 添加本地 JAR 文件
    implementation files('libs/spring-web-5.3.31.jar')
    
    // 其他依赖...
    implementation 'org.springframework.boot:spring-boot-starter-web:2.7.0'
    implementation 'org.springframework.security:spring-security-web:5.7.1'
}
```

### 依赖解析过程

```
spring-boot-starter-web:2.7.0
└── spring-web:5.3.18 (原始依赖)
└── spring-web:5.3.31 (强制使用)

spring-security-web:5.7.1
└── spring-web:5.3.20 (原始依赖)
└── spring-web:5.3.31 (强制使用)
```

### 优点
- 配置简单，一行代码解决问题
- 适用于所有配置（implementation, testImplementation等）
- Gradle 自动处理版本冲突

### 缺点
- 如果本地 JAR 与强制版本不匹配，可能会导致运行时错误
- 需要确保所有依赖与强制版本兼容

## 方法二：使用依赖替换

这种方法使用 Gradle 的依赖替换功能，将远程依赖替换为本地 JAR 文件。

### Gradle 配置

在 `build.gradle` 文件中添加以下配置：

```groovy
configurations.all {
    resolutionStrategy.dependencySubstitution {
        // 将所有 spring-web 依赖替换为本地文件
        substitute module('org.springframework:spring-web') 
            using files('libs/spring-web-5.3.31.jar')
    }
}

dependencies {
    // 其他依赖...
    implementation 'org.springframework.boot:spring-boot-starter-web:2.7.0'
    implementation 'org.springframework.security:spring-security-web:5.7.1'
}
```

### 依赖解析过程

```
spring-boot-starter-web:2.7.0
└── spring-web:5.3.18 (原始依赖)
└── libs/spring-web-5.3.31.jar (替换为本地文件)

spring-security-web:5.7.1
└── spring-web:5.3.20 (原始依赖)
└── libs/spring-web-5.3.31.jar (替换为本地文件)
```

### 优点
- 精确控制依赖来源
- 可以针对特定依赖进行替换
- 适用于复杂的依赖关系

### 缺点
- 配置相对复杂
- 需要确保替换的依赖接口兼容

## 方法三：排除并显式引入

这种方法从所有依赖中排除 Spring Web，然后显式引入本地 JAR 文件。

### Gradle 配置

在 `build.gradle` 文件中添加以下配置：

```groovy
configurations.all {
    // 全局排除所有 spring-web 依赖
    exclude group: 'org.springframework', module: 'spring-web'
}

dependencies {
    // 添加本地 JAR 文件
    implementation files('libs/spring-web-5.3.31.jar')
    
    // 其他依赖，使用 exclude 确保不传递 spring-web
    implementation ('org.springframework.boot:spring-boot-starter-web:2.7.0') {
        exclude group: 'org.springframework', module: 'spring-web'
    }
    implementation ('org.springframework.security:spring-security-web:5.7.1') {
        exclude group: 'org.springframework', module: 'spring-web'
    }
}
```

### 依赖解析过程

```
spring-boot-starter-web:2.7.0
└── spring-web:5.3.18 (已被排除)

spring-security-web:5.7.1
└── spring-web:5.3.20 (已被排除)

本地引入: libs/spring-web-5.3.31.jar
```

### 优点
- 完全控制依赖树
- 确保只使用本地 JAR
- 避免任何版本冲突

### 缺点
- 配置最复杂，需要为每个依赖添加排除规则
- 维护成本较高

## 推荐使用方法

**推荐使用方法一**：强制版本解析是最简单有效的方法，适用于大多数场景。只需确保您的本地 JAR 文件确实是 5.3.31 版本。

## 验证配置是否生效

添加以下任务到 `build.gradle` 来验证依赖解析结果：

```groovy
task checkDependencies {
    doLast {
        configurations.compileClasspath.resolvedConfiguration.resolvedArtifacts.each { artifact ->
            println "依赖: ${artifact.moduleVersion.id} -> 文件: ${artifact.file.name}"
        }
        
        // 特别检查 spring-web
        def springWebDeps = configurations.compileClasspath.incoming.dependencies
            .findAll { it.name == 'spring-web' }
        
        println "Spring Web 依赖:"
        springWebDeps.each { dep ->
            println "  - ${dep.group}:${dep.name}:${dep.version}"
        }
    }
}
```

运行 `./gradlew checkDependencies` 查看所有依赖及其解析结果。

## 关键要点

- **方法一（强制版本解析）**是最简单直接的方法，推荐大多数场景使用
- **方法二（依赖替换）**提供了更精确的控制，适用于复杂场景
- **方法三（排除并显式引入）**是最彻底的方法，但配置和维护成本较高
- 无论使用哪种方法，都需要确保本地JAR与项目其他依赖兼容

---

**项目结构参考**：
```
项目目录/
├── libs/
│   ├── spring-web-5.3.31.jar
│   └── 其他库文件.jar
├── build.gradle
├── settings.gradle
└── src/
```

通过以上配置，您可以确保所有依赖 Spring Web 的库都使用您指定的 5.3.31 版本本地 JAR 文件。
