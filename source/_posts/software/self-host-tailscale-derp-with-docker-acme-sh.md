---
title: Docker + acme.sh 搭建 Tailscale 自建 DERP 中继节点完整记录
date: 2026-07-05 15:22:32
tags: [docker, acme.sh, tailsacle, 自建derp]
categories: [软件]
---

# Docker + acme.sh 搭建 Tailscale 自建 DERP 中继节点完整记录

## 1. 方案架构

为了实现证书自动申请、续期与 DERP 服务无缝衔接，采用 Docker Compose 进行单机编排。`acme-sh` 容器负责通过 Cloudflare DNS 验证申请 Let's Encrypt 证书并导出为 DERP 所需的格式；`derper` 容器挂载相同的证书目录并运行中继服务。

```
[Cloudflare DNS-01 验证]
       │
       ▼
 ┌─────────── derp-acme (容器) ───────────┐
 │ 使用 CF_Token 申请证书，并将其安装转换为  │
 │ derp.example.com.crt / .key             │
 └───────────────────┬────────────────────┘
                     │ (共享挂载 ./certs)
                     ▼
 ┌────────── tailscale-derp (容器) ────────┐
 │ 监听 9443(TCP) 与 13478(UDP)           │
 │ 自动加载证书并提供中继服务                │
 └────────────────────────────────────────┘

```

---

## 2. 部署步骤

### 新建工作目录并配置 `.env` 文件

在 `/opt/derp`（或自定义目录）下新建 `.env` 文件，写入环境变量。

> **注意**：Cloudflare API Token 必须具备 **Zone.DNS:Edit** 权限，否则 `acme.sh` 无法在域名下添加用于验证的 TXT 记录。

```env
# 域名与 Token
DERP_DOMAIN=derp.example.com
CF_TOKEN=你的_Cloudflare_DNS_编辑权限_Token

# 自定义服务端口
DERP_TCP_PORT=9443
DERP_UDP_PORT=13478

```

### 编写 `docker-compose.yml`

配置两个服务。`acme-sh` 在首次启动时若检测到无证书文件，则触发申请并使用 `--install-cert` 转换导出，随后运行 `crond` 用于每日自动续期检测。

```yaml
services:
  acme-sh:
    image: neilpang/acme.sh:latest
    container_name: derp-acme
    restart: always
    environment:
      - CF_Token=${CF_TOKEN}
    volumes:
      - ./certs:/acme.sh
    entrypoint: >
      sh -c "
      if [ ! -f /acme.sh/${DERP_DOMAIN}_ecc/${DERP_DOMAIN}.crt ]; then
        acme.sh --issue --dns dns_cf -d ${DERP_DOMAIN} --server letsencrypt;
        mkdir -p /acme.sh/${DERP_DOMAIN}_ecc;
        acme.sh --install-cert -d ${DERP_DOMAIN} --cert-file /acme.sh/${DERP_DOMAIN}_ecc/${DERP_DOMAIN}.crt --key-file /acme.sh/${DERP_DOMAIN}_ecc/${DERP_DOMAIN}.key;
      fi;
      exec crond -f"

  derper:
    image: fredliang/derper:latest
    container_name: tailscale-derp
    restart: always
    ports:
      - "${DERP_TCP_PORT}:${DERP_TCP_PORT}"
      - "${DERP_UDP_PORT}:${DERP_UDP_PORT}/udp"
    environment:
      - DERP_DOMAIN=${DERP_DOMAIN}
      - DERP_CERT_MODE=manual
      - DERP_CERT_DIR=/app/certs/${DERP_DOMAIN}_ecc
      - DERP_ADDR=:${DERP_TCP_PORT}
      - DERP_HTTP_PORT=-1
      - DERP_STUN_PORT=${DERP_UDP_PORT}
      - DERP_VERIFY_CLIENTS=false
    volumes:
      - ./certs:/app/certs
    depends_on:
      - acme-sh

```

### 启动服务

清理历史缓存目录并启动容器：

```bash
docker compose down && rm -rf ./certs/* && docker compose up -d

```

确认 `derp-acme` 输出证书安装成功日志：

```text
derp-acme  | Your cert is in: /acme.sh/derp.example.com_ecc/derp.example.com.cer
derp-acme  | Installing cert to: /acme.sh/derp.example.com_ecc/derp.example.com.crt

```

确认 `tailscale-derp` 成功加载证书并开始监听：

```text
tailscale-derp  | derper: serving on :9443 with TLS

```

---

## 3. Tailscale 控制台 ACL 配置

1. 登录 Tailscale 管理后台，进入 **Access Controls** 页面。
2. 将编辑器切换至 **JSON editor**。
3. 在根对象内追加 `derpMap` 配置，指定自定义 Region ID（如 `901`）及对应端口：

```json
{
  "acls": [
    { "action": "accept", "src": ["*"], "dst": ["*:*"] }
  ],
  "derpMap": {
    "OmitDefaultRegions": false,
    "Regions": {
      "901": {
        "RegionID":   901,
        "RegionCode": "hk-custom",
        "RegionName": "Hong Kong Custom DERP",
        "Nodes": [
          {
            "Name":             "hk-01",
            "RegionID":         901,
            "HostName":         "derp.example.com",
            "DERPPort":         9443,
            "STUNPort":         13478,
            "InsecureForTests": false
          }
        ]
      }
    }
  }
}

```

保存配置，等待节点策略同步。

---

## 4. 状态验证与故障排查

### 客户端网络检查

在本地设备终端运行：

```bash
tailscale netcheck

```

控制台输出回显确认 `hk-custom` 节点已被识别，且延迟正常：

```text
Report:
	* UDP: true
	* IPv4: yes
	* Nearest DERP: Hong Kong Custom DERP
	* DERP latency:
		- hk-custom: 60.1ms   (Hong Kong Custom DERP)
		- tok: 230.7ms        (Tokyo)

```

### 故障记录：远端节点单向失联

**现象**：
本地测试 `tailscale ping <对端IP>` 依然绕道旧节点（如 `tok`），且响应时间过长。
运行 `tailscale status` 查看对端状态：

```text
100.123.186.96  earzer  linux  active; relay "tok", tx 7644 rx 0

```

本地有流量发送（`tx`），但未收到任何对端回包（`rx 0`），表明远端 Linux 设备未刷新最新的 DERP 路由图，仍向旧中继节点发送握手包。

**解决方法**：
登录该远端 Linux 设备，执行命令强制重置并刷新本地的 Tailscale 状态与中继映射缓存：

```bash
sudo tailscale down
sudo tailscale up --reset

```

重置后，两端节点均能正常通过自建的 `hk-custom` 节点进行双向中继通信。

## 引用

1. 