---
title: 当 React Router 遇见多运行时：一次关于 React 架构的思考
date: 2026-08-06 13:28:47
tags: [React, React Router, React 架构, Monorepo, Runtime]
categories: [前端]
---

# 起因

最近我准备重新开始一个新的 SaaS 平台，技术栈依然选择了 React + React Router + Vite。

一开始，我以为和以前一样，直接：

```bash
npm create vite@latest my-react-app -- --template react
```

整个架构应该还是：

```text
React
    ↓
React Router
    ↓
Vite
```

但是随着需求越来越明确，我在开发过程中发现了几个问题，也让我开始重新思考 React 应用到底应该怎么组织。

# 问题

## 第一个问题：同一个应用，不同页面应该采用不同渲染策略

以前我一直觉得，一个 React 应用选择 SSR、CSR 或 SSG 其中一种就可以了。

后来发现并不是这样。

对于一个 SaaS 平台来说，不同页面关注点完全不同。

例如：

```text
首页
    SSG / Prerender

商品详情
    SSR

Dashboard
    CSR

后台管理
    SPA Mode
```

首页主要承担品牌展示和 SEO，更适合静态生成。

商品详情既需要搜索引擎能够抓取，又希望数据能够尽可能实时，因此 SSR 会更加合适。

Dashboard 更关注交互体验，大部分数据本身也是通过接口获取，因此 CSR 已经足够。

后台管理基本没有 SEO 的需求，更适合 SPA Mode。

因此我越来越觉得：

> 一个 React 应用，不应该只有一种渲染策略。

渲染策略本身没有绝对优劣，它本质上是在 SEO、首屏速度、服务器成本、实时性以及交互体验之间做权衡。

不同业务模块，本来就应该拥有不同的最优解。

由于我们使用的是 React Router，因此我开始调研 React Router 的 Framework Mode。

Framework Mode 在 Data Mode 的基础上，通过 Vite Plugin 提供了 SSR、SPA Mode、Prerender 等能力，让不同页面能够采用不同的渲染策略。

---

## 第二个问题：同一个应用可能需要部署到不同 Runtime

我们目前计划部署在 Vercel。

但是我并不希望未来只能部署在 Vercel。

以后完全有可能迁移到：

- Cloudflare Workers
- 自建 Node.js
- Kubernetes

于是第二个问题出现了。

> 我的业务代码，为什么要绑定到某一个 Runtime？

Browser、Node.js、Edge Runtime 都只是代码运行的位置。

理论上，真正的业务逻辑并不应该关心自己运行在哪里。

真正发生变化的，只应该是 Runtime 提供的能力。

例如：

```text
Browser Runtime

Node.js Runtime

Edge Runtime
```

不同 Runtime 有不同 API。

但是业务逻辑应该保持一致。

---

## 第三个问题：现有方案是不是已经解决了？

### Next.js

Next.js 是一个非常优秀的全栈框架。

采用 Next.js 并没有问题，它同样支持部署到不同的平台。

但是它很多能力都围绕自身的运行时和生态进行设计。

而我的目标更偏向于 Runtime Agnostic（运行时无关）。

也就是说，我更希望业务代码能够尽可能脱离具体平台。

因此目前我并没有选择 Next.js。

---

### 微前端

在寻找方案的时候，第一个想到的是微前端。

微前端主要解决的是：

- 多团队协作
- 独立开发
- 独立部署
- 技术栈自由
- 增量升级

但是这些都不是我当前真正的问题。

我真正想解决的是：

```text
Runtime

↓

Rendering

↓

Deployment
```

也就是说：

- 支持不同 Runtime
- 支持不同渲染策略
- 支持不同部署平台

这和微前端解决的问题并不相同。

---

### Module Federation

第二个想到的是 Module Federation。

Module Federation 更偏向于：

> 多个应用之间如何在运行时共享模块。

例如共享：

- React 组件
- 页面
- 工具库
- UI

