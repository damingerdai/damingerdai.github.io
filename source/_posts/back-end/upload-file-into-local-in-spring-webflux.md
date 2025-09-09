---
title: 在 Spring WebFlux 中实现文件上传到本地
date: 2025-09-08 22:17:15
tags: [gradle, spring boot, spring webflux, 文件上传]
categories: [后端]
---

# 在 Spring WebFlux 中实现文件上传到本地

## 引言

本文旨在介绍如何在 Spring WebFlux 框架下，实现将文件上传至本地服务器的功能。虽然目前的代码示例侧重于本地存储，但其设计思路也为未来扩展至对象存储服务（如 MinIO）奠定了基础。通过本文的学习，您将掌握在响应式编程模型下处理文件上传的关键技术。

-----

## 具体实现过程

## 1\. 项目搭建与依赖引入

首先，您需要一个基于 Spring Boot 3.5.0 的项目。如果您使用 Gradle 作为构建工具，可以在 `build.gradle` 文件中添加以下依赖：

```gradle
dependencies {
    // ... 其他 Spring Boot 依赖
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    // Swagger UI 依赖
    implementation 'org.springdoc:springdoc-openapi-starter-webflux-ui:2.8.6' // 请检查最新版本
    // ...
}
```

`spring-boot-starter-webflux` 提供了构建响应式 Web 应用所需的核心功能，`springdoc-openapi-starter-webfluxui` 则是集成 Swagger UI 所需的依赖。

## 2\. 编写 Controller

接下来，我们将编写 `FileController` 来处理文件上传请求。

```java
import org.springframework.core.io.buffer.DataBufferUtils;
import org.springframework.http.codec.multipart.FilePart;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestPart;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;

import java.nio.file.Paths;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.io.IOException;

Controller
public class FileController {

    @PostMapping("/file")
    public Mono<String> uploadFile(@RequestPart("file") Mono<FilePart> filePartMono) {
        return filePartMono.flatMap(filePart -> {
            // 获取用户目录
            var userDir = System.getProperty("user.dir");
            // 构建文件保存路径，这里假设保存在项目根目录下的 nfs 文件夹中
            // 您可以根据实际需求修改路径
            Path path = Paths.get(userDir, "nfs", filePart.filename());

            // 确保 nfs 目录存在，如果不存在则创建
            try {
                java.nio.file.Files.createDirectories(path.getParent());
            } catch (IOException e) {
                return Mono.error(new RuntimeException("Failed to create upload directory", e));
            }

            // 使用 transferTo 将文件内容写入到指定路径
            // transferTo 方法返回一个 Mono<Void>，表示操作的完成
            return filePart.transferTo(path)
                    // then 方法用于在前一个 Mono 完成后执行下一个操作
                    // 这里我们返回一个包含成功消息的 Mono<String>
                    .then(Mono.just("File uploaded successfully: " + filePart.filename()));
        });
    }
}
```

**关键点解析：**

  * **`@PostMapping("/file")`**: 指定该方法处理 HTTP POST 请求到 `/file` 路径。
  * **`@RequestPart("file") Mono<FilePart> filePartMono`**:
      * `@RequestPart` 注解用于接收 `multipart/form-data` 请求中的部分。这里的 `"file"` 对应 HTML form 表单中 `<input type="file" name="file">` 的 `name` 属性。
      * `Mono<FilePart>` 表示这是一个异步的、最多包含一个元素的发布者，代表上传的文件。Spring WebFlux 使用 Reactor 库进行响应式编程，`Mono` 是其核心组件之一。
  * **`filePart.transferTo(path)`**:
      * 这是 Spring WebFlux 提供的用于将上传的文件内容 **直接传输** 到指定 `Path` 的方法。
      * 它返回一个 `Mono<Void>`，表示文件传输操作的完成，但不会返回任何数据。在响应式流中，`Void` 表示一个操作的完成信号。
  * **`.then(...)`**:
      * `then()` 操作符用于在源 `Mono`（在这里是 `filePart.transferTo(path)`）完成（发出完成信号）后，执行另一个 `Mono`。
      * `Mono.just("File uploaded successfully: " + filePart.filename())` 创建了一个新的 `Mono`，它会发出一个包含成功消息的字符串。
      * 因此，`.then(...)` 确保了文件传输完成后，再发出成功上传的消息。

## 3\. 使用 cURL 调用

您可以使用 `curl` 命令来测试这个上传接口：

```bash
curl -X 'POST' \
  'http://localhost:8080/file' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'file=@Arthur-Ming-FlowCV-Resume-20241101 (1).pdf;type=application/pdf'
```

-----

## Swagger UI 支持

为了方便 API 的可视化和测试，我们可以为其添加 Swagger UI 支持。

## 1\. 添加依赖

如前所述，在 `build.gradle` 中添加 `springdoc-openapi-starter-webfluxui` 依赖：

```gradle
dependencies {
    // ...
    implementation 'org.springdoc:springdoc-openapi-starter-webflux-ui:2.8.6' // 请检查最新版本
    // ...
}
```

## 2\. 配置 Controller 注解

`springdoc-openapi` 库会自动扫描带有 Spring MVC 或 Spring WebFlux 注解的 Controller。您只需要确保您的 Controller 和方法使用了标准的 Spring WebFlux 注解（如 `@RestController`, `@PostMapping`, `@RequestPart` 等），`springdoc-openapi` 就会自动为您生成 OpenAPI 文档，并在启动后通过访问 `/swagger-ui.html` （或其他默认路径，具体取决于版本和配置）来查看。

```java
@Operation(
    summary = "Upload a single file",
    description = "Uploads a single file to the server's NFS directory.",
    requestBody = @RequestBody(
            description = "File to upload",
            required = true,
            content = @Content(
                    mediaType = "multipart/form-data",
                    schema = @Schema(implementation = FileUploadRequest.class) // Use a custom schema
            )
    ),
    responses = {
            @ApiResponse(responseCode = "200", description = "File uploaded successfully",
                    content = @Content(mediaType = "application/json", schema = @Schema(type = "string"))),
            @ApiResponse(responseCode = "400", description = "Bad Request"),
            @ApiResponse(responseCode = "500", description = "Internal Server Error")
    }
)
```

在项目启动后，访问 `http://localhost:8080/swagger-ui.html` （假设您的应用运行在 8080 端口），您就可以看到文件上传接口的 Swagger UI 界面了。

-----

通过以上步骤，您就成功地在 Spring WebFlux 中实现了一个文件上传到本地的功能，并为其添加了 Swagger UI 支持。您可以基于此进一步扩展，例如添加文件类型校验、大小限制，以及将文件存储目标切换为 MinIO 等对象存储服务。