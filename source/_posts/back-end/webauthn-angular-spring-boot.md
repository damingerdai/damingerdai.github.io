---
title: 'WebAuthn 入门：从无密码认证原理到 Angular + Spring Boot 实战'
date: 2026-11-09 23:25:00
tags:
  - WebAuthn
  - Passkey
  - Angular
  - Spring Boot
  - Java
categories:
  - 技术实践
keywords:
  - WebAuthn
  - Passkey
  - FIDO2
  - Angular
  - Spring Boot
description: '从 WebAuthn 公钥密码学、Challenge、RP ID 和 Origin 入手，使用 Angular、SimpleWebAuthn、Spring Boot 与 WebAuthn4J 实现 Passkey 注册和登录。'
---

传统密码登录存在密码复用、凭证泄露和钓鱼攻击等问题。Passkey 则让我们通过指纹、面容识别或设备 PIN 完成登录，而不需要向服务器提交密码。

这篇文章分成两部分：先理解 **WebAuthn 是什么、为什么安全**，再结合我在 Batcher 项目中的实现，介绍 **Angular + Spring Boot 如何完成 Passkey 注册和登录**。


# 第一部分：WebAuthn 理论

## 1. WebAuthn 是什么？

WebAuthn（Web Authentication）是 W3C 制定的 Web 身份认证标准，也是 FIDO2 体系的一部分。它允许网站通过浏览器调用认证器（Authenticator），使用**公钥密码学**注册和验证身份。

认证器可以是 Mac 的 Touch ID、iPhone 的 Face ID、Windows Hello、硬件安全密钥，或支持 Passkey 的密码管理器。

传统密码认证依赖服务器验证用户提交的秘密；WebAuthn 则依赖认证器使用私钥签名，服务器使用此前登记的公钥验证签名。

```text
传统密码：用户输入密码 → 服务器验证密码
WebAuthn：服务器发送挑战 → 认证器用私钥签名 → 服务器用公钥验证
```

## 2. WebAuthn 与 Passkey 的关系

- **WebAuthn**：浏览器与服务端进行公钥注册和认证的标准/API。
- **FIDO2**：包含 WebAuthn 与 CTAP 等协议的技术体系。
- **Authenticator**：负责创建或使用凭证密钥的认证器。
- **Passkey**：面向用户的 FIDO 无密码登录凭证及体验，可以是设备绑定的，也可以通过支持的密码管理器同步。
- **Relying Party（RP）**：使用 WebAuthn 进行身份认证的网站。

注册时，浏览器可能让用户选择 Apple Passwords、Microsoft Password Manager、手机或安全密钥等位置。选择不同的凭证提供者，并不改变服务器验证签名的基本机制。

## 3. 公钥和私钥如何工作？

注册时，认证器创建一组密钥：

```text
认证器创建密钥对
├── 私钥：由认证器或凭证提供者保护，不交给网站服务器
└── 公钥：注册成功后由服务器保存
```

登录时，服务器发送随机 Challenge，认证器使用私钥签名，服务器用公钥验证。

```text
signature = Sign(privateKey, message)
valid = Verify(publicKey, message, signature)
```

这只是概念表达。WebAuthn 实际签名的数据是：

```text
authenticatorData || SHA256(clientDataJSON)
```

`clientDataJSON` 包含 Challenge、Origin、操作类型等信息；`authenticatorData` 包含 RP ID Hash、用户验证标志、签名计数器等数据。

**为什么攻击者不能只拿公钥登录？** 因为公钥可以验证签名，但不能直接产生与对应私钥等效的有效签名。服务器即使泄露公钥，也不等于泄露了用户的私钥。

## 4. Challenge 为什么必不可少？

如果每次登录都对固定内容签名，攻击者可能重放此前截获的有效认证响应。Challenge 是服务器为本次操作生成的一次性随机值，使每次认证需要签名的数据不同。

在 Demo 中使用 Java `SecureRandom` 生成 32 字节随机数：

```java
byte[] challenge = new byte[32];
secureRandom.nextBytes(challenge);
```

服务器还会保存操作类型、用户名、过期时间，并在验证时**一次性消费** Challenge。当前实现设置 5 分钟有效期。

## 5. RP ID 与 Origin

WebAuthn 的防钓鱼能力不仅来自签名，还来自浏览器和服务端对网站身份的校验。

