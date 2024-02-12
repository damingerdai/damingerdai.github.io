---
title: 使用bazel构建spring boot项目
date: 2024-02-11 21:23:35
tags: [bazel, java, spring boot]
categories: [后端]
---

# 前言

根据官网的定义，Bazel是类似于Make，Maven和Gradle的开源构建和测试工具。它使用人类可读的高级构建语言[Starlark](https://github.com/bazelbuild/starlark)(一种基于python的方言)。 Bazel支持多种语言的项目，并为多种平台构建输出。 

从我个人角度来看，bazel是一个强大且复杂的构建系统，通过`build rule`的概念，支持多种语言、不同平台，支持构建C/C++,Java,Android,IOS,Golang,Nodejs,Docker项目

本文的目的是使用bazel去构建并运行一个spring boot项目。

# 配置bazel编译java项目

在项目根目录中创建`.bazelrc`文件，设置bazel使用java17构建：

```bash
build --java_language_version=17 --java_runtime_version=17 --tool_java_language_version=17 --tool_java_runtime_version=17
test  --java_language_version=17 --java_runtime_version=17 --tool_java_language_version=17 --tool_java_runtime_version=17
```

在根目录中创建`workspace`文件，并引入相关的java依赖：

```bazel
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

RULES_JVM_EXTERNAL_TAG = "6.0"
RULES_JVM_EXTERNAL_SHA = "c44568854d8bb92fe0f7dd6b1e8957ae65e45e32a058727fcf62aaafbd36f17b"

http_archive(
    name = "rules_jvm_external",
    strip_prefix = "rules_jvm_external-%s" % RULES_JVM_EXTERNAL_TAG,
    sha256 = RULES_JVM_EXTERNAL_SHA,
    urls = [
        "https://github.com/bazelbuild/rules_jvm_external/archive/%s.zip" % RULES_JVM_EXTERNAL_TAG,
        "https://mirror.ghproxy.com/https://github.com/bazelbuild/rules_jvm_external/archive/%s.zip" % RULES_JVM_EXTERNAL_TAG,
    ]
   
)

load("@rules_jvm_external//:repositories.bzl", "rules_jvm_external_deps")

rules_jvm_external_deps()

load("@rules_jvm_external//:setup.bzl", "rules_jvm_external_setup")

# rules_jvm_external_setup()

load("@rules_jvm_external//:specs.bzl", "maven")
load("@rules_jvm_external//:defs.bzl", "maven_install")
```

然后使用使用`maven_install`导入spring boot项目的相关依赖。
这里的shiro依赖被我单独拎出来做特殊处理去兼容java17

```bazel
shiros = [
    maven.artifact(
        group = "org.apache.shiro",
        artifact = "shiro-spring",
        version = "1.13.0",
        classifier = "jakarta",
        exclusions = [
            maven.exclusion(
                  group = "org.apache.shiro",
                  artifact = "shiro-core",
            ),
            maven.exclusion(
                  group = "org.apache.shiro",
                  artifact = "shiro-web",
            ),
        ]
    ),
    maven.artifact(
        group = "org.apache.shiro",
        artifact = "shiro-core",
        version = "1.13.0",
        classifier = "jakarta",
    ),
    maven.artifact(
        group = "org.apache.shiro",
        artifact = "shiro-web",
        version = "1.13.0",
        classifier = "jakarta",
    ),
   
]

maven_install(
    artifacts = [
        "org.springframework.boot:spring-boot:3.2.2",
        "org.springframework.boot:spring-boot-starter:3.2.2",
        "org.springframework.boot:spring-boot-loader-tools:3.2.2",
        "org.springframework.boot:spring-boot-loader:3.2.2",
        "org.springframework.boot:spring-boot-starter:3.2.2",
        "org.springframework.boot:spring-boot-starter-web:3.2.2",
        "org.springframework.boot:spring-boot-starter-jdbc:3.2.2",
        "org.springframework.boot:spring-boot-starter-quartz:3.2.2",
        "org.springframework.boot:spring-boot-starter-aop:3.2.2",

        #"org.springframework.boot:spring-boot-configuration-processor:3.2.2",

        "io.springfox:springfox-boot-starter:3.0.0",
        "com.auth0:java-jwt:3.19.4",
        "org.postgresql:postgresql:42.4.0",

        "jakarta.servlet:jakarta.servlet-api:6.0.0",  
        'javax.annotation:javax.annotation-api:1.3.2',

        "org.springframework.boot:spring-boot-devtools:3.2.2",
        "org.springframework.boot:spring-boot-starter-test:3.2.2"
         
    ] + shiros,
    fetch_sources = True,
    repositories = [
        "https://maven.aliyun.com/repository/public/",
        "https://maven.aliyun.com/nexus/content/groups/public/",
        "http://uk.maven.org/maven2",
        "https://maven.google.com",
        "https://repo1.maven.org/maven2",
    ],
    # maven_install_json = "//:maven_install.json",
)
```

`maven_install_json = "//:maven_install.json"`目前是被注视掉的，这是因为我们需要自动生成改文件：

```bash
bazel run @maven//:pin 
```

执行成功之后需要在`workspace`中给`maven_install`添加`maven_install_json`属性, 并将加载`@maven//:defs.bzl`中的`pinned_maven_install`配置

```
maven_install(
    artifacts = # ...,
    repositories = # ...,
    maven_install_json = "@//:maven_install.json",
)

load("@maven//:defs.bzl", "pinned_maven_install")
pinned_maven_install()
```

在更新maven依赖的话，那么可以使用下面的依赖更新`maven_install.json`

```bazel
bazel run @unpinned_maven//:pin
```

`maven_install_json`是可以不用配置的，但是我推荐尽量配置。它有两个好处，可重复性和速度（reproducibility and speed）。

在根目录创建`BUILD.bazel`进行编译java项目的准备：

```bazel
oad("@rules_java//java:defs.bzl", "java_binary", "java_library", "java_test")

package(default_visibility = ["//visibility:public"])

java_library(
    name = "jobs-lib",
    srcs = glob([
        "src/main/java/org/daming/jobs/*.java",
        "src/main/java/org/daming/jobs/api/advice/*.java",
        "src/main/java/org/daming/jobs/api/controller/*.java",
        "src/main/java/org/daming/jobs/api/interceptor/*.java",
        "src/main/java/org/daming/jobs/base/*.java",
        "src/main/java/org/daming/jobs/base/**/*.java",
        "src/main/java/org/daming/jobs/config/**/*.java",
        "src/main/java/org/daming/jobs/pojo/**/*.java",
        "src/main/java/org/daming/jobs/security/**/*.java",
        "src/main/java/org/daming/jobs/service/**/*.java",
        "src/main/java/org/daming/jobs/task/**/*.java",
    ]),
    resources = glob(["src/main/resources/**"]),
    deps = [
        "@maven//:org_springframework_boot_spring_boot_starter_web",
        "@maven//:org_springframework_boot_spring_boot_starter_jdbc",
        "@maven//:org_springframework_boot_spring_boot_starter_quartz",
        "@maven//:org_springframework_boot_spring_boot_starter_aop",

        "@maven//:io_springfox_springfox_boot_starter",
        "@maven//:io_springfox_springfox_core",
        "@maven//:io_springfox_springfox_spi",
        "@maven//:io_springfox_springfox_oas",
        "@maven//:io_springfox_springfox_spring_web",
        "@maven//:io_swagger_swagger_annotations",

        "@maven//:com_auth0_java_jwt",
        "@maven//:org_postgresql_postgresql",
        "@maven//:org_springframework_boot_spring_boot_devtools",

        "@maven//:org_springframework_boot_spring_boot",
        "@maven//:org_springframework_boot_spring_boot_loader",
        "@maven//:org_springframework_boot_spring_boot_loader_tools",
        "@maven//:org_springframework_boot_spring_boot_autoconfigure",
        "@maven//:org_springframework_spring_aop",
        "@maven//:org_springframework_spring_beans",
        "@maven//:org_springframework_spring_core",
        "@maven//:org_springframework_spring_context",
        "@maven//:org_springframework_spring_expression",
        "@maven//:org_springframework_spring_web",

        # "@maven//:org_apache_shiro_shiro_spring_boot_starter",
        "@maven//:org_apache_shiro_shiro_core_jakarta",
        "@maven//:org_apache_shiro_shiro_spring_jakarta",
        "@maven//:org_apache_shiro_shiro_web_jakarta",

        "@maven//:org_slf4j_slf4j_api",

        "@maven//:org_quartz_scheduler_quartz",
        "@maven//:org_aspectj_aspectjweaver",
        "@maven//:org_apache_tomcat_embed_tomcat_embed_core",
        "@maven//:jakarta_servlet_jakarta_servlet_api",  
        "@maven//:jakarta_annotation_jakarta_annotation_api",
        "@maven//:jakarta_xml_bind_jakarta_xml_bind_api",
        "@maven//:com_fasterxml_jackson_core_jackson_core",
        "@maven//:com_fasterxml_jackson_core_jackson_databind"
    ],
)

java_binary(
    name = "jobs",
    main_class = "org.daming.jobs.JobsApplication",
    runtime_deps = [":jobs-lib"],
    deploy_manifest_lines = {
        "Main-Class": "org.daming.jobs.JobsApplication",
    },
)
```

然后我们可以执行bazel命令去构建一个java项目了:

```bazel
# 构建
bazel build //:jobs

# 构建并运行
bazel run //:jobs
```

然后你会发现构建没有问题，但是运行会报错:

```bash
jobs git:(master) ✗ java -jar bazel-bin/jobs.jar            
no main manifest attribute, in bazel-bin/jobs.jar
```

由于Springboot的代码需要使用Springboot loader进行启动，Springboot程序的打包逻辑与普通的Java程序不同。这意味着，Bazel原生的 java_binary 无法正常启动Springboot程序。

所以我们需要给bazel配置spring相关支持。


# 配置rule_spring去构建运行spring boot


我们使用[salesforce的rules_spring](https://github.com/salesforce/rules_spring)定义好了的rule去帮助我们构建spring boot项目。

我们在`workspace`添加`rules_spring`的规则文件

```
http_archive(
    name = "rules_spring",
    sha256 = "7bb891ccb2f53ca188a769b3a3777be1c38348e18091afea05320f3003b3e886",
    urls = [
        "https://github.com/salesforce/rules_spring/releases/download/2.3.1/rules-spring-2.3.1.zip",
        "https://mirror.ghproxy.com/https://github.com/salesforce/rules_spring/releases/download/2.3.1/rules-spring-2.3.1.zip",
    ],
)
```

然后在`BUILD.bazel`中间中导入`rules_spring`的相关规则：

```bazel
load("@rules_spring//springboot:springboot.bzl", "springboot")

springboot(  
    name = "springboot",  
    # specify the main class
    boot_app_class = "org.daming.jobs.JobsApplication",
    # refrence the library
    java_library = ":jobs-lib",
    # https://github.com/salesforce/rules_spring/issues/177
    boot_launcher_class = 'org.springframework.boot.loader.launch.JarLauncher',
)
```

由于`spring boot 3.2.0`之后使用新的启动器，所以我们这里指定了`boot_launcher_class`。 如果你使用的版本低于3.2.0，可以直接删除。

现在我们可以编译运行spring boot项目了

```bazsl
# 构建
bazel build //:springboot

# 运行
bazel run //:springboot
```

运行部分输出如下：

```bazel

Application    Name: 
Application Version: 
Spring Boot Version: 3.2.2 (v3.2.2)
2024-02-11 21:49:17.449 INFO [main] org.springframework.boot.StartupInfoLogger:50 Starting JobsApplication using Java 17 with PID 67028 (/private/var/tmp/_bazel_gming001/7a71463ee80a3358d2f71ab2db616aea/execroot/__main__/bazel-out/darwin_arm64-fastbuild/bin/springboot.jar started by gming001 in /private/var/tmp/_bazel_gming001/7a71463ee80a3358d2f71ab2db616aea/execroot/__main__/bazel-out/darwin_arm64-fastbuild/bin/springboot.runfiles/__main__)
2024-02-11 21:49:17.451 INFO [main] org.springframework.boot.SpringApplication:654 No active profile set, falling back to 1 default profile: "default"
2024-02-11 21:49:17.501 INFO [main] org.springframework.boot.logging.DeferredLog:252 For additional web related logging consider setting the 'logging.level.web' property to 'DEBUG'
2024-02-11 21:49:17.846 WARN [main] org.springframework.context.support.AbstractApplicationContext:632 Exception encountered during context initialization - cancelling refresh attempt: java.lang.TypeNotPresentException: Type javax.servlet.http.HttpServletRequest not present
2024-02-11 21:49:17.856 INFO [main] org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLogger:82 

```

# 遗留的问题

有两个问题没有解决，第一个是如何运行的单元测试，这个还没研究，不过这个不急。还有一个是这个项目实际上跑不起来，会报错：

```log
java.lang.TypeNotPresentException: Type javax.servlet.http.HttpServletRequest not present
```

我原本以为是bazel的配置问题，结果我发现直接maven也跑不起来。。。

我想了想了应该是springfox没有去适配spring boot3的问题，不想降级spring boot的版本话，只能[从springfox迁移到springdoc](https://www.cnblogs.com/xiao2/p/15621730.html)

# 源码

本文使用的源代码都可以在[jobs](https://github.com/damingerdai/jobs)看到。


# 参考资料

- [bazel简介](https://bazel.google.cn/about)
- [Add support for new Boot Loader in Spring Boot 3.2.0](https://github.com/salesforce/rules_spring/issues/177)
- [Bazel使用案例：构建Springboot工程](https://showme.codes/zh-cn/2024-01-19-bazel-springboot/#%e5%88%9b%e5%bb%bamaven%e5%b7%a5%e7%a8%8b%e7%bb%93%e6%9e%84)
- [在windows上构建angular项目 (上)](https://zhuanlan.zhihu.com/p/372134347)
- [spring boot 3.0+](https://github.com/apache/shiro/issues/891)
- [从springfox迁移到springdoc](https://www.cnblogs.com/xiao2/p/15621730.html)
- [New Java Project With Bazel](https://junctionbox.ca/2020/06/19/bazel-java-starter.html)

