---
title: 📝 PostgreSQL 18.x Docker 镜像 PGDATA 路径变更说明
date: 2025-10-04 19:28:35
tags: [docker, docker compose, 容器, postgres, postgres18]
categories: [软件]
---

# 📝 PostgreSQL 18.x Docker 镜像 PGDATA 路径变更说明

本文档旨在记录 PostgreSQL 官方 Docker 镜像自版本 18.0 起，其默认数据目录（PGDATA）路径的变化，并指导如何正确配置卷挂载以实现数据持久化。

# 🚀 核心变更：PGDATA 路径版本化

从 PostgreSQL 18.x 开始，官方 Docker 镜像将数据库的数据存储位置 (PGDATA 环境变量) 更改为版本特定的路径。

---

## PostgreSQL PGDATA 路径变更

| PostgreSQL 版本 |   旧版本 PGDATA 默认路径   |    18.x 版本 PGDATA 默认路径    |
| :-------------: | :------------------------: | :-----------------------------: |
| **17.x 及以前** | `/var/lib/postgresql/data` |               N/A               |
| **18.x 及以后** |            N/A             | `/var/lib/postgresql/18/docker` |

## 🔍 变更原因

采用版本化的 PGDATA 路径有以下重要意义：

1. 便于升级 (pg_upgrade): 允许用户将不同主版本的数据库数据目录（例如 /var/lib/postgresql/17/docker 和 /var/lib/postgresql/18/docker）并行挂载到父目录 /var/lib/postgresql 下，从而更有效地利用 pg_upgrade 等工具进行版本间升级。
2. 确保数据持久性: 在重新创建或升级容器时，通过将卷准确挂载到版本特定的路径，确保了数据的正确持久化。

# 💾 数据持久化配置

当使用 PostgreSQL 18.x 官方镜像时，必须将您的持久化卷（无论是命名卷还是绑定挂载）挂载到新的 `PGDATA` 路径：`/var/lib/postgresql/18/docker`。

## 使用 Docker Compose

在您的 docker-compose.yaml 文件中，应将卷配置指向新的路径：

```yaml
services:
  postgres:
    image: postgres:18.0-alpine3.22
    container_name: postgres
    # ... 其他配置 ...
    volumes:
      # ⚠️ 必须指向版本特定的 PGDATA 路径
      - pg18-data-volume:/var/lib/postgresql/18/docker

volumes:
  pg18-data-volume:
```

## 使用 Docker Run (绑定挂载示例)

使用 `docker run` 命令进行绑定挂载时，也需要指定新的路径：

```bash
docker run -d \
  -v my_postgres_data:/var/lib/postgresql/18/docker \
  -e POSTGRES_PASSWORD=your_password \
  --name postgres18 \
  postgres:18.0-alpline3.22
```

# ⚠️ 升级注意事项

在从 PostgreSQL 17.x 或更早版本升级到 18.x 时，由于主版本之间的内部存储格式不兼容，不能简单地更改镜像标签并重用旧的 /var/lib/postgresql/data 卷。

正确的主版本升级流程应是：

1. 从旧版本容器中执行 pg_dumpall 创建完整的 SQL 备份。
2. 停止旧容器，创建并启动一个新的 PostgreSQL 18.x 容器，并使用 新路径 挂载卷。
3. 将第 1 步中的 SQL 备份文件导入（psql）到新的 PostgreSQL 18.x 容器中。