- **RP ID** 通常是域名，Demo 中为 `localhost`。
- **Origin** 包括协议、主机和端口，Demo 中为 `http://localhost:4200`。

```java
private static final String RP_ID = "localhost";
private static final String ORIGIN = "http://localhost:4200";
```

Angular 页面运行在 `http://localhost:4200`，Spring Boot API 运行在 `http://localhost:8080`。尽管请求发送到后端，WebAuthn 客户端操作的 Origin 仍是 Angular 页面的来源。

**CORS 与 WebAuthn Origin 验证不同**：前者控制跨源 HTTP 访问，后者参与身份认证安全校验。正式部署通常需要 HTTPS；`localhost` 是允许本地开发的特殊情况。

## 6. 指纹和 Face ID 会发送给服务器吗？

不会。指纹、面容识别或 PIN 通常用于在本地授权认证器使用私钥。服务器收到的是公钥凭证注册数据或数字签名认证结果，而不是生物识别原始数据。

## 7. 注册与登录的完整流程

**注册（Registration）**：

```text
Angular                 Spring Boot                 Authenticator
   |  请求注册 Challenge     |                           |
   |----------------------->|                           |
   |  返回 CreationOptions   |                           |
   |<-----------------------|                           |
   |  startRegistration()                               |
   |--------------------------------------------------->|
   |                           创建密钥对、用户授权       |
   |<---------------------------------------------------|
   |  提交 Credential       |                           |
   |----------------------->|                           |
   |                   验证并保存公钥凭证                 |
   |<-----------------------|                           |
```

**登录（Authentication）**：

```text
Angular                 Spring Boot                 Authenticator
   |  请求登录 Challenge     |                           |
   |----------------------->|                           |
   |  返回 RequestOptions    |                           |
   |<-----------------------|                           |
   |  startAuthentication()                             |
   |--------------------------------------------------->|
   |                           用户授权、私钥签名         |
   |<---------------------------------------------------|
   |  提交 Assertion        |                           |
   |----------------------->|                           |
   |                   使用公钥验证签名                   |
   |<-----------------------|                           |
```

# 第二部分：Angular + Spring Boot 实践

## 8. 技术栈和项目结构

本次实现使用：Angular 22、Angular Material、`@simplewebauthn/browser` 14、Spring Boot、WebAuthn4J `0.32.0.RELEASE`，并暂时使用内存 DAO 保存 Challenge 和 Credential。

```text
com.github.damingerdai.batcher
├── api/WebAuthnController.java
├── config/WebAuthnConfig.java
├── dao/IWebAuthnDao.java
├── dao/impl/InMemoryWebAuthnDao.java
├── pojo/ChallengePurpose.java
├── pojo/PendingChallenge.java
├── pojo/StoredCredential.java
├── service/IWebAuthnService.java
└── service/impl/WebAuthnServiceImpl.java

ui/src/app
├── core/services/webauthn.service.ts
└── routes/passkey/
    ├── passkey.component.ts
    ├── passkey.component.html
    ├── passkey.component.scss
    └── passkey.routes.ts
```

Controller 负责 HTTP 请求；Service 负责注册、登录及协议验证；DAO 负责数据存储。这样的分层也方便以后接入 PostgreSQL 和 Redis。

## 9. Spring Boot 配置 WebAuthn4J

在 `build.gradle` 中引入：

```gradle
implementation 'com.webauthn4j:webauthn4j-core:0.32.0.RELEASE'
```

配置 `WebAuthnManager`：

```java
@Configuration
public class WebAuthnConfig {
    @Bean
    public WebAuthnManager webAuthnManager() {
        return WebAuthnManager.createNonStrictWebAuthnManager();
    }
}
```

这里的 `NonStrict` **不等于不验证签名**。它主要涉及注册时对认证器 Attestation 信任链的验证策略；普通 Passkey 应用通常不强制要求设备厂商证明。

## 10. Challenge 和 Credential 模型

`PendingChallenge` 保存用户、用途、随机挑战值和过期时间：

```java
public record PendingChallenge(
        String username,
        ChallengePurpose purpose,
        byte[] challenge,
        Instant expiresAt
) {
    public PendingChallenge {
        challenge = challenge.clone();
    }

    @Override
    public byte[] challenge() {
        return challenge.clone();
    }
}
```

`ChallengePurpose` 区分 `REGISTER` 和 `LOGIN`，避免混用挑战值。

