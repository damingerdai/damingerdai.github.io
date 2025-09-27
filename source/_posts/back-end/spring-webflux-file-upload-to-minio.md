---
title: Spring WebFlux 文件上传：从本地存储到 MinIO 集成 🚀
date: 2025-09-27 16:10:25
tags: [gradle, spring boot, spring webflux, 文件上传]
categories: [后端]
---

# 前言

在现代 Web 应用中，高效可靠的文件存储是必不可少的一环。在 Spring WebFlux 响应式编程模型下，处理文件上传需要特别注意流（Flux）和非阻塞操作。

本文将基于一个 Spring WebFlux 的文件上传接口，演示如何从最初的本地文件系统存储方案，平滑地迁移并集成到 MinIO 对象存储服务，实现更具可扩展性和稳定性的文件存储。

# 🎯 方案切换：为何选择 MinIO？

最初的文件上传方案通常是将文件直接存储在服务器的本地文件系统。对于生产环境或需要横向扩展的应用，MinIO（或其他兼容 S3 协议的对象存储）是更优的选择，它提供了高可用、高扩展性和可靠性强的优势，同时解耦了应用和存储。

# 🛠️ 第一步：部署 MinIO 服务

在本地开发或测试环境中，最便捷的部署 MinIO 的方式是使用 Docker Compose。

将以下配置保存为 docker-compose.yaml 文件：
```yaml
services:
  minio:
    image: minio/minio:RELEASE.2025-04-22T22-12-26Z
    container_name: minio_server
    restart: unless-stopped
    ports:
      # 映射 MinIO API 端口
      - "19000:9000"
      # 映射 MinIO Console/UI 端口
      - "19001:9001"
    environment:
      # 保持与 application.yaml 中配置一致
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: 12345678
    volumes:
      # 将 MinIO 数据持久化到本地目录
      - ./minio_data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 5s
      retries: 3
```

运行以下命令启动 MinIO 服务：

```bash
docker compose up -d
```

服务启动后，您可以通过浏览器访问 http://localhost:19001 进入 MinIO 控制台。

# ⚙️ 第二步：引入依赖与配置连接

## 引入 MinIO 依赖

在 build.gradle 中引入 MinIO Java 客户端：

```gradle
dependencies {
    // ... 其他依赖
    implementation 'io.minio:minio:8.5.17' 
    // ...
}
```
## 配置 Spring 应用

在 application.yaml 中配置 MinIO 的连接信息，确保这里的配置与 Docker Compose 中的 environment 变量保持一致。

```yaml
minio:
  # 注意：这里配置的是 MinIO API 映射出来的端口 19000
  endpoint: http://192.168.31.220:19000
  access: admin
  secret: 12345678
  bucket: batcher-dev
```

定义MinioProps获取 application.yaml 中配置 MinIO 的连接信息。

```java
@Component
@ConfigurationProperties(prefix = "minio")
public class MinioProps {

    private String endpoint;

    private String access;

    private String secret;

    private String bucket;

    get/set method...
}
```

自定义MinioConfig配置类将MinioClient注入spring bean中。

```java
@Configuration
public class MinioConfig {

    private Logger logger = LoggerFactory.getLogger(MinioConfig.class);

    private MinioProps minioProperties;

    @Bean(name = "minioClient")
    public MinioClient minioClient() throws Exception {
        this.logger.debug("---------- load minio client ----------");
        MinioClient minioClient = MinioClient.builder()
                .endpoint(this.minioProperties.getEndpoint())
                .credentials(this.minioProperties.getAccess(), this.minioProperties.getSecret())
                .build();
        boolean isExist = minioClient.bucketExists(BucketExistsArgs.builder().bucket(minioProperties.getBucket()).build());
        if (isExist) {
            logger.warn("---------- minio client bucket is existed ----------");
        } else {
            minioClient.makeBucket(MakeBucketArgs.builder().bucket(this.minioProperties.getBucket()).build());
            logger.debug("---------- minio client bucket is created ----------");
        }
        logger.debug("---------- minio client is loaded ----------");
        return minioClient;
    }

    @Autowired
    public void setMinioProperties(MinioProps minioProperties) {
        this.minioProperties = minioProperties;
    }
}
```

# 💻 第三步：重构文件上传控制器

核心变动在于 FileController，我们将注入 MinioClient 和 MinioProps，并重写 uploadFile 方法，将文件流导向 MinIO。

## 注入 MinIO 客户端

通过构造器注入 MinioClient，以便在上传方法中使用

```java
@RestController
@Tag(name = "File Upload", description = "APIs for uploading files")
public class FileController {

    private final MinioClient minioClient;
    private final MinioProps minioProps;

    public FileController(MinioClient minioClient, MinioProps minioProps) {
        this.minioClient = minioClient;
        this.minioProps = minioProps;
    }
    // ...
}
```

## 实现 MinIO 响应式上传逻辑

利用 Reactor 将 WebFlux 的 Flux<DataBuffer> 文件流转换为 MinIO 需要的 InputStream，并包装 MinIO 的阻塞调用以保持非阻塞性。

```java
@PostMapping("/file")
// ... 省略 Swagger 注解
public Mono<String> uploadFile(@RequestPart("file") Mono<FilePart> filePartMono) {
    var bucketName = minioProps.getBucket();

    return filePartMono.flatMap(filePart -> {
        var fileName = filePart.filename();
        var contentType = Objects.requireNonNull(filePart.headers().getContentType()).toString();

        // 1. 检查并创建 MinIO 存储桶（包装阻塞操作）
        return Mono.fromCallable(() -> {
            boolean found = minioClient.bucketExists(BucketExistsArgs.builder().bucket(bucketName).build());
            if (!found) {
                minioClient.makeBucket(MakeBucketArgs.builder().bucket(bucketName).build());
            }
            return true;
        }).flatMap(result -> 
            // 2. 将 Flux<DataBuffer> 转换为 List<InputStream>
            filePart.content()
                    .map(DataBuffer::asInputStream)
                    .collectList()
                    // 3. 将 List<InputStream> 合并为单个 SequenceInputStream 并执行上传
                    .flatMap(inputStreams -> {
                        try (var combinedInputStream = new SequenceInputStream(Collections.enumeration(inputStreams))) {
                            // 使用 -1 告诉 MinIO 客户端进行分块上传
                            var res = minioClient.putObject(PutObjectArgs.builder()
                                    .bucket(bucketName)
                                    .object(fileName)
                                    .stream(combinedInputStream, -1, 10485760) 
                                    .contentType(contentType)
                                    .build());
                            return Mono.just(res.etag());
                        } catch (Exception e) {
                            return Mono.error(new RuntimeException("MinIO upload failed: " + e.getMessage(), e));
                        }
                    })
        );
    });
}
```

# 总结

通过在 Spring WebFlux 中集成 MinIO，我们成功地将文件上传从易受限制的本地文件系统迁移到了健壮、可扩展的对象存储平台。这种方法利用了 Reactor 的响应式流处理能力，同时优雅地处理了 MinIO 客户端的阻塞 I/O 操作，实现了高效且非阻塞的文件上传链路。