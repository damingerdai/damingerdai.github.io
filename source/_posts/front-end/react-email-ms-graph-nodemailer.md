---
title: 基于 Microsoft Graph API 与 React Email 构建现代化邮件发送系统
date: 2025-12-27 20:05:34
tags: [react, react-email, Microsoft Graph API,nodemailer]
categories: [前端]
---

# 背景

在传统的 Node.js 服务端开发中，使用 Nodemailer 配合 SMTP 协议是发送邮件的标准做法。然而，随着安全标准的提升，微软已宣布将在 2026 年彻底停止 Outlook/Microsoft 365 对 SMTP 基础身份验证（Basic Auth）的支持。

为了应对这一变化，我们需要转向 Microsoft Graph API。同时，为了解决邮件模板开发中样式难以调试、兼容性差的痛点，我们可以引入 React Email。本文将介绍如何结合这两者，构建一套符合 Shadcn UI 审美风格且具备高度可靠性的邮件发送系统。

---

# 技术栈

* **React Email**: 使用 React 组件编写邮件模板，支持 Tailwind CSS。
* **Nodemailer**: 成熟的 Node.js 邮件发送库。
* **Microsoft Graph API**: 微软提供的现代化 API 接口，用于替代传统 SMTP。
* **MSAL Node**: 微软官方提供的身份验证库。

---

## 1. 架构设计

系统分为三层：模板层、逻辑层和传输层。

* **模板层**：利用 React Email 定义 UI，确保样式在各邮件客户端（如 Gmail、Outlook、iOS Mail）中表现一致。
* **传输层**：通过自定义 Nodemailer 的 `Transport` 接口，将底层的邮件发送请求导向 Microsoft Graph API。
* **认证层**：使用 Azure 应用注册（App Registration）获取 OAuth2 令牌。

---

## 2. 实现 Microsoft Graph 传输器

我们需要实现一个自定义的 `AzureTransport` 类。它的核心逻辑是：在发送邮件前检查令牌是否过期，若过期则通过客户端凭据流（Client Credentials Flow）获取新令牌，随后调用 Graph API 的 `/sendMail` 接口。

```typescript
import * as msal from '@azure/msal-node';
import { SentMessageInfo, Transport } from 'nodemailer';
import MailMessage from 'nodemailer/lib/mailer/mail-message';

export class AzureTransport implements Transport<SentMessageInfo> {
  name = 'Azure';
  version = '0.1';
  private msalClient: msal.ConfidentialClientApplication;
  private tokenInfo: msal.AuthenticationResult | null = null;

  constructor(private config: any) {
    this.msalClient = new msal.ConfidentialClientApplication({
      auth: {
        clientId: config.clientId,
        clientSecret: config.clientSecret,
        authority: `https://login.microsoftonline.com/${config.tenantId}`
      }
    });
  }

  private async getAccessToken() {
    if (!this.tokenInfo || (this.tokenInfo.expiresOn && Date.now() > this.tokenInfo.expiresOn.getTime())) {
      this.tokenInfo = await this.msalClient.acquireTokenByClientCredential({
        scopes: ['https://graph.microsoft.com/.default']
      });
    }
    return this.tokenInfo!.accessToken;
  }

  async send(mail: MailMessage, callback: any) {
    try {
      const { subject, from, to, html, text } = mail.data;
      const accessToken = await this.getAccessToken();

      const mailMessage = {
        message: {
          subject,
          body: { content: html || text, contentType: html ? 'HTML' : 'Text' },
          toRecipients: (Array.isArray(to) ? to : [to]).map((addr: any) => ({
            emailAddress: { address: typeof addr === 'string' ? addr : addr.address }
          })),
        }
      };

      const response = await fetch(`https://graph.microsoft.com/v1.0/users/${from}/sendMail`, {
        method: 'POST',
        headers: { Authorization: `Bearer ${accessToken}`, 'Content-Type': 'application/json' },
        body: JSON.stringify(mailMessage)
      });

      if (!response.ok) throw new Error(await response.text());
      callback(null, { messageId: response.headers.get('request-id') });
    } catch (error) {
      callback(error, null);
    }
  }
}

```

---

### 3. 使用 React Email 编写 Shadcn 风格模板

Shadcn UI 的核心在于简洁的 Zinc 色调和良好的间距。以下是为用户确认设计的通用模板：

```tsx
import { Body, Container, Head, Heading, Html, Preview, Text, Tailwind, Button } from "@react-email/components";

export const RequestReceivedEmail = () => (
  <Html>
    <Head />
    <Preview>Confirmation of your request</Preview>
    <Tailwind config={{ theme: { extend: { colors: { brand: "#09090b" } } } }}>
      <Body className="bg-white font-sans text-zinc-950">
        <Container className="border border-zinc-200 rounded-lg my-10 mx-auto p-8 max-w-[465px]">
          <Heading className="text-2xl font-semibold tracking-tight">Request Received</Heading>
          <Text className="text-sm leading-6 text-zinc-600 mt-4">
            We have successfully received your information. Our team will review the details and respond accordingly.
          </Text>
          <Button className="bg-zinc-950 rounded-md text-white text-sm font-medium px-6 py-3 mt-4 no-underline inline-block" href="https://app.aands.ai">
            Access Portal
          </Button>
        </Container>
      </Body>
    </Tailwind>
  </Html>
);

```

---

## 4. 业务逻辑整合

最后，我们将上述模块整合到一个服务函数中。该函数会同时向用户发送回执，并向运营团队发送内部通知。

```typescript
import { render } from "@react-email/components";
import nodemailer from "nodemailer";

const transporter = nodemailer.createTransport(new AzureTransport({
  clientId: process.env.AZURE_CLIENT_ID,
  clientSecret: process.env.AZURE_CLIENT_SECRET,
  tenantId: process.env.AZURE_TENANT_ID
}));

export const sendNotification = async (userData: any) => {
  const userHtml = await render(<RequestReceivedEmail />);
  
  await transporter.sendMail({
    from: "noreply@yourdomain.com",
    to: userData.email,
    subject: "Request Confirmation",
    html: userHtml
  });
};

```

---

# 总结

通过这种方式，我们不仅解决了 SMTP 认证即将过期的合规性问题，还提升了邮件的视觉质量。Microsoft Graph API 提供了比传统 SMTP 更强的可监控性和安全性，而 React Email 则极大地改善了邮件开发的开发者体验。

在实际生产环境中，请确保在 Azure 门户中为你的应用授予了 `Mail.Send` 权限，并根据业务需求配置环境变量。

# 参考资料

1. [构建和发送电子邮件的React组件的实例教程](https://juejin.cn/post/7157980257155285005)
2. [Sending Emails via Outlook with Nodemailer and Microsoft Graph](https://dev.to/gevik/sending-emails-via-outlook-with-nodemailer-and-microsoft-graph-b74)