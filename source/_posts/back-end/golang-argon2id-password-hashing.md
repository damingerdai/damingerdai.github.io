---
title: 从密码哈希迁移到 Argon2id：一次 Go 项目中的密码安全实践
date: 2026-10-08 19:16:53
description: "记录 Health Master 从 MD5 迁移到 Argon2id 的过程，理解 Memory-hard、Salt、密码哈希验证与 Timing Attack，以及 Go 中的工程实践。"
tags:
  - Go
  - Golang
  - Argon2id
  - Cryptography
  - Security
  - Password Hashing
categories: [后端]
---

# 从密码哈希迁移到 Argon2id：一次 Go 项目中的密码安全实践

最近在维护我的开源项目 [Health Master](https://github.com/damingerdai/health-master) 时，我完成了一次密码哈希机制的升级，将原有的 MD5 迁移到 Argon2id。

相关代码可以参考 [PR #333：Migrate password hashing to Argon2id and improve login errors](https://github.com/damingerdai/health-master/pull/333)。

最初的想法其实很简单：MD5 已经不适合存储用户密码了，应该换成更加安全的算法。

但在实际修改过程中，我发现这并不只是一次简单的函数替换。

比如：

- 为什么 MD5 和 SHA-256 都不适合直接存储密码？
- Argon2id 所谓的 Memory-hard 究竟是什么意思？
- 为什么 Argon2id 要把 Salt、算法版本和计算参数一起保存在数据库中？
- 如果攻击者拿到了完整的哈希字符串，为什么不能直接解密密码？
- 为什么 Go 中验证 Hash 时，还要使用 `subtle.ConstantTimeCompare`？
- 已经使用 MD5 存储密码的用户，又应该如何迁移？

为了弄清楚这些问题，我阅读了 [《Argon2id 密码哈希：memory-hard 的工程实践与前沿》](https://bsheepcoder.github.io/2026/07/01/cs-argon2id-explained/)，也重新回顾了陈皓在酷壳发表的 [《计时攻击 Timing Attacks》](https://gaojuqian.github.io/CoolShell/coolshell.cn/articles/21003.html)。

这篇文章记录一下这次学习和实践的过程。

## 一、为什么要从 MD5 迁移？

在早期的软件开发中，MD5 曾经被广泛用于生成数据摘要。

例如，在 Go 中计算一个字符串的 MD5：

```go
package main

import (
    "crypto/md5"
    "encoding/hex"
    "fmt"
)

func main() {
    password := "Hello123!"

    hash := md5.Sum([]byte(password))

    fmt.Println(hex.EncodeToString(hash[:]))
}
```

MD5 会生成一个固定长度的 128-bit 摘要。

从表面上看，使用 MD5 保存密码似乎是安全的：数据库中不再存储明文密码，而且 MD5 也不存在可以直接还原原文的解密函数。

但是，**密码哈希的安全性不能只看算法是否可逆。**

### 1.1 MD5 最大的问题是什么？

对于密码存储来说，MD5 最大的问题之一是计算速度太快。

假设攻击者获得了数据库中的密码哈希：

```text
5d41402abc4b2a76b9719d911017c592
```

攻击者并不需要把哈希逆向还原，而是可以不断猜测密码：

```text
MD5("123456")      → 不匹配
MD5("password")    → 不匹配
MD5("hello")       → 匹配
```

这就是离线密码猜测攻击。

由于 MD5 的计算成本非常低，攻击者可以利用 GPU 等硬件快速尝试大量候选密码。

需要注意，MD5 的碰撞安全问题与这里讨论的密码猜测并不完全相同。即使一个快速哈希算法没有已知的实用碰撞攻击，也不代表它适合直接存储密码。

### 1.2 换成 SHA-256 可以吗？

最初我也考虑过这个问题。

SHA-256 比 MD5 具有更好的密码学安全性质，但如果直接使用：

```go
sha256.Sum256([]byte(password))
```

仍然不能解决密码哈希计算速度过快的问题。

对于文件校验、数字签名等场景，快速计算是优点。

但对于密码存储，快速计算反而降低了离线猜测的成本。

因此，我们需要的并不是一个普通的哈希函数，而是专门为密码存储设计的 Password Hashing Function。

## 二、Argon2id 是什么？

Argon2 是 2015 年 Password Hashing Competition（PHC）的获胜算法，后来被标准化为 [RFC 9106](https://www.rfc-editor.org/rfc/rfc9106.html)。

它的核心设计思想是：

**通过增加计算成本和内存成本，提高攻击者进行大规模密码猜测的代价。**

Argon2 主要有三个变体：

| 算法 | 特点 |
|---|---|
| Argon2d | 数据相关的内存访问模式，侧重抵抗特定硬件上的破解优化 |
| Argon2i | 数据无关的内存访问模式，侧重抵抗侧信道攻击 |
| Argon2id | 结合两种模式，兼顾相关安全需求 |

对于一般的用户密码存储场景，Argon2id 是目前推荐优先考虑的选择。

### 2.1 什么是 Memory-hard？

这是我在学习 Argon2id 时遇到的第一个重要概念。

传统快速哈希算法主要依赖计算能力。

而 Argon2id 除了需要执行一定数量的计算，还需要使用一定规模的内存。

例如：

```text
memory = 64 MiB
iterations = 3
parallelism = 2
```

意味着一次密码哈希计算需要约 64 MiB 的工作内存，并按照指定参数进行计算。

对于普通服务器，这个开销通常可以接受。

但如果攻击者希望同时尝试成千上万个密码，就必须考虑这些计算任务所需要的内存资源和内存带宽。

这就是 Memory-hard 设计的意义。

它并不是让 GPU 无法执行 Argon2id，而是提高攻击者使用 GPU、ASIC 等硬件进行大规模并行破解的成本。

### 2.2 Argon2id 的参数

Argon2id 主要通过三个参数调整计算成本：

```text
m = memory cost
t = time cost
p = parallelism
```

例如：

```text
m=65536,t=3,p=2
```

其中：

- `m=65536`：内存成本为 65536 KiB，即 64 MiB。
- `t=3`：迭代次数为 3。
- `p=2`：并行度为 2，表示两个 lane。

这里有一个容易误解的地方：`p=2` 并不意味着内存成本自动变成 128 MiB。

`m` 指定的是整个 Argon2 计算的内存成本，而不是每个 lane 分别使用 64 MiB。

这些参数并不是越大越好。

因为提高攻击者成本的同时，也会增加正常用户登录时服务器的资源消耗。

实际生产环境应该通过 Benchmark 确定适合自己的参数。

## 三、Salt 和 Hash 都公开了，密码为什么仍然安全？

这是我在学习 Argon2id 时最感兴趣的问题。

一个典型的 Argon2id 密码哈希字符串如下：

```text
$argon2id$v=19$m=65536,t=3,p=2$salt$hash
```

它通常采用 PHC String Format。

把它拆开：

| 字段 | 含义 |
|---|---|
| `argon2id` | 使用的密码哈希算法 |
| `v=19` | Argon2 算法版本 |
| `m=65536` | 内存成本 |
| `t=3` | 迭代次数 |
| `p=2` | 并行度 |
| `salt` | 随机盐，实际为 Base64 编码 |
| `hash` | 密码哈希，实际为 Base64 编码 |

我的第一个疑问是：

**既然算法、参数、Salt 和 Hash 全部都保存在数据库中，攻击者拿到这个字符串后，不就知道密码是怎么计算出来的吗？**

答案是：攻击者确实知道计算方法，但这并不意味着可以反向恢复原始密码。

### 3.1 哈希不是加密

加密和哈希最重要的区别是：

加密是可逆的。拥有正确密钥的一方，可以通过解密算法恢复原始数据。

而密码哈希是单向的。

例如：

```text
password + salt
      ↓
   Argon2id
      ↓
     hash
```

我们可以通过密码和 Salt 计算 Hash，但不存在对应的 Argon2id 解密函数，能够直接从 Hash 还原原始密码。

### 3.2 那么攻击者能做什么？

攻击者可以进行猜测。

例如，数据库中保存：

```text
$argon2id$v=19$m=65536,t=3,p=2$salt$hash
```

攻击者可以使用同样的 Salt 和参数，不断尝试候选密码：

```text
Argon2id("123456", salt)     → 不匹配
Argon2id("password", salt)   → 不匹配
Argon2id("Hello123!", salt)  → 匹配
```

一旦匹配成功，就说明找到了一个可以通过验证的密码。

所以 Argon2id 不能保证密码永远不会被猜中。

它真正提供的是较高的单次猜测成本。

如果用户使用的是 `123456` 这样的弱密码，即使使用 Argon2id，也可能被字典攻击猜中。

### 3.3 为什么 Salt 不需要保密？

Salt 的主要作用不是作为秘密密钥，而是让相同密码在不同情况下生成不同的哈希。

例如两个用户都使用：

```text
Hello123!
```

如果使用不同的 Salt：

```text
Argon2id("Hello123!", saltA) → hashA
Argon2id("Hello123!", saltB) → hashB
```

最终结果就会不同。

这样可以避免攻击者直接根据相同的哈希判断用户是否使用相同密码，也能阻止攻击者高效复用预先计算好的哈希表。

因此，Salt 可以和 Hash 一起存储。

这里体现了密码学中的一个重要思想：**系统的安全性不应该依赖算法本身保密。**

## 四、在 Go 中实现 Argon2id

理解基本原理之后，就可以开始实际编码了。

Go 可以使用 `golang.org/x/crypto/argon2`：

```bash
go get golang.org/x/crypto/argon2
```

其中 `argon2.IDKey` 就是用于计算 Argon2id 的核心函数。

### 4.1 生成密码哈希

下面是一个简化的示例：

```go
package pwd

import (
    "crypto/rand"
    "encoding/base64"
    "fmt"

    "golang.org/x/crypto/argon2"
)

const (
    memory      = 64 * 1024
    iterations  = 3
    parallelism = 2
    saltLength  = 16
    keyLength   = 32
)

func HashPassword(password string) (string, error) {
    salt := make([]byte, saltLength)

    if _, err := rand.Read(salt); err != nil {
        return "", err
    }

    hash := argon2.IDKey(
        []byte(password),
        salt,
        iterations,
        memory,
        parallelism,
        keyLength,
    )

    return fmt.Sprintf(
        "$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
        argon2.Version,
        memory,
        iterations,
        parallelism,
        base64.RawStdEncoding.EncodeToString(salt),
        base64.RawStdEncoding.EncodeToString(hash),
    ), nil
}
```

这段代码主要完成三件事：

第一，使用 `crypto/rand` 生成随机 Salt。

第二，调用 `argon2.IDKey` 计算密码哈希。

第三，将算法版本、计算参数、Salt 和 Hash 编码为 PHC 字符串。

这里使用 `crypto/rand` 非常重要，因为 Salt 应该来自密码学安全的随机数生成器。

### 4.2 验证密码

密码验证的过程则是：

1. 从数据库读取完整的 PHC 字符串。
2. 解析算法、参数、Salt 和 Hash。
3. 使用用户输入的密码重新计算。
4. 比较计算结果与数据库中的 Hash。

核心逻辑如下：

```go
actualHash := argon2.IDKey(
    []byte(password),
    salt,
    iterations,
    memory,
    parallelism,
    uint32(len(expectedHash)),
)

valid := subtle.ConstantTimeCompare(
    actualHash,
    expectedHash,
) == 1
```

这里省略了 PHC 字符串解析和参数校验逻辑。

实际项目中必须校验算法版本、Salt 长度、Hash 长度，以及内存和迭代参数的上下限，不能直接信任解析出来的所有值。

否则，恶意或异常的哈希字符串可能导致服务器分配过多内存，造成拒绝服务风险。

不过，上面的代码还有一个值得继续讨论的地方：

**为什么比较两个 Hash，需要使用 `subtle.ConstantTimeCompare`？**

## 五、从 ConstantTimeCompare 到 Timing Attack

最初看到 Go 的 `crypto/subtle` 时，我并没有特别在意。

毕竟两个 Hash 都是字节数组，直接进行比较似乎就可以了。

但重新阅读陈皓在酷壳发表的 [《计时攻击 Timing Attacks》](https://gaojuqian.github.io/CoolShell/coolshell.cn/articles/21003.html) 后，我意识到普通比较操作也可能存在安全问题。

### 5.1 普通字节比较有什么问题？

假设我们自己实现一个比较函数：

```go
func Equal(a, b []byte) bool {
    if len(a) != len(b) {
        return false
    }

    for i := range a {
        if a[i] != b[i] {
            return false
        }
    }

    return true
}
```

这段代码在逻辑上完全正确。

但是它有一个特点：发现第一个不同的字节后，就立即返回。

例如：

```text
abcdef vs xbcdef
↑
第一个字符不同，立即返回
```

而另一种情况：

```text
abcdef vs abcdxf
             ↑
第五个字符才不同
```

第二次比较需要执行更多循环。

因此，不同输入可能产生微小的执行时间差异。

### 5.2 什么是 Timing Attack？

Timing Attack，即计时攻击，是一种侧信道攻击。

攻击者并不直接破解算法，而是通过观察程序的执行时间，推测内部数据。

例如，在一个使用不安全比较函数验证 HMAC 签名的接口中，攻击者可能通过反复提交不同的签名，并分析响应时间，逐步推测正确签名的前缀。

陈皓的文章讨论了这种利用执行时间差异泄露信息的攻击思路。

这里需要注意，真实网络环境中存在大量噪声，单次请求的时间差异通常并不可靠。

攻击者可能需要大量重复请求和统计分析，而且攻击能否成功还取决于具体实现。

但这足以说明一个问题：

**密码学实现中的微小行为差异，也可能成为信息泄露的来源。**

### 5.3 Go 的 ConstantTimeCompare

Go 标准库提供了：

```go
subtle.ConstantTimeCompare(x, y)
```

它的目标是避免等长字节数组的比较时间依赖于匹配前缀的长度。

例如：

```go
valid := subtle.ConstantTimeCompare(
    actualHash,
    expectedHash,
) == 1
```

与普通的提前返回式比较不同，`ConstantTimeCompare` 不会因为前几个字节不匹配，就跳过后续的比较。

不过，这里有两个需要注意的细节。

首先，`ConstantTimeCompare` 对长度不同的输入会立即返回，因此它并不能保证所有输入情况下执行时间完全相同。

其次，它也不能保证整个登录请求的响应时间恒定。

例如，数据库查询、用户是否存在、Argon2id 计算参数以及其他业务分支，都可能影响认证流程的耗时。

### 5.4 这是否意味着可以通过计时攻击猜出密码？

并不是。

需要区分普通字符串比较与 Argon2id 哈希验证。

Argon2id 具有密码学哈希函数所需的扩散性质，原始密码即使只有一个字符不同，也会产生完全不同的哈希结果。

因此，比较哈希时泄露的前缀信息，并不等同于泄露原始密码的前缀。

使用 `ConstantTimeCompare` 是密码学工程中的防御性实践，不能据此推断使用普通比较的 Argon2id 登录接口就一定存在可利用的逐字符密码恢复漏洞。

这也是这次学习中让我印象深刻的一点：

**密码安全不仅涉及算法本身，还涉及算法周围的实现细节。**

## 六、Health Master 的密码哈希迁移

理解了 Argon2id 的基本原理之后，我开始将它应用到 Health Master 中。

对应的改动记录在 [PR #333](https://github.com/damingerdai/health-master/pull/333)。

这次迁移主要涉及密码处理逻辑、用户认证、历史密码兼容、数据库字段和错误处理。

### 6.1 统一密码处理

这次修改引入了独立的 `pkg/pwd` 包，用于集中管理密码哈希与验证。

主要包括：

```go
HashPassword(...)
VerifyPassword(...)
Authenticate(...)
```

这样做的好处是，注册、登录和密码重置等业务逻辑不需要分别处理底层的密码哈希算法。

后续如果需要调整 Argon2id 参数，也可以集中维护。

### 6.2 旧 MD5 密码怎么办？

这是迁移过程中比较实际的问题。

数据库中已经存在使用 MD5 存储的密码，新的 Argon2id 验证逻辑无法直接验证这些旧哈希。

一种常见方案是：

用户登录时先验证 MD5，成功后再使用 Argon2id 重新计算并更新密码哈希。

这种方式对用户比较友好，但意味着迁移期间仍然需要保留旧 MD5 验证逻辑。

在这次 PR 中，我选择了另一种方式：识别旧 MD5 格式，并要求用户重置密码。

认证逻辑会返回专门的错误：

```go
errcode.ErrLegacyPasswordResetRequired
```

然后由业务层和前端处理这一状态。

这样可以逐步淘汰旧的 MD5 密码，而不需要在正常登录路径中长期保留旧哈希验证能力。

当然，这种方案也有代价：旧用户需要额外完成密码重置流程。

### 6.3 数据库字段长度需要调整

MD5 通常使用 32 个十六进制字符表示。

但 Argon2id 的 PHC 字符串包含算法、参数、Salt 和 Hash，长度明显更长。

因此，这次 PR 也调整了数据库中的 `users.password` 字段：

```sql
character varying(255)
```

这提醒了我，密码哈希算法迁移不只是修改一个函数，还涉及数据存储格式的兼容性。

### 6.4 登录错误处理

迁移后，登录流程还需要正确区分：

- 普通密码验证失败。
- 旧密码格式需要重置。
- 服务端内部错误。

这次 PR 同步调整了相关错误码及登录错误处理。

不过，认证错误信息也需要考虑安全性。

例如，不能让普通登录接口随意暴露用户是否存在，否则可能被用于账号枚举。

对于旧密码重置提示，也需要结合具体认证流程进行评估。

## 七、Argon2id 并不意味着可以忽略其他安全问题

经过这次学习，我认为有必要强调：Argon2id 只是密码安全体系中的一部分。

它主要解决的是数据库密码哈希泄露后，攻击者进行离线密码猜测的成本问题。

但它并不能直接解决：

- 用户使用弱密码。
- 登录接口缺少 Rate Limiting。
- 密码重置流程存在漏洞。
- 数据库权限配置不合理。
- 用户遭遇钓鱼攻击。
- 认证流程存在其他侧信道。

尤其是 Argon2id 的 Memory-hard 特性，也会增加服务器自身的资源消耗。

假设每次密码验证需要 64 MiB 内存，那么大量并发登录请求就可能带来明显的内存压力。

因此，生产环境应该根据服务器配置调整参数，并对认证请求实施合理的并发限制和速率限制。

密码哈希算法的选择，最终还是要结合完整的系统安全设计。

## 八、Argon2id 的性能测试：Memory-hard 到底有多昂贵？

前面提到，Argon2id 的重要特性是 Memory-hard。但仅仅理解算法原理还不够，我希望通过真实的 Benchmark，看看内存成本、迭代次数和并行度分别会带来多少开销，以及并发执行时系统的吞吐量如何变化。

本次测试在 Health Master 的 `pkg/pwd` 包中进行，环境为 macOS、Apple M2 Pro、`darwin/arm64`。每组测试执行三次，下面的汇总数据取三次 `ns/op` 的算术平均值；不同硬件和系统负载下结果可能不同。

### 8.1 单次哈希：Memory、Iterations 与 Parallelism

执行命令：

```bash
go test ./pkg/pwd -run '^$' \
  -bench BenchmarkHashPassword \
  -benchmem -count=3
```

测试代码如下：

```go
func BenchmarkHashPassword(b *testing.B) {
    for _, tc := range []struct {
        name   string
        modify func(*Argon2Params)
    }{
        {name: "Default"},
        {name: "Memory32MiB", modify: func(p *Argon2Params) { p.Memory = 32 * 1024 }},
        {name: "Memory128MiB", modify: func(p *Argon2Params) { p.Memory = 128 * 1024 }},
        {name: "Iterations1", modify: func(p *Argon2Params) { p.Iterations = 1 }},
        {name: "Parallelism1", modify: func(p *Argon2Params) { p.Parallelism = 1 }},
        {name: "Parallelism4", modify: func(p *Argon2Params) { p.Parallelism = 4 }},
    } {
        b.Run(tc.name, func(b *testing.B) {
            var params *Argon2Params
            if tc.modify != nil {
                copy := *DefaultParams
                tc.modify(&copy)
                params = &copy
            }
            b.ReportAllocs()
            for b.Loop() {
                if _, err := HashPassword("benchmark-password", params); err != nil {
                    b.Fatal(err)
                }
            }
        })
    }
}
```

默认配置为 `m=64 MiB, t=3, p=2`。除被测试的参数外，其他参数保持默认值。

| 测试配置 | 平均耗时 | 相对 Default | 每次分配内存（约） |
|---|---:|---:|---:|
| Default | 70.64 ms | 1.00× | 64 MiB |
| Memory32MiB | 32.12 ms | 0.45× | 32 MiB |
| Memory128MiB | 143.83 ms | 2.04× | 128 MiB |
| Iterations1 | 23.75 ms | 0.34× | 64 MiB |
| Parallelism1 | 135.29 ms | 1.92× | 64 MiB |
| Parallelism4 | 35.36 ms | 0.50× | 64 MiB |

**内存成本。** 将 `m` 从 32 MiB 增加到 128 MiB，平均耗时从约 32 ms 增加到 144 ms，累计分配内存也随之增加。这使 Memory-hard 的资源成本变得直观。不过，Go Benchmark 的 `B/op` 是**每次操作累计分配字节数**，不等于进程峰值 RSS。

**迭代次数。** 将 `t` 从 3 降为 1，耗时从约 71 ms 降至 24 ms。降低迭代次数虽然可以减少登录时的计算开销，但也会降低攻击者每次离线猜测所需付出的成本。

**并行度。** 在 Apple M2 Pro 上，`p=1` 的平均耗时约为 135 ms，`p=4` 约为 35 ms。这说明单次哈希能从更高并行度中获益；但这不意味着 `p` 越大，Web 服务的整体吞吐量就一定越高。多个登录请求同时执行时，CPU 和内存带宽仍然是共享资源。

### 8.2 并发 Benchmark：从单次耗时到整体吞吐量

为了进一步观察并发行为，我使用 `RunParallel` 测试默认参数，并调整 `GOMAXPROCS`：

```bash
go test ./pkg/pwd -run '^$' \
  -bench BenchmarkHashPasswordParallel \
  -benchmem -cpu 1,2,4,8 -count=3
```

这里 `-cpu` 设置的是 Go 运行时的 `GOMAXPROCS`，即允许同时执行 Go 代码的逻辑处理器数量，并**不是**模拟固定数量的用户登录请求。

| GOMAXPROCS | 三次测试的 ns/op（ms） | 平均 ns/op（ms） | 估算吞吐量（ops/s） | 相对 1 |
|---|---|---:|---:|---:|
| 1 | 134.30 / 129.37 / 126.14 | 129.94 | 7.70 | 1.00× |
| 2 | 69.57 / 69.30 / 66.35 | 68.41 | 14.62 | 1.90× |
| 4 | 37.77 / 35.82 / 34.27 | 35.95 | 27.81 | 3.61× |
| 8 | 23.45 / 24.43 / 45.72 | 31.20 | 32.05 | 4.16× |

> 表中吞吐量以 `1 / 平均每次操作耗时` 估算。`RunParallel` 的 `ns/op` 是测试总时间除以操作数，更接近吞吐成本，**不能直接当作单个请求的端到端响应延迟**。`B/op` 在各组中约为 64 MiB，表示每次哈希的累计分配量。

从这组结果可以看到：

1. `GOMAXPROCS` 从 1 增加到 4 时，估算吞吐量由约 7.70 ops/s 提高到 27.81 ops/s，扩展效果较明显。
2. 从 4 增加到 8 时，收益变小。增加可并行执行的 CPU 资源，并不会保证吞吐量等比例提升。
3. `GOMAXPROCS=8` 的第三次结果为 **45.72 ms/op**，明显高于前两次的约 23–24 ms/op。这可能与系统调度、后台负载、CPU 功耗或内存带宽竞争有关，但仅凭三次结果无法确定原因。

这一轮测试让我意识到：**单次哈希快，不等于并发登录时每个用户的等待时间短；吞吐量提高，也不等于延迟一定降低。**

### 8.3 Benchmark 对生产环境的启发

Argon2id 的 `m`、`t` 和 `p` 不只是密码学参数，也是服务端的资源预算。

以 64 MiB 工作内存为例，如果有 20 次哈希真正同时执行，Argon2 工作内存理论上就可能达到约 **1.25 GiB**，还不包含 Go 运行时、数据库连接和其他业务逻辑的开销。

因此，不能仅凭 Apple M2 Pro 上约 71 ms 的单次哈希结果，就认定这组参数适合 Health Master 的生产环境。尤其是在资源有限的 Kubernetes 节点中，仍然需要在实际部署机器上测量：

- 固定并发请求数下的吞吐量与 p50/p95/p99 响应延迟；
- CPU 使用率、峰值 RSS 和 Pod 的 Memory Limit；
- 不同 `m`、`t`、`p` 组合下的性能与安全成本；
- 登录限流与哈希计算并发上限是否合理。

目前我会将 `m=64 MiB, t=3, p=2` 保留为**待验证的基准配置**，而不是因为 `p=4` 的单次 Benchmark 更快，就直接修改生产参数。

这次 Benchmark 让我对 Memory-hard 有了更具体的认识：Argon2id 增加的是攻击者进行密码猜测的成本，但这份成本同样需要由正常的认证服务承担。安全性和资源消耗之间的权衡，必须通过实际测量来完成。

## 九、总结

这次 Health Master 的密码哈希迁移，最初只是想解决 MD5 不再适合密码存储的问题。

但在学习 Argon2id 的过程中，我逐渐理解了几个以前没有认真思考过的问题。

首先，密码哈希与加密是两种不同的概念。密码哈希不需要解密，认证时只需要重新计算并验证结果。

其次，Salt 和哈希参数不需要保密。Argon2id 的安全性并不依赖隐藏算法，而是依赖密码本身的不可预测性以及哈希计算成本。

再次，Memory-hard 让我意识到，密码哈希算法的设计目标和普通哈希算法并不相同。对于密码存储来说，有意增加计算和内存成本是一种安全设计。

最后，通过 `subtle.ConstantTimeCompare` 和 Timing Attack，我进一步认识到，安全问题不仅存在于密码学算法本身，也存在于具体的工程实现中。

回到 Health Master，这次改动也让我重新审视了旧密码迁移、数据库结构、错误处理和认证流程之间的关系。

对于一个已经存在用户数据的项目来说，安全升级不能只考虑新代码是否正确，还必须考虑历史数据如何处理，以及迁移会对用户产生什么影响。

这次 PR 是一次密码存储机制的升级，也是一次很有价值的学习过程。

## 参考资料

1. [Argon2id 密码哈希：memory-hard 的工程实践与前沿](https://bsheepcoder.github.io/2026/07/01/cs-argon2id-explained/)
2. [陈皓：计时攻击 Timing Attacks — 酷壳](https://gaojuqian.github.io/CoolShell/coolshell.cn/articles/21003.html)
3. [Health Master PR #333 — Migrate password hashing to Argon2id and improve login errors](https://github.com/damingerdai/health-master/pull/333)
4. [RFC 9106 — Argon2 Memory-Hard Function](https://www.rfc-editor.org/rfc/rfc9106.html)
5. [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
6. [Go crypto/subtle](https://pkg.go.dev/crypto/subtle)
7. [Go x/crypto/argon2](https://pkg.go.dev/golang.org/x/crypto/argon2)