而我的需求并不是运行时共享。

我更希望共享的是：

- Domain
- Use Case
- UI
- API
- 业务逻辑

然后根据不同 Runtime 构建不同的产物。

也就是说，我需要的是构建阶段共享，而不是运行时共享。

因此 Module Federation 对我来说有点"用力过猛"。

---

## 最后，我又重新回到了 React Router

绕了一圈以后，我发现：

React Router 本来就在解决 Route 的问题。

Framework Mode 又提供了 SSR、SPA Mode、Prerender 等能力。

如果再结合 Monorepo，其实已经能够满足我目前的大部分需求。

# 思路

我想实现的大概是这样子的设计：

![架构图](/images/react-router-meets-multi-runtime/1.png)

整个思路其实可以总结成下面这样：

```text
业务需求

↓

不同页面需要不同渲染策略

↓

不同 Runtime 需要不同入口

↓

希望共享业务代码

↓

React Router Framework Mode

↓

Monorepo

↓

Multi Runtime
```

为了实现**业务共享 → UI 共享 → Use Case 共享 → Runtime 不同**，我目前更倾向于下面这样的目录结构：

```text
apps/
    web-node/
    web-edge/

packages/
    domain/
    application/
    ui/
    api/
    utils/
```

其中：

- apps 用来解决 Runtime。
- packages 用来解决共享。

不同 Runtime 只是不同的入口。

真正的业务代码全部放到 packages。

---

## 真正需要隔离的是 Runtime

后来我发现，真正需要隔离的其实不是 React。

而是 Runtime。

例如：

- Session
- Cache
- Storage
- Queue
- Email
- Logger

这些能力，在 Browser、Node.js 和 Edge Runtime 中都可能存在不同的实现。

因此 packages 不应该直接依赖具体 Runtime。

而应该依赖抽象。

例如：

```text
Cache

Storage

Session
```

然后不同 Runtime 再分别提供自己的实现。

例如：

```text
Node.js Runtime

↓

Redis

Edge Runtime

↓

KV
```

这样真正变化的只有 Runtime。

业务逻辑本身完全不需要修改。

---

## Monorepo 并不能解决 Runtime

很多人会把 Monorepo 当成万能方案。

但我觉得并不是。

Monorepo 解决的是：

- 代码组织
- 依赖管理
- 统一构建
- 统一版本

它并不会自动解决 Runtime 差异。

真正解决 Runtime 的，是不同 Runtime 的入口以及对应的 Adapter。

Monorepo 只是让这一切更容易维护。

---

## 一个代码仓库，不代表只有一个构建产物

很多人会觉得：

```text
一套代码

↓

一个构建产物

↓

部署 everywhere
```

实际上我认为更合理的是：

```text
共享业务代码

↓

不同 Runtime

↓

分别构建

↓

不同部署产物
```

例如：

```bash
pnpm build:node

pnpm build:edge
```

最终得到：

```text
dist/node

dist/edge
```

共享的是源码。

不是构建产物。

# 总结

这是我最近对于 React 架构的一次思考。

目前来看，我更希望最终形成这样的架构：

- 不同页面，可以自由选择不同渲染策略；
- 不同 Runtime，只需要更换入口和 Adapter；
- 业务代码尽可能共享；
- 部署平台尽可能自由。

当然，目前还有不少问题没有真正验证：

1. 还没有在线上大规模实践；
2. Cloudflare 和 Node.js Runtime 目前仍然存在能力差异；
3. React Router 后续的发展方向还需要继续观察；
4. TanStack Start 或许也是一个值得继续研究的方向。

至少目前来看，我觉得 React Router Framework Mode + Monorepo 已经能够满足我目前对于 React 架构的大部分设想。

# 引用

1. https://reactrouter.com/start/framework/installation
2. https://reactrouter.com/start/framework/rendering
3. https://wujie-micro.github.io/doc/guide/
4. https://module-federation.io/zh/guide/start/index.html