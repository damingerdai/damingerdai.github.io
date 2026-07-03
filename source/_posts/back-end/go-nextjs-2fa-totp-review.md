---
title: Go + Next.js 实现 2FA (TOTP) 的踩坑记录与架构复盘
date: 2026-07-03 23:04:38
tags: [go, nextjs, 2fa, totp]
categories: [后端]
---

# Go + Next.js 实现 2FA (TOTP) 的踩坑记录与架构复盘

最近给手头的项目加上了基于 TOTP（基于时间的一次性密码）的双因子认证（2FA）流程。整体思路是传统的**双阶段认证（Two-Stage）**：

1. 第一步先验密码，对上了且开启了 2FA，后端不发正式 Token，而是塞给前端一个只有 3 ~ 5 分钟寿命的 `challengeToken`。
2. 前端监听到这个状态，通过 Next.js 的 URL 参数（比如 `/login?step=2fa`）做深链接导航，直接切到 2FA 验证页，用 `input-otp` 组件让用户填 6 位动态码，最后去后端换取真正的登录凭证。

本来以为一套流程跑下来挺顺畅，但利用AI对着提交的 Git Diff 仔细过了一遍安全性后，发现里面其实藏了不少逻辑漏洞和优化空间。趁着还没线上翻车，把这次的数据库改动和后续的重构 TODO 记录下来（感激AI的review）。

## 数据库表结构变更

这次 2FA 的底层支持直接做在了原有的 `users` 表上，通过变更 DDL 增加了三个字段：

```sql
ALTER TABLE users
ADD COLUMN two_factor_enabled BOOLEAN DEFAULT FALSE,
ADD COLUMN two_factor_secret TEXT,
ADD COLUMN two_factor_verified_at TIMESTAMPTZ;

```

* `two_factor_enabled`: 标记用户是否正式开启了 2FA。
* `two_factor_secret`: 存储经过加密（或明文，取决于后续优化）的 TOTP 密钥。
* `two_factor_verified_at`: 记录用户首次激活或最后一次成功验证的时间戳，用来做激活确认和状态审计。

这个设计非常轻量，但也正是因为这种“一刀切”的单表字段设计，直接引发了后面要聊的第 6 个 TODO。

### Go的实现

我使用的是github.com/pquerna/otp这个库。

```Golang
package service

type TwoFactorService struct {
	Aes            *cryptox.AES
	UserRepository *repository.UserRepository
}

type Setup2FaResult struct {
	Secret string `json:"secret"`
	QRCode string `json:"qr_code"`
}

func (s *TwoFactorService) Generate(ctx context.Context, userID string, email string) (*model.Setup2FaResult, error) {
	if s.Aes == nil {
		return nil, errors.New("2fa is not configured")
	}

	key, err := totp.Generate(totp.GenerateOpts{
		Issuer:      "HealthMaster",
		AccountName: email,
	})
	if err != nil {
		return nil, err
	}

	secret := key.Secret()
	aseSecret, err := s.Aes.Encrypt(secret)
	if err != nil {
		return nil, err
	}
	err = s.UserRepository.SaveTwoFactorSecret(ctx, userID, aseSecret)
	if err != nil {
		return nil, err
	}

	return &model.Setup2FaResult{
		QRCode: key.URL(),
		Secret: secret,
	}, nil
}

func (s *TwoFactorService) Enable(ctx context.Context, userID string, code string) error {
	if s.Aes == nil {
		return errors.New("2fa is not configured")
	}

	user, err := s.UserRepository.Find(ctx, userID)
	if err != nil {
		return err
	}
	if user == nil {
		return errors.New("user not found")
	}
	if user == nil {
		return errors.New("user not found")
	}
	if user.TwoFactorSecret == nil {
		return errors.New("2fa secret not found")
	}

	twoFactorSecret, err := s.Aes.Decrypt(*user.TwoFactorSecret)
	if err != nil {
		return err
	}

	ok := totp.Validate(code, twoFactorSecret)
	if !ok {
		return errors.New("invalid code")
	}

	return s.UserRepository.EnableTwoFactor(ctx, userID)
}

func (s *TwoFactorService) Disable(ctx context.Context, userID string, code string) error {
	/**
        和Enable类似
    **/

	return s.UserRepository.DisableTwoFactor(ctx, userID)
}

func (s *TwoFactorService) VerifyCode(ctx context.Context, userID string, code string) error {
	/**
        和Enable类似
    **/

	if !totp.Validate(code, secret) {
		return errors.New("invalid verification code")
	}

	return nil
}

```


