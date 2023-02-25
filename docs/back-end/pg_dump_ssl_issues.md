# 记录使用pg_dump中遇到的ssl问题

## 问题描述

在使用`pg_dump`过程中遇到的两个问题：

1. SSL connection is required. Please specify SSL options and retry.
2. SSL error: unsafe legacy renegotiation disabled


##  SSL option问题

这是因为postgresql需要使用TLS/SSL connections，我们可以设置一个环境变量`PGSSLMODE`为`required`就可以了。

针对Linux或者Macos用户，使用:
```bash
export PGSSLMODE=require
```
针对Windows用户，使用:
```bash
SET PGSSLMODE=require
```

## unsafe legacy renegotiation disabled

这是因为ssl协议存在一个漏洞，后来升级的时候提供了一个开关默认关闭这个了漏洞，这个解决方法存在一定安全隐患，请谨慎使用。

创建一个自定义的`openssl.cnf`,并写入：

```ini
openssl_conf = openssl_init

[openssl_init]
ssl_conf = ssl_sect

[ssl_sect]
system_default = system_default_sect

[system_default_sect]
Options = UnsafeLegacyRenegotiation
```

将该config文件的路径写入名为`OPENSSL_CONF`的环境变量：

```bash
export OPENSSL_CONF=/path/to/custom/openssl.cnf
```

## 参考资料

1. [Migrate your PostgreSQL database by using dump and restore](https://learn.microsoft.com/en-us/azure/postgresql/migrate/how-to-migrate-using-dump-and-restore)
2. [SSL error unsafe legacy renegotiation disabled](https://stackoverflow.com/questions/71603314/ssl-error-unsafe-legacy-renegotiation-disabled)