--- 
title: Docker 把我宿主机网络"吃"了：一次 172.18 网段冲突的完整复盘
date: 2026-07-07 14:33:21
tags: [Docker 网络冲突]
categories: [软件]
---

# Docker 把我宿主机网络"吃"了：一次 172.18 网段冲突的完整复盘

> **摘要**：本文记录了一起由 Docker 自定义网络自动分配网段引发的宿主机网络故障。现象是宿主机无法访问内网某台服务器，报错 `From 172.18.0.1 Destination Host Unreachable`。最终定位到是 docker-compose 中的 `pg-network` 抢占了 `172.18.0.0/16`，并给出了不丢数据、不卸载 Docker 的最终解决方案。

---

## 一、问题现象

某台物理服务器（**宿主机 A**），需要访问内网另一台服务器（**目标机器 B**）。

执行 ping 命令：

```bash
ping 172.18.1.10
```

返回结果：

```
From 172.18.0.1 icmp_seq=1 Destination Host Unreachable
```

**令人困惑的点：**
- ✅ 目标机器在线（其他机器可正常访问）
- ✅ 交换机、VLAN 配置近期未变更
- ✅ 防火墙策略已排查，无异常

**更诡异的是：**
- ❌ 停止 `docker.service` 后问题依旧
- ❌ 停止 `docker.socket` 后问题依旧

仿佛 Docker 已经"阴魂不散"地占据了网络通道……

---

## 二、排查过程（关键线索）

### 1️⃣ 检查宿主机网卡信息

执行命令：

```bash
ip addr show
```

发现一个**可疑接口**：

```
5664: br-9159e3498b65: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-9159e3498b65
```

⚠️ **注意两个关键点：**
- `172.18.0.1/16` 正好覆盖了目标地址 `172.18.1.10` 所在的网段
- 这是一个 **Linux bridge**，不是物理网卡

### 2️⃣ 查看路由表验证

执行命令：

```bash
ip route
```

存在这条路由：

```
172.18.0.0/16 dev br-9159e3498b65 scope link src 172.18.0.1
```

**Linux 内核路由优先级规则：**
```
直连路由 > 默认路由
```

因此访问流程变成了：
1. 访问 `172.18.1.10`
2. 内核匹配到 `172.18.0.0/16` 直连路由
3. 认为目标在本地 bridge `br-9159e3498b65` 上
4. 实际 bridge 中不存在该 IP
5. 返回 `Destination Host Unreachable`

### 3️⃣ 确认 bridge 归属

执行命令：

```bash
docker network ls
```

输出结果：

```
9159e3498b65   postgresql_pg-network   bridge    local
```

**Docker 命名规则：**
```
br-<NETWORK_ID_PREFIX>
```

✅ **实锤确认**：`br-9159e3498b65` 就是 Docker 为 `pg-network` 创建的虚拟网桥。

---

## 三、根因分析（核心教训）

问题出在这个 **docker-compose.yaml** 配置：

```yaml
networks:
  pg-network:
    external: false
```

**致命缺陷：**
- ❌ 没有指定 `ipam` 配置
- ❌ 没有指定 `subnet` 子网段
- ✅ Docker 自动从 `172.17.0.0/16` 起始的网段中挑选
- ✅ 本次自动选中了 `172.18.0.0/16`
- ✅ 恰好覆盖了物理网络中的 `172.18.1.0/24` 网段

**结果：**
Docker 的虚拟网桥"鸠占鹊巢"，宿主机访问物理网段时被错误引流到虚拟 bridge，导致网络中断。

---

## 四、为什么停止 Docker 服务没用？

很多人（包括当时的我）会本能地执行：

```bash
systemctl stop docker
```

但 Docker 的行为逻辑是：

| 操作 | 是否清理 bridge | 是否清理路由 |
|------|----------------|-------------|
| `systemctl stop docker` | ❌ 不会 | ❌ 不会 |
| `systemctl stop docker.socket` | ❌ 不会 | ❌ 不会 |
| `docker compose down` | ✅ 会 | ✅ 会 |