## Code Review 后的重构 TODO 列表

以下是目前首版代码里暴露出来的隐患，也是接下来的重构重点：

### ⬜ TODO 1: 拦截 2FA 重放攻击（Replay Attack）

* **现状**：TOTP 的 6 位验证码在 30 秒的步长内是静态不变的。这意味着如果请求被截获，或者前端因为网络抖动在 30 秒内连发了两次请求，后端会连续验证通过两次。
* **改法**：引入 Redis 缓存。只要某个验证码在当前时间步长内被成功消费了一次，就往 Redis 里扔个带过期时间的标记。下次再进来相同的码，直接拒绝，确保一个验证码 30 秒内只能用一次。

### ⬜ TODO 2: 绑定阶段的幂等与中间态保护

* **现状**：目前调用 `GET /api/v1/settings/2fa` 时，后端每次都会刷新并生成全新的 `secret`。如果用户扫完 QR 码后页面不小心刷新了，前端拿到了新 Secret，而用户输入旧 App 里的验证码去调用 `PUT` 激活，就会直接报错。
* **改法**：完善 Pending 状态。如果用户还没正式开启 2FA，`GET` 请求应该先看数据库里有没有未激活的 Secret，有就复用旧的，别盲目生成新的。只有在 `PUT` 验证成功后，才把 `two_factor_enabled` 改为 `true`。

### ⬜ TODO 3: 限制单 TOTP 绑定，为未来扩展留出空间（Passkey / 多设备）

* **现状**：因为这次直接在 `users` 表上硬编码了 `two_factor_secret`，导致一个用户**只能绑定一个 TOTP 密钥**，不支持绑定多个设备，更没法兼容像 Passkey（FIDO2）、WebAuthn、或者短信/邮件等其他认证方式。
* **改法**：目前第一版先收拢业务口，明确“仅支持单个 TOTP 绑定”的限制。但后续如果业务需要支持多设备或者 Passkey，必须把这三个字段从 `users` 表里剥离出来，重构成一张独立的 `user_credentials` 或 `two_factor_methods` 表，用一对多的关系来承载不同的认证实体。

### ⬜ TODO 4: 增加“恢复码 / 救砖码”（Backup Codes）

* **现状**：目前要是用户手机丢了、误删了 Authenticator App，基本上就彻底死锁，只能找后台提工单删数据库字段。
* **改法**：在 `PUT` 激活 2FA 成功的那一刻，后端生成 8~10 个一次性的随机恢复码展示给用户。数据库再开一张表存储这些恢复码的哈希值（绝对不能存明文）。登录时，2FA 验证框同时兼容 6 位动态码和恢复码。

### ⬜ TODO 5: 收紧 Challenge Token 的越权风险

* **现状**：第一阶段密码校验通过后发给前端的临时 Token，如果没有做好权限隔离，可能会被用来尝试请求其他受保护的业务接口。
* **改法**：在临时 Token 的 Payload 里塞进严格的 `scope: "2fa_challenge"`。后端的权限中间件必须拦截所有非 2FA 验证接口，只要看到带这个 Scope 的 Token 访问其他业务，一律拍回 `403 Forbidden`。

### ⬜ TODO 6: 高危操作的二次验证（Double Check）

* **现状**：现在 2FA 只是把住了登录的关口。一旦登录进去，攻击者如果拿到 Session，可以直接调用接口把 2FA 关掉，或者去改密码。
* **改法**：在关闭 2FA（`PUT /api/v1/settings/2fa` 传 `enabled: false`）或者修改密码、改绑定邮箱等高危接口里，Request Body 必须强制要求再传入一次当前的 TOTP 动态码，验证通过才执行修改，做到纵深防御。

---

## 碎碎念

写业务逻辑确实快，但认证和安全这块，往往是“写代码半小时，补漏洞两天”。这次的 Diff 算是把骨架搭起来了，接下来几天得把上面这几个 TODO 逐一消掉。如果你也准备给自己的项目手写 2FA，建议在设计之初就把重放攻击和恢复码的逻辑考虑进去，免得后面重构数据库时抓耳挠腮。

## 引用

1. [基于时间的一次性密码 TOTP 详解](https://zhuanlan.zhihu.com/p/664203530)
2. [使用 Golang 实现基于时间的一次性密码 TOTP](https://zhuanlan.zhihu.com/p/665398081)