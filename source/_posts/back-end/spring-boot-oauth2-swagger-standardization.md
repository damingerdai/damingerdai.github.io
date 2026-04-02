---
title: 技术复盘：基于 OAuth 2.0 规范的 Spring Boot 鉴权标准化改造
date: 2026-04-02 12:47:19
tags: [spring boot, swagger ui, oauth2, spring doc]
categories: [后端]
---

# 现状与需求

在 Hoteler 项目的开发测试阶段，由于原有的登录接口 /token 采用非标设计（自定义 JSON 结构、Header 传参），导致 Swagger UI (SpringDoc) 无法识别认证状态，必须手动复制 Token 并粘贴至 Authorize 框中，测试效率低。

## 目标：
实现符合 OAuth 2.0 Password Grant 规范的接口，对接 Swagger UI 自动鉴权流。

# 协议参考
本次改造主要参考了以下关于 OAuth 2.0 协议实现的标准逻辑：

1. [理解OAuth 2.0](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
2. [OAuth 2.0 的一个简单解释](https://www.ruanyifeng.com/blog/2019/04/oauth_design.html)
3. [OAuth 2.0 的四种方式](https://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html)
4. [GitHub OAuth 第三方登录示例教程](https://www.ruanyifeng.com/blog/2019/04/github-oauth.html)

# 技术实现方案

## 响应报文的标准化（Schema Mapping）

OAuth 2.0 规定 Access Token Response 的根路径必须包含 access_token。为兼容原有业务对象 UserToken，采用接口注入方式扩展响应体：

```java
public interface OAuth2Compatible {
    @JsonProperty("access_token")
    String getAccessToken(); // 映射至业务 Token 字段

    @JsonProperty("token_type")
    default String getTokenType() { return "bearer"; }
    
    @JsonProperty("expires_in")
    default long getExpiresIn() { return 2592000; }
}
```

通过 UserTokenResponse implements OAuth2Compatible，在不修改原有 userToken 嵌套对象的前提下，输出了符合 RFC 6749 要求的 JSON 字段


## 登录接口参数兼容

修改 /token 接口，支持 application/x-www-form-urlencoded 消费类型，以接收 Swagger UI 默认发送的 username 和 password 表单参数。

```java
@Operation(summary = "创建token")
@PostMapping(value = "token", consumes = { MediaType.APPLICATION_FORM_URLENCODED_VALUE, MediaType.APPLICATION_JSON_VALUE })
public UserTokenResponse createToken(@RequestParam(value = "username", required = false) String formUser,
                                        @RequestParam(value = "password", required = false) String formPass,
                                        @RequestHeader(value = "username", required = false) String headUser,
                                        @RequestHeader(value = "password", required = false) String headPass,
                                        HttpSession session) {
    var username = (headUser != null) ? headUser : formUser;
    var password = (headPass != null) ? headPass : formPass;
    try {
        var unauthenticated = UsernamePasswordAuthenticationToken.unauthenticated(username, password);
        var authenticate = authenticationManager.authenticate(unauthenticated);
        var userToken = tokenService.createToken(username, password);
        SecurityContextHolder.getContext().setAuthentication(authenticate);
        session.setAttribute(HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY, SecurityContextHolder.getContext());
        var response = new UserTokenResponse();
        response.setUserToken(userToken);
        return response;
    } catch (AuthenticationException ex) {
            LoggerManager.getApiLogger().error("AuthenticationException: " + ex.getMessage());
            throw ex;
    } catch (HotelerException ex) {
        LoggerManager.getApiLogger().error("HotelerException: " + ex.getMessage());
        throw ex;
    } catch (Exception ex) {
        LoggerManager.getApiLogger().error("Exception: " + ex.getMessage());
        throw ex;
    } finally {
        // TODO
    }

}
```

## OpenAPI SecurityScheme 配置

在 OpenApiConfig 中定义 SecurityScheme。关键点在于 type 声明为 OAUTH2，并正确指向 tokenUrl。

```java
private Components components() {
    return new Components()
            .addSecuritySchemes("bearer-key", new SecurityScheme().type(SecurityScheme.Type.HTTP).scheme("bearer").bearerFormat("JWT"))
            .addSecuritySchemes("OAuth2Password", new SecurityScheme()
                    .type(SecurityScheme.Type.OAUTH2)
                    .description("输入用户名和密码登录以获取令牌")
                    .flows(new OAuthFlows()
                            .password(new OAuthFlow()
                                    // 这里的 URL 必须对应你 Controller 里的 @PostMapping("token")
                                    .tokenUrl("/api/v1/token")
                                    .scopes(new Scopes().addString("read", "读权限"))
                            )
                    ));
    }
```

# 调试记录

## Header 填充失效问题

现象： 弹窗登录成功后，调用业务接口仍未自动携带 Authorization Header。
定位： 接口声明的 SecurityRequirement 名称与 SecurityScheme 的 Key 不一致。
解决： 统一使用 OAuth2Password 作为唯一标识符或者直接删除原有的SecurityScheme 的 Key。

```java
@Operation(security = { @SecurityRequirement(name = "OAuth2Password") })
```

# 结论

通过本次改造，后端认证模块具备了以下特性：

1. 协议对齐：完全兼容 OAuth 2.0 密码模式。
2. 前端自动化：Swagger UI 可实现“登录-存储-填充”闭环。
3. 零破坏升级：原有基于 Header 传参的旧客户端调用不受影响。

项目仓库： [damingerdai/hoteler](https://github.com/damingerdai/hoteler)
