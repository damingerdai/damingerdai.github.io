---
title: 寻找轻量级 S3 替代方案：极简分布式对象存储 Garage 搭建与全线打通指南
date: 2026-07-12 20:49:12
tags: [s3, garage]
categories: [软件]
---

# 寻找轻量级 S3 替代方案：极简分布式对象存储 Garage 搭建与全线打通指南

在自建基础设施、HomeLab 或微型云环境时，MinIO 通常是大家首选的 S3 兼容对象存储。然而，[MinIO](github.com/minio/minio)于2026 年 2 月 12 日宣布不再维护，2026 年 4 月 25 日正式归档为只读状态

最近关注到一篇优秀的技术分享 _[Garage: The Minimalist Distributed Object Store — Your Lightweight S3 Alternative](https://blog-ocampoge.medium.com/garage-the-minimalist-distributed-object-store-your-lightweight-s3-alternative-b7ca8be162b0)_，文中极力推荐了 **Garage**。这是一个用 Rust 编写的轻量级、去中心化分布式对象存储，专为不规则、低带宽的多节点网络设计，单个节点跑起来甚至只需要几十兆内存，可以说是将“极简主义”贯彻到了极致。

本文将基于最新的 **Garage v2.3.0**，详细记录如何通过 Docker Compose 快速搭建单节点集群，并分享在部署以及后续使用 SDK 进行客户端测试时遇到的**四大核心深坑**与避坑姿势。

---

## 1. 环境准备与架构设计

根据极简主义的设计原则，我们采用 Docker Compose 进行微服务化部署.

### 核心配置文件

由于 Garage 容器镜像内部运行环境极致轻量化（甚至没有默认 shell），我们需要将宿主机的配置文件、元数据目录以及数据存储目录通过 Volume 挂载进去。

#### `docker-compose.yml`

```yaml
services:
  garage:
    image: dxflrs/garage:v2.3.0
    container_name: garage
    restart: unless-stopped
    ports:
      - "3900:3900" # S3 API 端点
      - "3901:3901" # RPC 集群内通讯端口
      - "3902:3902" # S3 Web 静态托管
      - "3903:3903" # Admin API 管理端点
    volumes:
      - ./garage.toml:/etc/garage.toml:ro
      - ./meta:/var/lib/garage/meta
      - ./data:/var/lib/garage/data
```

#### `garage.toml`

```toml
metadata_dir = "/var/lib/garage/meta"
data_dir = "/var/lib/garage/data"
db_engine = "sqlite"
replication_factor = 1

# RPC 相关配置
rpc_bind_addr = "[::]:3901"
# 🔥 必须是严格的 32 字节（64位十六进制字符）强随机密钥
rpc_secret = "f07d2a5b6e8f9c1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a"

[s3_api]
s3_region = "garage"
api_bind_addr = "[::]:3900"

[s3_web]
bind_addr = "[::]:3902"
root_domain = ".web.garage"

[admin]
api_bind_addr = "[::]:3903"
admin_token = "your_admin_token_here"

```

---

## 2. 单节点集群初始化三部曲

因为 Garage 本质上是一个“分布式”架构，哪怕我们是单机单节点运行，也必须显式宣告其拓扑结构（Cluster Layout）并为其分配磁盘容量，否则服务无法建立任何 S3 Key 或 Bucket。

### 第一步：角色与容量指派（Staging Layout）

首先查看状态获取当前节点的 `Node ID`，随后指派该节点。

```bash
# 查看并获取 Node ID
docker compose exec garage /garage status

# 指派角色（以 Node ID 为 cf1675b039dc847d 为例，指定 10G 额度）
docker compose exec garage /garage layout assign cf1675b039dc847d -t node1 -z zone1 -c 10G

```

### 第二步：使拓扑结构落盘生效

运行以下命令激活刚才暂存（Staged）的拓扑改动。Garage 会自动计算 256 个哈希数据分区（Partitions）：

```bash
docker compose exec garage /garage layout apply --version 1

```

### 第三步：生成 S3 凭证

拓扑激活后，终于可以爽快地创建 S3 的 Access Key 和 Secret Key 了：

```bash
docker compose exec garage /garage key create my-key

```

控制台会直接打印出你的 `Key ID` 和 `Secret key`。

---

## 3. 核心踩坑与全链路调优记录（重难点）

在从零到彻底打通的过程中，我们极易遇到以下四个官方文档不会明说的“隐形坑”：

### 坑一：`rpc_secret` 严格的长度和格式校验

如果图省事随便写一个普通字符串密码作为 `rpc_secret`，容器会直接崩溃报错。Garage 强制要求该密钥必须通过高强度随机算法生成，且满足 **32 字节（即 64 位 Hex 字符）**。

> **正确生成姿势：** 在宿主机运行 `openssl rand -hex 32`，将其作为密匙填入即可。

### 坑二：极简镜像找不到 `garage` 命令执行文件

当尝试运行 `docker compose exec garage garage status` 时，会报 `executable file not found`。
这是因为镜像为了极致轻量，没有将二进制文件塞入 `/usr/bin`。我们需要**显式指定根目录绝对路径**进行交互：

```bash
docker compose exec garage /garage status

```

### 坑三：v2.x 拓扑阶段容量参数（Capacity）要求严格单位

在指派 Layout 时，如果传递 `-c 1` 想代表权重，会触发 `Capacity should be at least 1K` 的错误。在新版 Garage 中，**`-c` 必须是合规的容量单位格式（如 `10G`, `1T`），且至少大于 `1K (1024 bytes)**`。

### 坑四：新旧版本 key 权限子命令变更

许多旧博客引用的 `key allow-create` 在最新的 **v2.3.0** 中已经废弃。新版本中统一收敛到了 **`key allow`** 子命令下：

```bash
# 赋予全局建桶权限
docker compose exec garage /garage key allow <YOUR_KEY_ID> --create-bucket

```

### 坑五：桶已经建立，但客户端依然报 403 AccessDenied 错误

当我们在后台使用管理员命令创建了一个桶（`bucket create test-bucket`）后，通过 Go 语言或客户端上传文件时可能会遇到以下报错：

> `api error AccessDenied: Forbidden: Operation is not allowed for this key.`

这是 Garage 非常严谨的**最小权限隔离设计**。即使你的 Key 拥有全局建桶的权限（`Can create buckets: true`），但对于已经存在的桶（尤其是管理员建立的桶），它默认处于无权访问的状态。**必须显式进行桶级别的授权**：

```bash
docker compose exec garage /garage bucket allow test-bucket --key <YOUR_KEY_ID> --read --write

```

当看到控制台的 `KEYS FOR THIS BUCKET` 权限表刷新出 `RW` 标志时，才代表该 Key 真正打通了对该存储桶的读写全通路。

---

## 4. Golang 访问与全链路验证测试

为了验证服务彻底可用，我们可以写一段简单的 Go 脚本来进行连接、上传、下载的闭环测试。

这里使用官方的 **AWS SDK for Go v2**。由于本地没有配置通配符域名解析，初始化客户端时**必须指定 `UsePathStyle: true` 启用路径样式路由**。

此外需要注意，在官方 SDK 中，上传对象参数的正确结构体名称是 **`s3.PutObjectInput`**（手写时容易误写成 `s3.Input` 导致编译失败）。

#### `main.go`

```go
package main

import (
	"context"
	"bytes"
	"fmt"
	"io"
	"log"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/credentials"
	"github.com/aws/aws-sdk-go-v2/service/s3"
)

func main() {
	endpoint := "http://localhost:3900"
	region := "garage"
	accessKey := "你的AccessKey"
	secretKey := "你的SecretKey"
	bucketName := "test-bucket"

	cfg, err := config.LoadDefaultConfig(context.TODO(),
		config.WithRegion(region),
		config.WithCredentialsProvider(credentials.NewStaticCredentialsProvider(accessKey, secretKey, "")),
	)
	if err != nil {
		log.Fatalf("无法加载 SDK 配置: %v", err)
	}

	s3Client := s3.NewFromConfig(cfg, func(o *s3.Options) {
		o.BaseEndpoint = aws.String(endpoint)
		o.UsePathStyle = true // 必须开启 Path-Style
	})

	fmt.Println("成功连接到 Garage 对象存储...")

	// 上传对象验证
	objectKey := "hello-garage.txt"
	content := []byte("Hello Arthur, Garage S3 is working perfectly with Go!")

	_, err = s3Client.PutObject(context.TODO(), &s3.PutObjectInput{
		Bucket: aws.String(bucketName),
		Key:    aws.String(objectKey),
		Body:   bytes.NewReader(content),
	})
	if err != nil {
		log.Fatalf("文件上传失败: %v", err)
	}
	fmt.Printf("成功上传对象: %s\n", objectKey)

	// 下载对象验证
	resp, err := s3Client.GetObject(context.TODO(), &s3.GetObjectInput{
		Bucket: aws.String(bucketName),
		Key:    aws.String(objectKey),
	})
	if err != nil {
		log.Fatalf("文件下载失败: %v", err)
	}
	defer resp.Body.Close()

	downloadedContent, err := io.ReadAll(resp.Body)
	if err != nil {
		log.Fatalf("读取下载内容失败: %v", err)
	}

	fmt.Printf("成功下载对象，内容为:\n> %s\n", string(downloadedContent))
}

```

---

## 5. 总结

比起传统的 MinIO，Garage 极其精简的 Rust 运行时带来了惊人的低开销和极高的敏捷度。虽然在部署和多层级权限划分时有一些严谨的限制（也就是我们一路上踩过的坑），但只要理清了 **Layout 指派 -> 生效 -> 全局 Key 赋权 -> 存储桶粒度赋权** 这条链路，它在轻量级私有云和 HomeLab 环境中绝对能成为一件完美的 Minimalist 艺术品。

---