```java
public enum ChallengePurpose {
    REGISTER,
    LOGIN
}
```

`StoredCredential` 保存用户、Credential ID 和 WebAuthn4J 的凭证记录：

```java
public record StoredCredential(
        String username,
        byte[] credentialId,
        CredentialRecord credentialRecord
) {
    public StoredCredential {
        credentialId = credentialId.clone();
    }

    @Override
    public byte[] credentialId() {
        return credentialId.clone();
    }
}
```

在 `InMemoryWebAuthnDao` 中，使用 `ConcurrentHashMap` 保存数据：

```java
private final Map<String, PendingChallenge> challenges =
        new ConcurrentHashMap<>();

private final Map<String, StoredCredential> credentials =
        new ConcurrentHashMap<>();

public Optional<PendingChallenge> consumeChallenge(String requestId) {
    return Optional.ofNullable(challenges.remove(requestId));
}
```

`remove()` 能原子地消费 Challenge；`putIfAbsent()` 可以防止同一个 Credential ID 被重复注册。应用重启后内存数据会丢失，因此这只适合 Demo。

## 11. 服务端生成注册选项

`WebAuthnServiceImpl` 先生成 Challenge：

```java
private String createChallenge(String username, ChallengePurpose purpose) {
    byte[] challenge = new byte[32];
    secureRandom.nextBytes(challenge);

    String requestId = UUID.randomUUID().toString();
    webAuthnDao.saveChallenge(
            requestId,
            new PendingChallenge(
                    username,
                    purpose,
                    challenge,
                    Instant.now().plusSeconds(300)
            )
    );
    return requestId;
}
```

然后返回注册选项，结构如下（省略了外层 Java `Map.of` 代码）：

```json
{
  "requestId": "generated-request-id",
  "publicKey": {
    "challenge": "base64url-challenge",
    "rp": { "id": "localhost", "name": "Passkey Demo" },
    "user": {
      "id": "base64url-user-id",
      "name": "arthur",
      "displayName": "arthur"
    },
    "pubKeyCredParams": [
      { "type": "public-key", "alg": -7 },
      { "type": "public-key", "alg": -257 }
    ],
    "authenticatorSelection": {
      "userVerification": "required",
      "residentKey": "preferred"
    },
    "attestation": "none",
    "timeout": 60000
  }
}
```

`-7` 是 ES256，`-257` 是 RS256；`userVerification: required` 要求认证器进行用户验证；`residentKey: preferred` 表示优先支持可发现凭证。

**生产环境注意**：示例中将用户名的 UTF-8 字节编码为 WebAuthn User ID，只是为了简化演示。正式应用应使用稳定、唯一且不暴露个人信息的内部用户标识。

## 12. 后端验证注册响应

浏览器返回注册 Credential 后，服务端先消费 Challenge，然后交给 WebAuthn4J 解析和验证：

```java
PendingChallenge pending = consumeChallenge(
        requestId, username, ChallengePurpose.REGISTER
);

RegistrationData registrationData =
        webAuthnManager.parseRegistrationResponseJSON(credentialJson);

RegistrationParameters parameters = new RegistrationParameters(
        serverProperty(pending.challenge()),
        null,
        true,
        true
);

webAuthnManager.verify(registrationData, parameters);
```

服务端安全上下文由 `ServerProperty` 提供：

```java
private ServerProperty serverProperty(byte[] challenge) {
    return ServerProperty.builder()
            .origin(new Origin("http://localhost:4200"))
            .rpId("localhost")
            .challenge(new DefaultChallenge(challenge))
            .build();
}
```

验证成功后，从 `registrationData` 提取 Credential ID，创建 `CredentialRecordImpl`，再通过 DAO 保存公钥凭证记录。

## 13. 后端生成并验证登录请求

登录前先查询该用户的凭证，构造 `allowCredentials`，返回新的 Challenge：

```json
{
  "requestId": "generated-request-id",
  "publicKey": {
    "challenge": "base64url-challenge",
    "rpId": "localhost",
    "allowCredentials": [
      { "type": "public-key", "id": "base64url-credential-id" }
    ],
    "userVerification": "required",
    "timeout": 60000
  }
}
```

浏览器返回 Authentication Response 后，后端根据 Credential ID 查询已保存的凭证，并确认它属于当前用户：

