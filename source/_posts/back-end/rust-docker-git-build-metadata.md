---
title: 在 Docker 构建 Rust 项目时注入 Git 版本信息
date: 2026-07-29 13:46:50
tags: [rust, docker, git, vergen, vergen_git2]
categories: [后端]
---

# 在 Docker 构建 Rust 项目时注入 Git 版本信息

在开发 Rust 服务时，我们经常希望程序能够输出自身的构建信息，例如：

- 当前版本对应的 Git commit；
- Git 分支；
- 最近的 tag 或 `git describe` 结果；
- commit 的提交时间；
- Rust 编译器和 Cargo 版本；
- 程序的构建时间。

这些信息对于排查线上问题很有帮助。看到某个容器正在运行时，我们可以快速确认它究竟由哪一次提交构建，而不必仅依赖容易被重复使用的 Docker tag。

本文记录我在 Rust 项目中使用 `vergen-git2` 生成构建信息，并在 Docker 构建阶段通过 build arguments 注入 Git 元数据的过程。

## 一、为什么 Docker 构建时可能读取不到 Git 信息

在本地直接执行：

```bash
cargo build --release
```

构建脚本通常可以沿着当前目录找到 `.git`，因此 `vergen-git2` 能直接读取 commit、branch 和 tag 等信息。

但是 Docker 构建并不一定能看到 `.git`。常见原因包括：

1. `.dockerignore` 中排除了 `.git`；
2. Dockerfile 只复制了 `Cargo.toml`、`Cargo.lock`、`build.rs` 和 `src`；
3. 构建使用的是源码压缩包，而不是完整的 Git 仓库；
4. CI 平台使用浅克隆，某些 tag 或历史记录不可用。

当然，也可以直接把 `.git` 复制进构建上下文，但这会增加上下文大小，降低缓存效率，还可能把不必要的仓库信息带入构建过程。

我的处理方式是：

1. 在宿主机上通过 Git 命令取得版本信息；
2. 使用 `docker build --build-arg` 将其传给 Docker；
3. Dockerfile 把 build arguments 转成构建阶段的环境变量；
4. `build.rs` 将环境变量写入最终的 Rust 二进制文件；
5. 如果 Git 信息不可用，则使用 `unknown`，保证构建不会失败。

## 二、使用 `vergen-git2` 生成构建信息

项目使用 `vergen-git2` 收集构建、Cargo、Git、Rust 编译器和系统信息。一个完整的 `build.rs` 可以写成：

```rust
use std::error::Error;
use vergen_git2::{Build, Cargo, Emitter, Git2, Rustc, Sysinfo};

const GIT_KEYS: &[&str] = &[
    "VERGEN_GIT_DESCRIBE",
    "VERGEN_GIT_SHA",
    "VERGEN_GIT_COMMIT_DATE",
    "VERGEN_GIT_BRANCH",
];

fn main() -> Result<(), Box<dyn Error>> {
    println!("cargo:rerun-if-changed=build.rs");

    // 默认值以及外部环境变量变更检测。
    for key in GIT_KEYS {
        println!("cargo:rerun-if-env-changed={key}");
        println!("cargo:rustc-env={key}=unknown");
    }

    let mut emitter = Emitter::default();

    // 无法读取 .git 时，不阻止项目继续构建。
    let _ = emitter.add_instructions(&Build::all_build());
    let _ = emitter.add_instructions(&Cargo::all_cargo());
    let _ = emitter.add_instructions(&Git2::all_git());
    let _ = emitter.add_instructions(&Rustc::all_rustc());
    let _ = emitter.add_instructions(&Sysinfo::all_sysinfo());
    let _ = emitter.emit();

    // Docker build arguments 或外部环境变量拥有最高优先级。
    for key in GIT_KEYS {
        if let Ok(value) = std::env::var(key) {
            if !value.trim().is_empty() {
                println!("cargo:rustc-env={key}={value}");
            }
        }
    }

    Ok(())
}
```

这里有三层数据来源，优先级从低到高依次为：

1. `unknown` 默认值；
2. `vergen-git2` 从当前 Git 仓库读取的值；
3. Docker 或外部环境变量注入的值。

同一个 `cargo:rustc-env` 变量可能被输出多次，最终以后输出的值为准。因此，即使 Docker 构建环境里不存在 `.git`，宿主机传入的值仍然可以覆盖默认值。

`cargo:rerun-if-env-changed` 也很重要。它告诉 Cargo：当这些环境变量发生变化时，需要重新运行构建脚本。否则在复用 Cargo 缓存时，新的 Git commit 信息可能不会进入二进制文件。