**原因：**
- `docker.service` 停止 ≠ 删除已创建的 bridge
- `docker.socket` 仍在监听
- **自定义 bridge 网络默认不会自动删除**
- 路由表条目仍然存在，内核继续使用

**直到执行：**
```bash
docker compose down
```
bridge 才会被清理，路由表才会恢复。

---

## 五、数据安全澄清（非常重要）

在整个排查过程中，我始终担心一个问题：

> **会不会误删 PostgreSQL 数据？**

**答案是：不会。**

| 操作 | 是否影响容器数据 | 说明 |
|------|----------------|------|
| `docker compose down` | ❌ 不影响 | 仅停止容器，不删除 volumes |
| `ip link delete br-*` | ❌ 不影响 | 仅删除虚拟网桥 |
| `ip route del` | ❌ 不影响 | 仅删除路由条目 |
| 修改 `daemon.json` | ❌ 不影响 | 仅修改 Docker 配置 |
| 重启 Docker | ❌ 不影响 | 容器数据保存在 volumes 中 |

**⚠️ 唯一危险的操作：**
```bash
rm -rf /var/lib/docker   # 千万不要执行！
```
而我们**从未执行**它。

✅ PostgreSQL 数据目录完好无损（如 `./pg-data`）  
✅ 其他 volume 数据全部完好

---

## 六、最终解决方案（推荐）

针对这类问题，我提供**两种解决方案**，你可以根据实际情况选择：

---

### 方案一：项目级修复（推荐用于单个项目）

修改 `docker-compose.yaml`，为当前项目显式指定 subnet。

#### ✅ 修复 docker-compose.yaml

**修改前：**
```yaml
networks:
  pg-network:
    external: false
```

**修改后：**
```yaml
networks:
  pg-network:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.211.0/24
```

#### ✅ 执行修复流程

```bash
# 1. 停止 compose（不删数据）
docker compose down

# 2. 清理残留 bridge（如果还存在）
sudo ip link set br-9159e3498b65 down
sudo ip link delete br-9159e3498b65
sudo ip route del 172.18.0.0/16

# 3. 重新启动
docker compose up -d
```

---

### 方案二：全局禁用风险网段（生产级推荐） ⭐

> **这是 Docker 官方支持的方案，专门解决"企业内网大量使用 172.16–172.31"的场景。**

#### 🔧 操作步骤

**1. 编辑 Docker 配置文件**

```bash
sudo vim /etc/docker/daemon.json
```

如果文件不存在，直接创建即可。

**2. 写入配置**

```json
{
  "default-address-pools": [
    { "base": "192.168.200.0/20", "size": 24 }
  ]
}
```

**配置说明：**
- `base`: 指定 Docker 自动分配网段的**总池子**
- `size`: 每个 Docker 网络分配的子网掩码

**实际效果：**
```
192.168.200.0/20  →  包含 192.168.200.0 ~ 192.168.215.255
                    →  可划分为 16 个 /24 子网
                    →  Docker 自动从这些子网中分配
```

**3. 重启 Docker 服务**

```bash
sudo systemctl restart docker
```

**4. 验证生效**

```bash
docker network ls
docker network inspect <network_name> | grep Subnet
```

---

### 📊 两种方案对比

| 对比维度 | 方案一：项目级修复 | 方案二：全局配置 ⭐ |
|---------|------------------|-------------------|
| **适用范围** | 单个项目 | 所有 Docker 网络 |
| **配置位置** | docker-compose.yaml | /etc/docker/daemon.json |
| **是否需要修改项目** | ✅ 需要 | ❌ 不需要 |
| **一次性配置** | ❌ 每个项目都要改 | ✅ 全局生效 |
| **适合场景** | 已上线的项目快速修复 | 新环境/DevOps 标准化 |
| **维护成本** | 高（项目多时重复劳动） | 低（一劳永逸） |

**我的建议：**
- **紧急修复** → 先用方案一
- **长期治理** → 落地方案二，后续所有项目自动规避风险网段

---

### ✅ 避坑指南：如何选择安全的 subnet

#### ❌ 绝对避免的网段

