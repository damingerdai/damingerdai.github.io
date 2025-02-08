---
title: Spring Security 中基于用户信息的动态密码加密
date: 2025-02-08 15:48:53
tags: [Spring Security]
---

# Spring Security 中基于用户信息的动态密码加密

## 前言

通过自定义 Spring Security 的 DaoAuthenticationProvider，可以实现根据用户信息的不同，动态切换不同的密码加密方式。这种方法允许您根据用户的角色、组或其他属性，选择最合适的加密算法，从而提高安全性或满足特定的业务需求。

## DaoAuthenticationProvider

DaoAuthenticationProvider是用于实现用户名密码的登陆AuthenticationProvider，通过自定义实现additionalAuthenticationChecks方法可以实现从UserDetails获取一个字段从而动态获取密码加密服务。

## 实现

自定义DaoAuthenticationProvider：

```java
@Component
public class HotelerAuthenticationProvider extends DaoAuthenticationProvider implements AuthenticationProvider, ApplicationContextAware {

    private ApplicationContext applicationContext;

    protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
        if (authentication.getCredentials() == null) {
            this.logger.debug("Failed to authenticate since no credentials provided");
            throw new BadCredentialsException(this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
        }
        var user =  (User)userDetails;
        var passwordType = CommonUtils.isNotEmpty(user.getPasswordType()) ? user.getPasswordType() : "noop";
        var passwordService = this.getPasswordService(passwordType);
        var presentedPassword = authentication.getCredentials().toString();
        if (!user.getPassword().equals(passwordService.encodePassword(presentedPassword))) {
            this.logger.debug("Failed to authenticate since password does not match stored value");
            throw new BadCredentialsException(this.messages.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));

        }
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    protected void doAfterPropertiesSet() {
        super.doAfterPropertiesSet();
        Assert.notNull(this.applicationContext, "A applicationContext must be set");
    }

    protected IPasswordService getPasswordService(String passwordType) {
        return Objects.requireNonNull(this.applicationContext).getBean(passwordType + "PasswordService", IPasswordService.class);
    }

    public HotelerAuthenticationProvider(UserDetailsService userDetailsService) {
        super();
        this.setUserDetailsService(userDetailsService);
    }
}
```

将HotelerAuthenticationProvider注入到AuthenticationManager中：

```java
@Bean
public AuthenticationManager authenticationManager(HotelerAuthenticationProvider authenticationProvider) {;
    ProviderManager pm = new ProviderManager(authenticationProvider);

    return pm;
}
```