```java
AuthenticationData authenticationData =
        webAuthnManager.parseAuthenticationResponseJSON(credentialJson);

StoredCredential storedCredential = webAuthnDao
        .findCredential(authenticationData.getCredentialId())
        .orElseThrow(() -> new IllegalArgumentException("Credential not found"));

if (!storedCredential.username().equals(username)) {
    throw new IllegalArgumentException("Credential does not belong to user");
}

AuthenticationParameters parameters = new AuthenticationParameters(
        serverProperty(pending.challenge()),
        storedCredential.credentialRecord(),
        List.of(storedCredential.credentialId()),
        true,
        true
);

webAuthnManager.verify(authenticationData, parameters);
```

这一步验证成功，代表 WebAuthn 认证响应通过了服务端检查，**但还不代表应用已经建立 Session**。

## 14. REST API 设计

Controller 提供四个端点，统一前缀为 `/api/auth/webauthn`：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| POST | `/register/challenge` | 获取注册选项 |
| POST | `/register/verify` | 验证注册结果 |
| POST | `/login/challenge` | 获取登录选项 |
| POST | `/login/verify` | 验证登录结果 |

请求 DTO：

```java
public record BeginRequest(String username) {}

public record VerifyRequest(
        String username,
        String requestId,
        JsonNode credential
) {}
```

本项目使用 Jackson 3 的 `tools.jackson.databind.JsonNode` 和 `ObjectMapper`。Controller 将 Credential JSON 序列化为字符串后传给 WebAuthn4J。不要混用 Jackson 2 的 `com.fasterxml.jackson.databind.ObjectMapper` 类型。

## 15. Angular 集成 SimpleWebAuthn

在 `ui` 目录安装：

```bash
npm install @simplewebauthn/browser
```

使用 `@Injectable()` 定义 Angular Service，并显式导入 SimpleWebAuthn 的 JSON 类型，避免与 DOM 内置同名类型冲突：

```typescript
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { firstValueFrom } from 'rxjs';
import {
  startRegistration,
  startAuthentication,
  browserSupportsWebAuthn,
} from '@simplewebauthn/browser';
import type {
  PublicKeyCredentialCreationOptionsJSON as RegistrationOptions,
  PublicKeyCredentialRequestOptionsJSON as AuthenticationOptions,
} from '@simplewebauthn/browser';

interface ChallengeResponse<T> {
  requestId: string;
  publicKey: T;
}

interface WebAuthnResult {
  status: string;
  message: string;
}

@Injectable({ providedIn: 'root' })
export class WebauthnService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = 'http://localhost:8080/api/auth/webauthn';

  isSupported(): boolean {
    return browserSupportsWebAuthn();
  }

  async register(username: string): Promise<WebAuthnResult> {
    const { requestId, publicKey } = await firstValueFrom(
      this.http.post<ChallengeResponse<RegistrationOptions>>(
        `${this.baseUrl}/register/challenge`, { username }
      )
    );

    const credential = await startRegistration({ optionsJSON: publicKey });

    return firstValueFrom(this.http.post<WebAuthnResult>(
      `${this.baseUrl}/register/verify`,
      { username, requestId, credential }
    ));
  }

  async login(username: string): Promise<WebAuthnResult> {
    const { requestId, publicKey } = await firstValueFrom(
      this.http.post<ChallengeResponse<AuthenticationOptions>>(
        `${this.baseUrl}/login/challenge`, { username }
      )
    );

    const credential = await startAuthentication({ optionsJSON: publicKey });

    return firstValueFrom(this.http.post<WebAuthnResult>(
      `${this.baseUrl}/login/verify`,
      { username, requestId, credential }
    ));
  }
}
```

`startRegistration()` 和 `startAuthentication()` 是整个 Angular 实现的核心。它们封装了原生 `navigator.credentials.create()` / `get()` 及 ArrayBuffer、Base64URL、JSON 之间的转换。

**SSR 提醒**：Angular SSR 环境没有 `window`、`document` 等浏览器全局对象。实际项目中应通过 `isPlatformBrowser()` 或 `afterNextRender()` 在客户端检测 WebAuthn 支持情况，而不是在服务端渲染阶段直接执行浏览器 API。

## 16. Angular Material 页面

页面只需要一个用户名输入框、两个按钮，以及加载、成功、失败状态。使用 Reactive Forms 和 Signals：

