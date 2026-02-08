---
title: 实践分享：在 Go 项目中实现一个安全的密码重置流程
date: 2026-02-08 22:39:45
tags: [golang, postgres]
categories: [后端]
---

# 实践分享：在 Go 项目中实现一个安全的密码重置流程

## 1. 业务场景

在我们的 [health-master](https://github.com/damingerdai/health-master/issues/243) 项目中，我们需要一个安全且用户体验良好的密码重置功能。用户流程如下：

1. 输入邮箱申请重置  收到邮件链接。
2. 点击链接  页面显示脱敏后的账号（确认没点错）。
3. 输入并提交新密码  链接失效  修改成功。

## 2. 技术难点与设计方案

### 2.1 数据库建模

我们不只是存一个 Token 字符串，而是将“重置请求”看作一个有生命周期的资源。

```
CREATE TABLE tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    token_hash VARCHAR(64) NOT NULL, -- 存储 SHA-256 哈希值而非明文
    category VARCHAR(20) NOT NULL,   -- 类型，如 password_reset
    used_count INT DEFAULT 0,       -- 已使用次数
    max_uses INT DEFAULT 1,         -- 最大使用次数（1即阅后即焚）
    expires_at TIMESTAMP NOT NULL,  -- 过期时间
    is_revoked BOOLEAN DEFAULT FALSE -- 是否手动撤销
);
CREATE INDEX idx_token_hash ON tokens(token_hash);
```

### 2.2 Token 的安全性：Hash 存储

直接在数据库存储明文 Token 是危险的。我们效仿密码存储的逻辑，在数据库中存储 Token 的 **SHA-256 哈希值**。

* **生成：** `crypto/rand` 生成 32 字节随机数  发送给用户。
* **存储：** `sha256(raw_token)`  存入 `token_hash` 字段。
* **校验：** 接收用户传来的 Token  哈希后去数据库匹配。

### 2.3 原子核销：防止并发重置

为了保证“阅后即焚”（重置一次即失效），我们使用了 PostgreSQL 的原子更新操作。在 `ConsumeToken` 时，我们通过一行 SQL 同时完成校验与核销：

```sql
UPDATE tokens
SET used_count = used_count + 1, last_used_at = NOW()
WHERE token_hash = $1 
  AND category = 'password_reset'
  AND used_count < max_uses  -- 核心：确保只用一次
  AND expires_at > NOW()
RETURNING user_id;

```

### 2.4 隐私保护：SQL 层面的脱敏

用户打开页面时，我们希望展示其 Email 但不暴露完整地址。为了性能，我们直接在 SQL 中处理：

```sql
SELECT 
    CASE 
        WHEN POSITION('@' IN u.email) > 3 THEN 
            OVERLAY(u.email PLACING '***' FROM 3 FOR POSITION('@' IN u.email) - 3)
        ELSE 
            OVERLAY(u.email PLACING '***' FROM 2 FOR POSITION('@' IN u.email) - 2)
    END AS masked_email
FROM tokens t JOIN users u ON t.user_id = u.id ...

```

## 3. 代码架构实现 (Service 层)

在 `ResetPassword` 的 Service 实现中，我们利用了 Go 的 **命名返回值 (Named Return Parameters)** 来优雅地管理事务。这种写法确保了只要有任何一步报错，`tx.Rollback()` 就会被正确执行。

```go
func (userService *UserService) ResetPassword(ctx context.Context, email, rawToken, newPassword string) (user *model.User, err error) {
    // 1. 耗时操作：Bcrypt 放在事务外
    hashedPassword, _ := bcrypt.GenerateFromPassword([]byte(newPassword), bcrypt.DefaultCost)

    tx, _ := userService.pool.Begin(ctx)
    defer func() {
        if err != nil { tx.Rollback(ctx) } else { err = tx.Commit(ctx) }
    }()

    repos := repository.New(tx)
    // 2. 核销 Token
    tokenRecord, err := repos.TokenRecordRepository.ConsumeToken(ctx, rawToken, "password_reset")
    // 3. 更新密码...
}

```

## 4. RESTful API 设计

为了符合现代 API 规范，我们将重置流程定义为对 `password-resets` 资源的操作：

* `POST /password-resets`: 创建重置请求。
* `GET /password-resets/:token`: 预检查并获取用户信息。
* `PUT /password-resets/:token`: 执行更新。

## 5. TODO：后续优化

这个 PR 已经打下了坚实的基础，后续我们还可以：

1. **自动清理：** 增加 Cron Job 每天定时清理过期超过 24 小时的 Token 记录。
2. **安全审计：** 在重置成功后，发送一封“密码已修改”的提醒邮件。