## 三、编写多阶段 Dockerfile

Dockerfile 使用 Rust 官方镜像完成编译，再把二进制文件复制到更精简的 Debian 运行时镜像中：

```dockerfile
FROM rust:1.97.1-slim-trixie AS builder

WORKDIR /app

# Git metadata injected at build time.
# Defaults keep the build working when the caller provides no Git information.
ARG VERGEN_GIT_DESCRIBE=unknown
ARG VERGEN_GIT_SHA=unknown
ARG VERGEN_GIT_COMMIT_DATE=unknown
ARG VERGEN_GIT_BRANCH=unknown

ENV VERGEN_GIT_DESCRIBE=$VERGEN_GIT_DESCRIBE \
    VERGEN_GIT_SHA=$VERGEN_GIT_SHA \
    VERGEN_GIT_COMMIT_DATE=$VERGEN_GIT_COMMIT_DATE \
    VERGEN_GIT_BRANCH=$VERGEN_GIT_BRANCH

COPY Cargo.toml Cargo.lock build.rs ./
COPY src ./src

RUN cargo build --release

FROM debian:trixie-slim

WORKDIR /app

COPY --from=builder /app/target/release/rammal /app/rammal

EXPOSE 8000

ENTRYPOINT ["/app/rammal"]
```

这里选择 `rust:1.97.1-slim-trixie` 作为 builder，并使用 `debian:trixie-slim` 作为运行时。二者都基于 Debian Trixie，能够减少因为 glibc 版本不一致导致的运行时兼容问题。

需要注意的是，`ARG` 只在 Docker 构建阶段存在。把它们赋给 `ENV` 后，执行 `cargo build` 的进程才能通过 `std::env::var` 读取这些值。

这些环境变量只定义在 `builder` 阶段，不会自动进入最终的 Debian 镜像。Git 信息已经在编译过程中写入二进制文件，因此运行容器时也不再需要这些环境变量。

## 四、编写统一的构建脚本

为了避免每次手动填写 build arguments，我编写了一个 `build.sh`：

```bash
#!/usr/bin/env bash

set -euo pipefail

IMAGE_NAME="damingerdai/rammal"
IMAGE_TAG="${1:-latest}"

echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"

GIT_DESCRIBE=$(git describe --always --dirty 2>/dev/null || echo "unknown")
GIT_SHA=$(git rev-parse HEAD 2>/dev/null || echo "unknown")
GIT_COMMIT_DATE=$(git log -1 --format=%cI 2>/dev/null || echo "unknown")
GIT_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")

# Detached HEAD is common in CI, so an empty branch is also treated as unknown.
GIT_BRANCH="${GIT_BRANCH:-unknown}"

docker build \
  --build-arg VERGEN_GIT_DESCRIBE="${GIT_DESCRIBE}" \
  --build-arg VERGEN_GIT_SHA="${GIT_SHA}" \
  --build-arg VERGEN_GIT_COMMIT_DATE="${GIT_COMMIT_DATE}" \
  --build-arg VERGEN_GIT_BRANCH="${GIT_BRANCH}" \
  -t "${IMAGE_NAME}:${IMAGE_TAG}" \
  .

echo "Build completed:"
docker images "${IMAGE_NAME}:${IMAGE_TAG}"
```

给脚本添加执行权限：

```bash
chmod +x build.sh
```

不传参数时，镜像 tag 默认为 `latest`：

```bash
./build.sh
```

也可以指定 tag：

```bash
./build.sh v2026.729.0
```

最终构建出的镜像分别是：

```text
damingerdai/rammal:latest
```

或者：

```text
damingerdai/rammal:v2026.729.0
```

## 五、在 Rust 程序中读取构建信息

`build.rs` 通过 `cargo:rustc-env` 写入的值，可以在 Rust 源码中使用 `env!` 或 `option_env!` 读取。

如果已经确保 `build.rs` 总会提供默认值，可以直接使用：

```rust
pub struct VersionInfo {
    pub git_describe: &'static str,
    pub git_sha: &'static str,
    pub git_commit_date: &'static str,
    pub git_branch: &'static str,
}

pub fn version_info() -> VersionInfo {
    VersionInfo {
        git_describe: env!("VERGEN_GIT_DESCRIBE"),
        git_sha: env!("VERGEN_GIT_SHA"),
        git_commit_date: env!("VERGEN_GIT_COMMIT_DATE"),
        git_branch: env!("VERGEN_GIT_BRANCH"),
    }
}
```

