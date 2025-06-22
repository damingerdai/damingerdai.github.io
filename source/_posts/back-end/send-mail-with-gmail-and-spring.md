---
title: 在spring中使用gmail发送邮件
date: 2025-06-22 17:16:53
tags:
---

# 申请gmail的应用专用密码

首先你需要一个gmail账号，然后才能申请gmail的应用专用密码。

想要申请应用专用密码必须开启两步验证：

![开启两步验证页面](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/back-end/gmail-two-step-vertify.png)

google似乎默认并不想让用户直接创建应用专用密码，需要通过右上角搜索框输入“应用专用密码”才能进入相关页面。如果使用英文页面则搜索“App Password”。

![搜索应用专用密码](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/back-end/search-app-password-gmail.png)

然后你就可以创建“应用专用密码”

![创建“应用专用密码”](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/back-end/create-app-password-in-gmail.png)

保存密码，gmail仅仅会显示这一次。

# 使用spring发送邮件

这里实际使用的是spring boot。

导入*spring-boot-starter-mail*:

对于maven用户：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
    <version>${spring-boot-version}</version>
</dependency>
```

对于gradle用户：

```groovy
implementation 'org.springframework.boot:spring-boot-starter-mail:${spring-boot-version}'
```

最后可以使用JavaMailSenderImpl发送邮件了；

```java
 dvar message = javaMailSender.createMimeMessage();
var helper = new MimeMessageHelper(message, true);
helper.setTo("mingguobin@live.com");
helper.setSubject("batcher support");
helper.setText("hello, this is batcher");
javaMailSender.send(message);
```

# 参考

1. [使用应用专用密码登录](https://support.google.com/accounts/answer/185833)
2. [Gmail with an App Password](https://shallowsky.com/blog/tech/email/gmail-app-passwds.html)
3. [Guide to Spring Email](https://www.baeldung.com/spring-email)
4. [使用 Google 作为邮件服务器从 Spring Boot 应用程序发送邮件的技巧](https://medium.com/tuanhdotnet/tips-for-sending-mail-from-a-spring-boot-application-using-google-as-mail-server-fcf5ab042594)