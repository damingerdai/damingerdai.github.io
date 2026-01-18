---
title: Next.js + Supabase 实现完美的重置密码工作流（含 React Email 与 Resend SMTP 配置）
date: 2026-01-18 08:28:27
tags: [nexjts,spabase,react email, resend]
categories: [前端]
---


# 前言

在现代 Web 开发中，身份验证（Auth）是一个核心环节。Supabase 提供了强大的 Auth 服务，但如何将其完美集成到 Next.js (App Router) 中，并使用 React Email 自定义精美的邮件模板？本文将带你走通整个流程。

# 1. 核心工作流设计

重置密码并非简单的“输入新密码”，为了安全性，它必须遵循 **PKCE (Proof Key for Code Exchange)** 流程：

1. **Forgot Password**: 用户输入邮箱，请求重置。
2. **Magic Link**: Supabase 发送一封带有 `code` 的邮件。
3. **Auth Callback**: 用户点击链接，回到应用后端交换 `code` 获取 `Session`。
4. **Update Password**: 用户进入受保护的修改页面，提交新密码。

# 2. 环境配置：让 Supabase 认识你的多域名

Supabase 的 **Site URL** 只能设置一个（通常设为生产环境），但你可以通过 **Redirect URLs** 支持多环境（如 localhost 和 Vercel 预览）。

## 后台配置清单：

* **Site URL**: `https://fieldglass-app.vercel.app`
* **Redirect URLs**:
* `http://localhost:3000/**`
* `https://fieldglass-app.vercel.app/**`



> **注意**：一定要添加 `/**` 通配符，否则带有查询参数的回调地址会被拦截。


# 3. 发送请求：Forgot Password 页面

使用 **shadcn/ui** 和 **react-hook-form** 构建一个极简的邮箱提交页面。

```tsx
async function onSubmit(values: z.infer<typeof formSchema>) {
  setIsLoading(true);
  const { error } = await supabase.auth.resetPasswordForEmail(values.email, {
    // 动态处理重定向地址
    redirectTo: `${window.location.origin}/auth/callback?next=/update-password`,
  });

  if (error) {
    toast.error(error.message);
  } else {
    toast.success("Reset link sent! Please check your inbox.");
  }
  setIsLoading(false);
}

```


# 4. 邮件视觉：React Email + Resend SMTP

Supabase 默认邮件配额较低且样式简陋。我们通过 **Resend** 配置自定义 SMTP。

## SMTP 配置 (Supabase Dashboard):

* **Host**: `smtp.resend.com` | **Port**: `465`
* **User**: `resend` | **Password**: `你的Resend_API_KEY`

## React Email 模板设计:

使用 React Email 编写 UI，并保留 Supabase 的变量 `{{ .ConfirmationURL }}`。

```tsx
import {
  Body,
  Button,
  Container,
  Head,
  Heading,
  Html,
  Preview,
  Section,
  Text,
  Tailwind,
  Hr,
} from "@react-email/components";

export const ResetPasswordEmail = () => {
  return (
    <Html>
      <Head />
      <Preview>Reset your Fieldglass App password</Preview>
      <Tailwind>
        <Body className="bg-white my-auto mx-auto font-sans">
          <Container className="border border-solid border-[#e5e7eb] rounded-lg my-[40px] mx-auto p-[32px] max-w-[465px] shadow-sm">
            <Section>
              {/* Text-based Logo / App Name */}
              <Text className="text-black text-[20px] font-bold tracking-tight m-0">
                Fieldglass App
              </Text>
            </Section>
            
            <Heading className="text-black text-[24px] font-semibold text-left p-0 my-[30px] mx-0">
              Reset your password
            </Heading>
            
            <Text className="text-[#374151] text-[14px] leading-[24px]">
              We received a request to reset the password for your **Fieldglass App** account. 
              Click the button below to proceed.
            </Text>

            <Section className="mt-[32px] mb-[32px]">
              <Button
                className="bg-[#000000] rounded-md text-white text-[14px] font-medium no-underline text-center px-6 py-3"
                href="{{ .ConfirmationURL }}"
              >
                Reset Password
              </Button>
            </Section>

            <Text className="text-[#6b7280] text-[13px] leading-[22px]">
              If you did not request a password reset, please ignore this email or reply to let us know. This link is only valid for the next 24 hours.
            </Text>

            <Hr className="border border-solid border-[#e5e7eb] my-[26px] mx-0 w-full" />
            
            <Text className="text-[#9ca3af] text-[12px] leading-[18px]">
              &copy; 2026 Fieldglass App. All rights reserved. <br />
              This is an automated security notification.
            </Text>
          </Container>
        </Body>
      </Tailwind>
    </Html>
  );
};
```

---

## 5. 安全中转站：Auth Callback

这是 Next.js 中最关键的路由处理程序。它负责将邮件里的 `code` 变成浏览器 Cookie。

**文件路径**: `app/auth/callback/route.ts`

```typescript
export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get('code');
  const next = searchParams.get('next') ?? '/';

  if (code) {
    const supabase = await createClient(); // server client
    const { error } = await supabase.auth.exchangeCodeForSession(code);
    if (!error) {
      return NextResponse.redirect(`${origin}${next}`);
    }
  }
  return NextResponse.redirect(`${origin}/login?error=invalid_token`);
}

```


# 6. 最终步骤：更新密码页面

当用户通过 Callback 跳转到 `/update-password` 时，他们已经处于“已登录”状态，可以直接调用 `updateUser`。

```tsx
const onSubmit = async (values: z.infer<typeof formSchema>) => {
  const { error } = await supabase.auth.updateUser({
    password: values.password,
  });

  if (error) {
    toast.error(error.message);
  } else {
    toast.success("Password updated!");
    router.push("/login");
  }
};

```

---

# 7. 常见坑点排查

1. **收到的邮件链接还是 localhost？** 检查 Supabase 后台的 Site URL 是否已改为生产域名，并确保 `redirectTo` 在白名单内。
2. **点击链接后没登录？** 检查 `auth/callback` 路由是否正确执行了 `exchangeCodeForSession`。
3. **Resend 发送失败？** 确保你的发件域名在 Resend 后台已通过 DNS 验证。

---

# 结语

通过这套组合拳，我们不仅实现了功能，还通过 **React Email** 提升了用户体验，通过 **Resend** 保证了邮件的送达率。这才是生产环境该有的样子。