如果希望源码脱离 `build.rs` 后仍能编译，可以使用更保守的 `option_env!`：

```rust
const GIT_SHA: &str = option_env!("VERGEN_GIT_SHA").unwrap_or("unknown");
```

这些信息可以展示在：

- `--version` 命令输出中；
- `/version` 或 `/health` 接口中；
- 程序启动日志中；
- 管理后台的系统信息页面中。

例如启动时输出：

```rust
tracing::info!(
    git_sha = env!("VERGEN_GIT_SHA"),
    git_describe = env!("VERGEN_GIT_DESCRIBE"),
    git_commit_date = env!("VERGEN_GIT_COMMIT_DATE"),
    git_branch = env!("VERGEN_GIT_BRANCH"),
    "starting rammal"
);
```

## 六、几个容易忽略的问题

### 1. `fail_on_error()` 和忽略错误不能同时表达严格失败

如果调用了：

```rust
emitter.fail_on_error();
```

却又使用：

```rust
let _ = emitter.emit();
```

那么错误依然被调用方忽略了。既然本文的目标是让缺少 `.git` 的 Docker 构建继续完成，就不必启用严格失败模式。

如果希望只有 Git 信息可以失败、其他构建信息必须成功，最好把 Git 和非 Git 信息分开处理，而不是统一忽略所有结果。

### 2. Docker build arguments 会影响缓存

Git SHA 每次提交都会变化，因此相关构建层的缓存会失效。这是必要的：如果继续使用旧缓存，最终二进制文件里记录的可能是上一次提交。

当前 Dockerfile 在 `cargo build` 之前声明并使用这些参数，因此 Git 信息变化会使编译层重新执行。

### 3. `--dirty` 只能说明工作区发生过修改

`git describe --always --dirty` 可能得到类似：

```text
v1.2.0-3-g1a2b3c4-dirty
```

其中 `dirty` 只能说明构建时工作区存在未提交修改，并不能记录具体修改内容。因此，使用脏工作区构建出来的镜像不一定能仅凭 commit 完整复现。

正式发布镜像时，可以增加检查：

```bash
if ! git diff --quiet || ! git diff --cached --quiet; then
  echo "Refusing to build a release image from a dirty working tree."
  exit 1
fi
```

是否禁止 dirty build，可以根据开发镜像和发布镜像的用途决定。

### 4. CI 中可能处于 detached HEAD

CI 经常检出某个具体 commit，而不是停留在本地分支上。这时：

```bash
git branch --show-current
```

可能返回空字符串，所以脚本需要将空值转换为 `unknown`，或者优先读取 CI 平台提供的分支环境变量。

### 5. 不要把密钥作为 build arguments 传递

Git SHA、branch 和 commit date 不属于敏感信息，适合通过 build arguments 注入。但密码、token 和私钥不应该使用这种方式传递，因为它们可能出现在镜像历史、构建日志或缓存元数据中。

## 七、可以继续做的优化

当前方案已经能够稳定构建。如果项目后续构建时间变长，还可以继续进行以下优化：

1. 使用 `cargo-chef` 缓存依赖编译结果；
2. 使用 BuildKit cache mount 缓存 Cargo registry 和 `target`；
3. 使用 `docker buildx` 同时构建 `linux/amd64` 和 `linux/arm64`；
4. 给镜像增加 OCI labels，记录 revision、source 和创建时间；
5. 在 CI 中同时用 Git SHA 和语义化版本创建镜像 tag。

例如可以给最终镜像添加 revision label：

```dockerfile
ARG VERGEN_GIT_SHA=unknown

LABEL org.opencontainers.image.revision=$VERGEN_GIT_SHA
```

不过 Docker 的多阶段构建中，`ARG` 的作用域以阶段为单位。如果要在最终阶段使用它，需要在最终的 `FROM` 之后重新声明对应的 `ARG`。

## 总结

这套方案的关键不是强行让 Docker 构建环境读取 `.git`，而是把职责拆开：

- 宿主机或 CI 负责读取 Git 仓库；
- `build.sh` 负责传递 Git 元数据；
- Dockerfile 负责提供编译环境；
- `build.rs` 负责把元数据写入 Rust 二进制；
- `unknown` 作为兜底，确保源码包和非 Git 环境仍然可以构建。

这样既不需要把整个 `.git` 目录复制进 Docker 构建上下文，也能让每个运行中的二进制文件明确对应到具体的源码版本。对于需要部署和长期维护的 Rust 服务来说，这是一项成本很低、但非常实用的可观测性改进。
