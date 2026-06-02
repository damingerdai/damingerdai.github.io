---
title: 🚀 记一次本地 Homelab 环境 Gitea 升级迁移与 Actions (CI/CD) 完美集成实战
date: 2026-06-02 21:46:09
tags: [git, gitea, gitea action, docker, CI/CD]
categories: [软件]
---

# 🚀 记一次本地 Homelab 环境 Gitea 升级迁移与 Actions (CI/CD) 完美集成实战

## 📌 前言

最近对本地 Homelab 服务器的私有代码托管平台 **Gitea** 进行了一次大版本升级迁移（升级至 `v1.26`）。原本以为只是一次简单的 `docker compose up`，没想到中间经历了**数据库连错库**、**旧运行器残留**、以及 **Gitea Actions (Act Runner) 容器网络隔离**等一连串经典的暗坑。

本文将完整记录这次迁移过程、踩坑原因以及最终如何完美集成兼容 GitHub Actions 语法的 **Gitea Actions** 并成功跑通 **Go 自动化编译** 的全案。

---

## 🛠️ 第一阶段：大版本迁移与数据库踩坑

在新的服务器上，我准备好了原有的数据卷 `./data` 并编写了新的 `docker-compose.yaml`。然而初次启动后，发现网页端一片空白，**原有的代码仓库、组织和用户全都不见了！**

### 🔍 踩坑原因

检查 `docker-compose.yaml` 发现，在环境变量中误指定了一个全新的数据库名：

```yaml
- GITEA__database__NAME=gitea2 # 👈 罪魁祸首

```

由于原本的 PostgreSQL 中旧数据存在于 `gitea` 库中，指定为 `gitea2` 导致 Gitea 在启动时自动初始化了一套全新的空白表。

### 💡 解决方案

1. 执行 `docker compose down` 停止服务。
2. 对齐旧环境，将数据库名改回原本正确的名字（例如 `gitea`）。
3. 重新启动后，从日志中成功看到大版本所需的迁移脚本自动执行，历史仓库完美回归！

---

## 🏃 第二阶段：绕过 CLI 限制，数据库直击“软删除”

由于是迁移环境，进入 **站点管理 -> Actions -> Runners** 后，发现列表里赫然躺着一个“上次在线时间为去年”的离线旧 Runner（ID 为 3）。

由于跨版本升级导致前端 UI 样式适配问题，网页端的删除按钮无法正常点击。如果尝试使用官方的 `docker compose exec gitea gitea admin runners unregister` 命令，又会高频触发各种执行环境或权限报错。

### 💡 解决方案（硬核外挂）

Gitea 底层对运行器采用了标准的**软删除（Soft Delete）**设计。与其在被权限限制死死的容器命令里打转，不如直接连入 PostgreSQL 数据库，把 `action_runner` 表中该记录的 `deleted` 字段（`bigint` 类型）变更为**当前的时间戳**：

```sql
-- 连入 Postgres 数据库执行，直接对 ID=3 的旧运行器进行软删除
UPDATE action_runner SET deleted = extract(epoch from now())::bigint WHERE id = 3;

```

执行完毕后刷新网页，这个去年的残留旧 Runner 瞬间在前端彻底消失，清爽！

---

## 🚀 第三阶段：集成 Gitea Actions (Act Runner)

Gitea 本身只负责调度，干苦力跑流水线需要额外部署 **Act Runner**。在集成的过程中，有几个事关本地局域网（`192.168.31.x`）能否跑通的关键深坑。

### 🙅 避坑指南与硬核操作

1. **官方超大镜像国内拉不动：** 默认生成的 `config.yaml` 中，`ubuntu-latest` 映射的是 `docker.gitea.com` 的官方大镜像。直接拉取极易因超时导致流水线挂掉。
* **🔥 我的硬核解法：** 不改动 `config.yaml` 的默认配置。直接利用华为云的 DDN 加速代理将镜像曲线拉下，再通过 `tag` 重新打标伪装成官方域名，让 Runner 直接吃宿主机本地缓存！
```bash
# 利用加速通道拉取
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/gitea/runner-images:ubuntu-latest
# 重新打标伪装成官方域名
docker tag swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/gitea/runner-images:ubuntu-latest docker.gitea.com/runner-images:ubuntu-latest

```




