# Solana简单开始

## 什么是Solana

[官网](https://docs.solana.com/introduction)提到：

*Solana is an open source project implementing a new, high-performance, permissionless blockchain.*

 翻译过来就是Solana是一个全新的、高性能的、无需认证的开源的区块链。

 ## 本地开发

 本文使用Macos系统

 ### 安装Rust环境

 打开终端

```bash
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```

安装c语言编译器

```bash
xcode-select --install
```

检查安装是否成功

```bash
rustc -V

rustc 1.65.0 (897e37553 2022-11-02)
```

### 安装Solana开发环境

安装Solana的命令行工具

```bash
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"
```

检查solana是否安装成功

```bash
solana --version

solana-cli 1.13.5 (src:959b760c; feat:1365939126)
```

连接到开发网络

```
solana config set --url https://api.devnet.solana.com

Config File: /Users/daming/.config/solana/cli/config.yml
RPC URL: https://api.devnet.solana.com 
WebSocket URL: wss://api.devnet.solana.com/ (computed)
Keypair Path: /Users/daming/.config/solana/id.json 
Commitment: confirmed 
```

创建本地账号

```bash
solana-keygen new
# (如果已经有账号可以使用`solana-keygen new --force`强制重新创建)
```

查看账号的公钥

```bash
solana-keygen pubkey 

HLZJ6dhFs29s2NRP9AbdVAJ1TARudCeafoyCVxvQU943
```

在开发网上申请SQL空投

```bash
solana airdrop 2

# solana似乎存在限制，在devnet上最多只能2个sql
Requesting airdrop of 2 SOL

Signature: gTZ9afTi9isLsnXp8st7T8PH2mAafyGumkbJQLWugJYv5ktjBr6NSMEoqUYmECgrdrgEgqVeJYHNUnFG77DQmyL

2 SOL
```

查看余额

```bash
solana-keygen new -o solana_memo_program.json

# key: 8vk9zEsPY8oNDYudFB4eriY17HWAf8Uc4ncNg4TPwdrZ
```

保存好key， 这个后面再发布程序的时候使用。前面那个账号是发布程序的账号，这个账号就是对应的程序的地址。


### 智能合约

直接使用solana的例子

```bash
git clone https://github.com/solana-labs/solana-program-library.git
```

进入`memo/program`，然后执行

```bash
cargo build-bpf

# 如果出现
# namespaced features with the `dep:` prefix are only allowed on the nightly channel and requires the `-Z namespaced-features` flag on the command-line
# 请使用
# cargo build-bpf -- -Z namespaced-features
```

 ## 参考资料

 1. [solana](https://docs.solana.com/)
 2. [Solana开发教程：初识合约开发](https://solongwallet.medium.com/solana%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6%E5%BC%80%E5%8F%91-%E5%BC%80%E5%8F%91%E5%B7%A5%E5%85%B7%E9%93%BE-121ed91acca1)
 3. [Rust语言圣经(Rust Course)](https://course.rs/about-book.html)
 4. [如何开发并部署Solana智能合约](https://blog.chain.link/how-to-build-and-deploy-a-solana-smart-contract-zh/)
 5. [solana airdrop 10でBalance unchaged](https://blog.tanebox.com/archives/1615/)
 6. [【区块链】Solana 开发笔记 Part 1](https://rustmagazine.github.io/rust_magazine_2021/chapter_10/solana-learn-part1.html)