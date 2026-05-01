---
title: 使用 Resend + Golang 发送邮件的完整踩坑与解决方案
date: 2026-05-01 17:25:12
tags: [resend, golang]
categories: [后端]
---

# 使用 Resend + Golang 发送邮件的完整踩坑与解决方案

在实际开发中，我尝试使用 Resend 的 SMTP 服务结合 Golang 发送邮件（用于重置密码等场景），过程中踩了多个典型坑。本文从 **错误现象 → 原因分析 → 最终正确方案** 进行完整总结。

---

## 一、背景

目标：

* 使用 Golang 发送 HTML 邮件
* 接入 Resend SMTP
* 支持自定义域名发信（提升送达率）

---

## 二、Resend SMTP 基本认知（关键）

Resend 的 SMTP 和传统 SMTP 有一个重要区别：

| 类型            | 含义      | 示例                |
| ------------- | ------- | ----------------- |
| SMTP Username | 登录用户名   | `resend`（固定）      |
| SMTP Password | API Key | `re_xxx`          |
| From（发件人）     | 邮件发送地址  | `noreply@xxx.com` |

👉 **重点：Username ≠ From**

---

## 三、踩坑记录（按时间顺序）

---

### ❌ 问题 1：501 Bad sender address syntax

```txt
501 Error: Bad sender address syntax
```

#### 原因

```go
auth := smtp.PlainAuth("", m.from, m.password, m.host)
```

把 `from=resend` 当成邮箱使用。

#### 本质问题

* `resend` 不是合法邮箱地址
* SMTP 要求符合 RFC 规范

---

### ❌ 问题 2：535 Invalid username

```txt
535 Invalid username
```

#### 原因

```go
auth := smtp.PlainAuth("", from, m.password, m.host)
```

把邮箱当成 SMTP username。

#### 正确规则

```txt
username 必须是：resend
```

---

### ❌ 问题 3：550 Missing or invalid "from" field

```txt
550 Missing or invalid "from" field: undefined
```

#### 原因

手写 SMTP message 时，没有加 Header：

```txt
From: xxx
```

#### 关键点

SMTP 有两个 from：

| 类型            | 说明      |
| ------------- | ------- |
| envelope from | SMTP 参数 |
| header from   | 邮件内容    |

👉 两者必须同时存在

---

### ❌ 问题 4：550 domain is not verified

```txt
The damingerdai.com domain is not verified
```

#### 原因

使用了：

```txt
notifications@damingerdai.com
```

但实际验证的是：

```txt
notifications.damingerdai.com
```

👉 域名不匹配

---

## 四、关键认知（最重要）

### 1️⃣ Resend 只认“已验证域”

假设你验证了：

```txt
notifications.damingerdai.com
```

那么合法发件人只能是：

```txt
anything@notifications.damingerdai.com ✅
```

而不是：

```txt
anything@damingerdai.com ❌
```

---

### 2️⃣ local-part 是自由的

```txt
noreply@notifications.damingerdai.com
support@notifications.damingerdai.com
hello@notifications.damingerdai.com
```

👉 前缀不需要真实邮箱存在

---

## 五、最终正确配置

### 环境变量

```env
SMTP_HOST=smtp.resend.com
SMTP_PORT=587

SMTP_USERNAME=resend
SMTP_PASSWORD=your_api_key

SMTP_FROM=noreply@notifications.damingerdai.com
```

---

## 六、Golang 实现

---

### 方案一：使用 net/smtp（低层）

```go
auth := smtp.PlainAuth("", m.username, m.password, m.host)

headers := ""
headers += fmt.Sprintf("From: %s\r\n", m.from)
headers += fmt.Sprintf("To: %s\r\n", toEmail)
headers += "Subject: Reset Password\r\n"
headers += "MIME-Version: 1.0\r\n"
headers += "Content-Type: text/html; charset=\"UTF-8\"\r\n"
headers += "\r\n"

message := []byte(headers + body)

err := smtp.SendMail(
	addr,
	auth,
	m.from,
	[]string{toEmail},
	message,
)
```

---

### 方案二：使用 go-mail（推荐）

```go
msg := mail.NewMessage()
msg.SetHeader("From", "Health Master <"+m.from+">")
msg.SetHeader("To", toEmail)
msg.SetHeader("Subject", "Reset Password")
msg.SetBody("text/html", body)

d := mail.NewDialer(m.host, 587, m.username, m.password)

err := d.DialAndSend(msg)
```

---

## 七、最佳实践

---

### ✅ 使用子域发信（强烈推荐）

```txt
notifications.damingerdai.com
```

优势：

* 不影响主域信誉
* 更专业（邮件服务隔离）

---

### ✅ 设置规范发件人

```txt
noreply@notifications.damingerdai.com
```

---

### ✅ Header 使用带名称格式

```txt
Health Master <noreply@notifications.damingerdai.com>
```

---

## 八、总结

整个过程可以总结为三点：

---

### 1️⃣ 三个字段必须分清

```txt
SMTP_USERNAME = resend
SMTP_PASSWORD = API Key
SMTP_FROM     = 你的邮箱
```

---

### 2️⃣ 域名必须完全匹配

```txt
验证的是 A，就不能用 B
```

---

### 3️⃣ SMTP 是“严格协议”

不会容错：

* 少一个 Header ❌
* 格式不对 ❌
* 域名不对 ❌

---

## 九、建议

如果不是必须用 SMTP：

👉 优先考虑 Resend API（更简单、更少坑）

---

## 十、附录（调试建议）

常见排查：

```bash
dig TXT yourdomain.com
```

检查：

* SPF
* DKIM

---

## 结尾

这次接入的核心教训：

> SMTP 的问题，80% 不是代码问题，而是协议和配置问题。

如果你也在用 Golang + 邮件服务，这一套坑基本可以帮你全部避开。