| 网段 | 原因 |
|------|------|
| `172.16.0.0/12` | Docker 默认首选，企业内网重灾区 |
| `192.168.0.0/24` | 家用路由器默认，极易冲突 |
| `192.168.1.0/24` | 同左，最常用的默认网段 |
| `10.0.0.0/8` | 部分企业内部使用，需谨慎 |

#### ✅ 推荐使用的网段

| 方案 | 网段 | 说明 |
|------|------|------|
| **项目级** | `10.200.0.0/24` | 10.x 中不常用的子段 |
| **项目级** | `192.168.200.0/24` | 避开 0、1 这些常见段 |
| **全局配置** | `192.168.200.0/20` | 可划分 16 个 /24 子网，足够使用 |
| **全局配置** | `10.233.0.0/16` | Kubernetes 常用，避开主流段 |

#### 💡 全局配置的高级用法

如果企业内网同时使用了多个网段，可以这样配置：

```json
{
  "default-address-pools": [
    { "base": "192.168.200.0/20", "size": 24 },
    { "base": "10.233.0.0/16", "size": 24 }
  ]
}
```

Docker 会按照配置顺序依次分配，第一个池子用完后再用第二个。

---

### 验证结果

#### 方案一验证

```bash
# 测试网络连通性
ping 172.18.1.10

# 检查 Docker 网络配置
docker network inspect postgresql_pg-network | grep Subnet

# 检查路由表
ip route | grep 172.18
```

**预期结果：**
- ✅ `ping` 通目标机器
- ✅ subnet 变为 `192.168.211.0/24`
- ✅ 不再存在 `172.18.0.0/16` 路由

#### 方案二验证

```bash
# 创建一个测试网络
docker network create test-net

# 查看分配的 subnet
docker network inspect test-net | grep Subnet

# 清理测试网络
docker network rm test-net
```

**预期结果：**
- ✅ 分配的子网在 `192.168.200.0/20` 范围内
- ✅ 不再出现 `172.17.0.0/16`、`172.18.0.0/16` 等网段

---

## 七、最佳实践总结（经验沉淀）

### ✅ Docker 网络配置三原则

1. **永远不要相信 Docker 的自动网段分配**
   - 开发环境可能侥幸避开，生产环境必踩坑

2. **生产环境必须显式声明 subnet（项目级）或配置全局地址池**
   - 不要给 Docker "自由发挥"的空间
   - 方案二更适合 DevOps 标准化环境

3. **避开 172.16.0.0/12 和 192.168.0.0/24 等常见冲突段**
   - 这些是企业内网和 Docker 默认的"兵家必争之地"

### ✅ 容器内访问宿主机的最佳方式

**❌ 不推荐：**
```yaml
extra_hosts:
  - "host:172.18.1.10"   # 硬编码 IP，换环境就挂
```

**✅ 推荐：**
```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

容器内统一使用：
```
host.docker.internal
```

### ✅ 排查 Docker 网络问题的黄金命令

```bash
# 查看网卡接口
ip addr show

# 查看路由表
ip route

# 列出 Docker 网络
docker network ls

# 查看网络详情
docker network inspect <network>

# 查看 bridge 详细信息
bridge link

# 查看 Docker 全局配置
docker info | grep -A 5 "Default Address Pools"
```

---

## 八、写在最后

这次故障让我深刻意识到：

> **Docker 的"自动化"在开发环境是便利，在生产环境是陷阱。**

尤其是：
- 企业内网大量使用 `172.16.0.0/12` 网段
- Docker 默认也偏爱这个区间
- 一旦冲突，宿主机网络直接被静默劫持，毫无预警

**两个层面的教训：**

1. **项目层面**：每个 compose 项目都要显式声明网络配置
2. **基础设施层面**：在 `daemon.json` 中全局禁用风险网段，一劳永逸

**如果你也在用 Docker Compose 管理数据库、中间件，现在就去检查你的配置：**

```bash
# 检查当前 Docker 网络分配情况
docker network ls | xargs -I {} docker network inspect {} | grep -E "Subnet|Name"
```

别等到 `ping` 不通的时候，才想起这篇博客。


---

*📅 记录于 2026 年 7 月*  
*🔧 希望这篇文章能帮你避免一次深夜 OnCall*