2. **网络隔离 Connection Refused：** 如果将 `config.yaml` 中的 `container.network` 留空，Act Runner 启动的临时任务容器会自动加入 Docker 的默认 `bridge` 网络，导致其无法反向汇报日志。

### 📝 终极配置文件展示

#### 1. 优化后的 `config.yaml`

找到 `container:` 下的 `network:` 这一行，显式指定和你的 `docker-compose.yaml` 里面完全一样的网络名字，打通网络：

```yaml
runner:
  # 保持默认的官方标签映射即可，因为我们已经在宿主机通过 docker tag 完成了“偷天换日”
  labels:
    - "ubuntu-latest:docker://docker.gitea.com/runner-images:ubuntu-latest"

container:
  # 👈 核心：让流水线的临时容器加入到宿主机的 Compose 网络中，拒绝网络隔离
  network: "gitea_v2_gitea" 

```

#### 2. 健壮的 `docker-compose.yaml`

将 `GITEA_INSTANCE_URL` 换成**宿主机物理局域网 IP**，彻底规避容器初始化阶段 IP 漂移引发的断连：

```yaml
services:
  gitea:
    image: docker.io/gitea/gitea:1.26
    container_name: gitea
    environment:
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=host.docker.internal:5432
      - GITEA__database__NAME=gitea
      - GITEA__actions__ENABLED=true # 👈 开启 Actions 总开关
    # ... 其他保持原样 ...

  runner:
    image: gitea/act_runner:0.6.0
    container_name: gitea_runner
    restart: always
    depends_on:
      - gitea
    environment:
      CONFIG_FILE: /config.yaml
      # 👈 改用宿主机直连 IP 绕过 Docker 内部握手死锁
      GITEA_INSTANCE_URL: http://192.168.31.220:13000 
      GITEA_RUNNER_REGISTRATION_TOKEN: <Your token>
      GITEA_RUNNER_NAME: "damingerdai gitea runner"
    volumes:
      - ./config.yaml:/config.yaml
      - ./runner-data:/data
      - ./runner_data/cache:/root/.cache
      - /var/run/docker.sock:/var/run/docker.sock # 👈 允许调用宿主机 Docker
    networks:
      - gitea

```

修改完成后，执行清空并重启三板斧：

```bash
docker compose down
sudo rm -f ./runner-data/.runner # 删掉旧凭证强制触发新配置注册
docker compose up -d

```

回到网页端，新 Runner 的**常亮绿灯 (Idle)** 瞬间激活！

---

## 🐹 第四阶段：编写 Go 自动化编译工作流 (Workflow)

环境打通后，在代码仓库根目录下创建 `.gitea/workflows/go-ci.yaml`：

```yaml
name: Gitea Go 自动化编译测试
run-name: ${{ gitea.actor }} 正在测试 Go 编译 🚀

on: [push]

jobs:
  standalone-go-test:
    runs-on: 'ubuntu-latest' # 👈 精确匹配刚才在 config.yaml 里配的标签
    steps:
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Create Test Main.go
        run: |
          cat << 'EOF' > main.go
          package main
          import "fmt"
          func main() {
              fmt.Println("Hello Gitea Actions! This is a compiled Go binary!")
          }
          EOF

      - name: Compile Go Code
        run: go build -o main main.go

      - name: Run Compiled Binary
        run: ./main

```

### 🎉 成果验收

代码 `git push` 上去后，流水线秒级唤醒，在刚才伪装成功的 `ubuntu-latest` 容器环境内，最后的编译产物成功打印：

```text
Hello Gitea Actions! This is a compiled Go binary!

```

至此，一套完全纯血、本地化、高效率的自建私有化 CI/CD 环境宣告完美落成！

---

## 💡 总结与反思

1. **直击底层的思维：** 当官方 CLI 在特殊容器环境遇到阻碍时，理解其数据库的软删除机制（修改 `deleted` 时间戳字段），直接对数据库下手往往效率最高。
2. **巧妙避坑：** 面对国内环境拉不动超大外网镜像的难题，利用 **DDN 代理拉取 + 本地 docker tag 重新打标伪装** 是一种对业务配置零侵入、极度优雅且高效的破局方案。