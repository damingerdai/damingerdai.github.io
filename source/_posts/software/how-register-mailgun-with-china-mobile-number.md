---
title: 使用中国手机号码验证mailgun账号
date: 2025-08-13 19:38:39
tags: [mailgun]
categories: [软件]
---

# 前言

Mailgun 是一个专为开发者设计的邮件发送平台，支持通过 API 高效、稳定地发送和接收邮件。它广泛用于用户通知、营销邮件、自动化开发信等场景。每个月可免费发送10000封邮件，可以添加1000个域名，每封邮件都有跟踪日志，简单明了的管理界面。

# 注册

点 mailgun.com 右上角的“start for free“进去是不会给免费版选项的，只有 free trial，而 https://www.mailgun.com/pricing/ 页面则有免费 plan，常规注册即可，无需填写信用卡.

# 验证

mailgun会发送邮件要求验证号码。在验证页面会要求用户提供手机号验证。但是对中国手机号码来说，可能存在收不到验证码的可能。

![mailgun验证码失败](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/mailgun-validate-error.png)

# 解决

在mailgun后台，点击右上角的问号❓，点击“Support”按钮。提交一个工单，点击左下方“Open a ticket”的开工单按钮。在开工单页面下拉选择“Account Management”类别，然后输入问题标题和具体内容。

这是我总结的模版：

```md
Subject: Unable to Verify Email Due to Excessive Verification

Content:
I am unable to complete email verification because I encountered an issue after entering my mobile phone number (+86 xxxxxxxxxxx) and requesting a verification code. The system notified me that I have exceeded the allowed number of attempts, along with other unspecified reminders.
```

然后大概在24小时内就能解决。(我是40分钟就解决了)。

# 其他

别相信咸鱼上那帮所谓提供海外手机号码验证服务的那帮垃圾。

# 引用

1. [Mailgun账号注册、域名配置、API开通教程](https://www.jiadingqiang.com/31941.html)
2. [Mailgun China](https://mailgun-china.com/)
3. [国外Mailgun邮件服务账号验证不成功怎么办](https://zhuanlan.zhihu.com/p/685968877)