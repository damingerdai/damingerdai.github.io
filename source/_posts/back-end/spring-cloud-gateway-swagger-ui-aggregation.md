---
title: Spring Cloud Gateway 聚合 Swagger UI：实现微服务统一 API 文档入口
date: 2026-01-07 22:37:22
tags: [spring cloud, spring cloud gateway, open api]
categories: [后端]
---

# 🚀 前言

在微服务架构中，随着服务数量的增加，每个业务服务（如 `auth`、`workflow`）都有自己独立的 Swagger UI 页面。对于开发者和测试人员来说，频繁切换不同的 URL 和端口来查看 API 文档非常低效。

为了解决这个问题，我在我的开源项目 [hoteler-cloud](https://github.com/damingerdai/hoteler-cloud) 中，利用 **Spring Cloud Gateway** 和 **Springdoc OpenAPI** 实现了 Swagger UI 的聚合。现在，只需要访问网关服务（orchestration）的一个地址，即可通过下拉菜单切换所有后端服务的 API 文档。

# 💡 核心思路

1.  **网关层引入 UI 依赖**：在网关服务 `orchestration` 中引入支持 WebFlux 的 Springdoc UI 包。
2.  **配置聚合路由**：在 `application.yml` 中配置 `springdoc.swagger-ui.urls`，手动指定各微服务 OpenAPI 定义文件的路径。
3.  **网关转发逻辑**：利用网关现有的路由机制，确保能够正确转发对各服务 `/v3/api-docs` 接口的请求。

# 🛠️ 具体实现步骤

## 1. 添加依赖

在 `orchestration` (Gateway) 服务的 `pom.xml` 中引入 `springdoc-openapi-starter-webflux-ui`。

> **注意**：由于 Spring Cloud Gateway 是基于 WebFlux 构建的，必须使用对应的 WebFlux Starter。

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
    <version>${springdoc.version}</version>
</dependency>

```

## 2. 配置 application.yml

这是最关键的一步。我们需要在网关的配置文件中告诉 Swagger UI 去哪里拉取其他服务的 API 定义（JSON 格式）。

```yaml
springdoc:
  swagger-ui:
    # 启用聚合功能并定义下拉菜单
    urls:
      - name: 认证服务 (Auth Service)
        url: /auth/v3/api-docs
      - name: 工作流服务 (Workflow Service)
        url: /workflow/v3/api-docs
    # 自定义 Swagger UI 的路径
    path: /swagger-ui.html

spring:
  cloud:
    gateway:
      routes:
        - id: auth-service
          uri: lb://auth
          predicates:
            - Path=/auth/**
          filters:
            - StripPrefix=1
        - id: workflow-service
          uri: lb://workflow
          predicates:
            - Path=/workflow/**
          filters:
            - StripPrefix=1

```

**配置说明：**

* `urls` 列表中的 `url` 属性必须能够被网关路由规则匹配。
* 例如，访问 `/auth/v3/api-docs` 时，网关会根据 `auth-service` 的路由规则将其转发到真正的 `auth` 服务。

## 3. 微服务侧配置

确保 `auth` 和 `workflow` 服务本身也引入了 Springdoc 依赖，并且没有禁用 `api-docs` 路径。


# ✅ 效果展示

现在，当我们启动所有服务并访问网关地址：
`http://localhost:8080/swagger-ui.html`

1. 页面右上角会出现一个 **Select a definition** 的下拉选框。
2. 下拉菜单中会显示“认证服务”和“工作流服务”。
3. 切换选项后，页面会动态加载对应服务的 API 列表，并可以直接进行接口调试。


# ⚠️ 避坑指南

* **跨域问题 (CORS)**：如果网关和子服务不在同一个域，可能会遇到跨域问题。建议在网关层通过 Spring Cloud Gateway 的配置统一处理 CORS。
* **路径匹配**：确保 `springdoc.swagger-ui.urls` 里的路径与 Gateway 的 `routes` 配置能够对应上，否则会出现 404 无法加载 JSON 的情况。
* **版本兼容**：Spring Boot 3.x 务必使用 `springdoc-openapi-starter-webflux-ui` 而不是旧版的 `springdoc-openapi-ui`。


# 📝 总结

通过在网关层聚合 Swagger UI，我们不仅提升了开发体验，还统一了系统的入口。这也是构建企业级微服务架构中“开发者友好型”基础设施的重要一环。

如果你对实现细节感兴趣，欢迎查看我的 PR 详情：
🔗 [PR #149: Feat: add swagger ui for orchestration](https://github.com/damingerdai/hoteler-cloud/pull/149)

**项目地址：** [damingerdai/hoteler-cloud](https://www.google.com/url?sa=E&source=gmail&q=https://github.com/damingerdai/hoteler-cloud)
如果你觉得这个项目对你有帮助，欢迎点个 **Star** ⭐！