```typescript
readonly username = new FormControl('', {
  nonNullable: true,
  validators: [Validators.required],
});
readonly loading = signal(false);
readonly error = signal('');
readonly success = signal('');

async register(): Promise<void> {
  await this.execute(() =>
    this.webauthnService.register(this.username.value.trim())
  );
}

async login(): Promise<void> {
  await this.execute(() =>
    this.webauthnService.login(this.username.value.trim())
  );
}
```

模板示意：

```html
<mat-card appearance="outlined">
  <mat-card-header>
    <mat-card-title>Passkey Authentication</mat-card-title>
  </mat-card-header>
  <mat-card-content>
    <mat-form-field appearance="outline">
      <mat-label>Username</mat-label>
      <input matInput [formControl]="username" autocomplete="username" />
    </mat-form-field>
    <button mat-flat-button (click)="register()">
      <mat-icon>fingerprint</mat-icon>
      Register Passkey
    </button>
    <button mat-stroked-button (click)="login()">
      <mat-icon>login</mat-icon>
      Login with Passkey
    </button>
  </mat-card-content>
</mat-card>
```

实际页面应禁用重复提交，并通过 `loading()`、`error()`、`success()` 显示对应状态。注意不要在认证失败后仍然执行页面跳转；应只在服务端确认认证成功后继续后续流程。

## 17. 本地测试

```text
Angular:      http://localhost:4200
Spring Boot:  http://localhost:8080
RP ID:        localhost
Origin:       http://localhost:4200
```

1. 输入用户名 `arthur`，点击 **Register Passkey**。
2. 在浏览器弹出的界面中选择 Apple Passwords、Microsoft Password Manager 或其他认证器，完成注册。
3. 再次输入相同用户名，点击 **Login with Passkey**。
4. 认证器完成私钥签名，Spring Boot 使用保存的公钥凭证验证响应。

成功时，后端返回类似：

```json
{
  "status": "success",
  "message": "Passkey authentication successful"
}
```

这里返回的是**WebAuthn 验证结果**，不是 JWT 或已建立的登录 Session。

## 18. 从 Demo 到生产环境还差什么？

当前实现主要用于学习，正式使用前至少需要完善以下方面：

1. **持久化**：用 PostgreSQL 保存 Credential，用具备 TTL 和原子消费能力的存储管理 Challenge；内存 DAO 重启即丢失数据。
2. **可信用户绑定**：不能允许任何人仅凭用户名为现有账号注册 Passkey。必须先完成账号所有权验证，并从可信上下文获取用户 ID。
3. **会话管理**：WebAuthn 验证成功后，需要与 Spring Security、Session 或 Token 机制集成。
4. **凭证状态更新**：维护签名计数器和备份状态；可同步 Passkey 的计数器不一定严格递增。
5. **凭证管理**：支持查看、添加、撤销多个 Passkey，并设计安全的账号恢复机制。
6. **异常与安全策略**：限制滥用、处理重复注册和过期 Challenge、保护认证接口并增加测试。

另外，删除 Apple Passwords 中的 `localhost` Passkey 不会自动删除服务器保存的 Credential；删除服务端 Credential 也不会自动删除设备中的凭证。

## 总结

WebAuthn 的核心**不是指纹识别，而是公钥密码学与浏览器对网站身份的绑定**。指纹、Face ID 和 PIN 通常只是本地授权使用私钥的方式。

```text
注册：认证器生成密钥对 → 服务器验证并保存公钥凭证
登录：服务器生成 Challenge → 认证器私钥签名 → 服务器公钥验证
```

Angular 负责获取 Challenge、调用认证器并提交结果；Spring Boot 负责生成 Challenge、验证响应及管理 Credential；SimpleWebAuthn 和 WebAuthn4J 则封装了底层协议细节。

通过这个 Demo，我们完成了最基本的 Passkey 注册和认证闭环。下一步可以继续接入 Spring Security、数据库持久化和正式的账号体系。

## 参考资料

- [W3C Web Authentication](https://www.w3.org/TR/webauthn-3/)
- [MDN Web Authentication API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API)
- [SimpleWebAuthn Documentation](https://simplewebauthn.dev/docs/)
- [WebAuthn4J Documentation](https://webauthn4j.github.io/webauthn4j/en/)
- [FIDO Alliance — Passkeys](https://fidoalliance.org/passkeys/)

> 本文示例基于 Batcher 项目的 WebAuthn 实验代码，用于原理学习与技术演示，不应直接视为生产级认证方案。
