---
title: 使用docker-compose备份Postgres Docker容器的解决方案
date: 2024-07-10 17:18:24
tags:
---

# 使用docker-compose备份Postgres Docker容器的解决方案

## 备份

使用`pg_dumpall`命令备份Postgres数据库。

```bash
docker-compose exec <postgres_service> pg_dumpall -U postgres > dump_`date +%Y-%m-%d"_"%H_%M_%S`.sql
```

1. `docker-compose exec <postgres_service>` 在名为`<postgres_service>`的Postgres容器上执行命令。
2. `-U postgres` 指定数据库的用户名。Docker的默认用户名是`postgres`，如果你使用不同的用户名，请进行修改。

## 恢复

将`dump_`date +%Y-%m-%d"_"%H_%M_%S`.sql`文件放置在`backup`文件夹中。

然后使用Docker卷将`backup`文件夹绑定到Postgres容器上：

```bash
volumes:
    - ./backup:/backup
```

删除现有的Postgres容器并创建一个新的容器：

```bash
docker-compose down && docker-compose up -d
```

执行数据库导入命令：

```bash
docker-compose exec postgres psql -f /backup/dump_xxx.sql postgres -U postgres
```

## 参考资料

1. [使用docker-compose备份和恢复Postgres数据库](https://wdt.im/posts/docker-compose-postgres-backup-restore/)

