---
title: 从 RAG 到有记忆的 Agent：Mastra Memory 与 PostgreSQL 实战
date: 2026-10-19 18:30:00
description: 在 Mastra、DeepSeek、SiliconFlow 与 PostgreSQL/pgvector RAG MVP 基础上加入 Memory，实测对话历史持久化、跨进程恢复、Working Memory 跨 Thread 共享及 Resource 隔离，并记录实际排错过程。
tags:
  - Mastra
  - Agent
  - Memory
  - PostgreSQL
  - TypeScript
categories:
  - AI
---

上一篇文章《[从零构建 Mastra RAG MVP：DeepSeek、SiliconFlow 与 PostgreSQL/pgvector 实战](https://blog.damingerdai.com/2026/10/18/ai/mastra/mastra-rag-mvp-deepseek-siliconflow-pgvector/)》中，我把自己的 Markdown 技术文章切分、向量化并存入 PostgreSQL/pgvector，再通过 Mastra Agent 和 DeepSeek 调用检索工具完成知识问答。

这解决了“Agent 如何知道我的知识库内容”，但没有解决另一个问题：**Agent 如何记住之前的对话？** 如果用户刚刚问过 Health Master 为什么从 MD5 迁移到 Argon2id，下一句再问“它有什么优势”，Agent 应该能够理解上下文。更进一步，用户在一个对话中说自己喜欢 Rust，换一个对话后，Agent 能不能继续记住？

这次在原有 `mastra-rag-app` 上增加 Mastra Memory，继续使用同一个 PostgreSQL 实例，并把实际测试和排错过程记录下来。以下代码以本次项目使用的 API 形态为例；Mastra 更新较快，复制时应核对安装版本的类型签名。

## 一、先区分 RAG、Conversation History 和 Working Memory

它们解决的不是同一类问题。

| 机制 | 保存或使用的内容 | 典型问题 |
| --- | --- | --- |
| RAG | 外部知识库的文档 Chunk 和向量 | “Health Master 为什么迁移到 Argon2id？” |
| Conversation History | 当前 Thread 的历史消息 | “刚才提到的算法有什么优势？” |
| Working Memory | Agent 维护的用户信息或任务状态 | “我最喜欢的编程语言是什么？” |
| Semantic Recall | 通过 Embedding 检索相关历史消息 | “几个月前我们讨论过什么索引方案？” |

Conversation History 保存对话过程，Working Memory 则更像一份持续更新的状态文档。Working Memory 不要求 Embedding；Semantic Recall 才涉及向量检索。两者不能混为一谈。

Mastra Memory 使用两个关键标识：

- `resource`：记忆所属的主体，例如 `user-001`。
- `thread`：一段具体对话，例如 `thread-001`。

同一个 Resource 可以拥有多个 Thread。默认对话历史按 Thread 读取；若 Working Memory 配置 `scope: "resource"`，则可在同一个 Resource 的不同 Thread 间共享工作记忆。

## 二、给原有 RAG Agent 增加 PostgreSQL Memory

原项目已经安装了 `@mastra/core` 和 `@mastra/pg`，现在补充：

```bash
npm install @mastra/memory
```

原有 `.env` 中已经有数据库连接地址：

```dotenv
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/mastra_rag
```

### 1. 在 Mastra 实例中配置 Storage

`src/mastra/index.ts`：

```typescript
import { Mastra } from "@mastra/core";
import { PostgresStore } from "@mastra/pg";
import { knowledgeAgent } from "./agents/knowledge.agent.js";

export const mastra = new Mastra({
  storage: new PostgresStore({
    id: "mastra-storage",
    connectionString: process.env.DATABASE_URL!,
  }),
  agents: { knowledgeAgent },
});
```

这里的 `PostgresStore` 用于保存 Mastra 的 Thread、消息及其他框架数据。它与上一篇使用的 `PgVector` 不是一回事：前者是结构化持久化 Storage，后者负责向量索引和相似度查询。

### 2. 在 Knowledge Agent 中启用 Memory

保留上一篇的 DeepSeek 模型和 `knowledgeSearchTool`，增加 `memory`：

```typescript
import { Agent } from "@mastra/core/agent";
import { Memory } from "@mastra/memory";
import { createOpenAICompatible } from "@ai-sdk/openai-compatible";
import { knowledgeSearchTool } from "../tools/knowledge-search.tool.js";

const deepseek = createOpenAICompatible({
  name: "deepseek",
  baseURL: "https://api.deepseek.com/v1",
  apiKey: process.env.DEEPSEEK_API_KEY!,
});

export const knowledgeAgent = new Agent({
  id: "knowledge-agent",
  name: "Knowledge Agent",
  model: deepseek.chatModel("deepseek-chat"),
  instructions: `
你是中文技术知识助手。
技术知识问题可以调用 knowledgeSearchTool 检索知识库。
用户个人偏好和对话上下文优先使用可用的 Memory。
不要把用户个人偏好伪装成知识库文档来源。
如果没有可靠信息，请明确说明。
  `,
  tools: { knowledgeSearchTool },
  memory: new Memory({
    options: {
      lastMessages: 20,
      workingMemory: {
        enabled: true,
        scope: "resource",
      },
    },
  }),
});
```

`lastMessages: 20` 控制每次加载到上下文中的近期消息数量，**不是数据库最多只能保存 20 条消息**。Working Memory 开启后，Agent 可以在支持的更新机制下维护相关信息，但不保证每一句话都会自动提取或保存。

上面的示例将 Storage 配在 Mastra 实例中。若直接实例化并单独调用 Agent，而不是通过 `mastra.getAgent()` 使用它，需要确认 Memory 是否正确取得 Storage；也可以按当前版本支持的 API 显式传入 Storage。

## 三、第一次排错：PgVector 初始化失败

刚开始执行测试：

```bash
npx tsx src/scripts/test-memory.ts
```

遇到了：

```text
MastraError: PgVector: invalid config.
Provide either {pool}, {connectionString},
{host,port,database,user,password}, or a pg ClientConfig.

id: MASTRA_VECTOR_PG_INITIALIZATION_FAILED
```

堆栈指向的不是 Memory，而是原有的 `src/mastra/lib/vector-store.ts`。原因是执行脚本时，导入链会初始化 RAG 向量存储；如果 `.env` 尚未加载，`process.env.DATABASE_URL` 就可能是 `undefined`。

尤其需要注意：

```typescript
connectionString: process.env.DATABASE_URL!
```

这里的 `!` 只是 TypeScript 非空断言，并不会在运行时检查变量是否存在。

解决方式是在脚本入口最先加载环境变量：

```typescript
import "dotenv/config";

import { mastra } from "../mastra/index.js";
```

或者在 Node.js 24 下直接运行：

```bash
node --env-file=.env --import tsx src/scripts/test-memory.ts
```

可以单独验证变量是否存在，避免打印数据库密码：

```bash
node --env-file=.env -e \
  "console.log(Boolean(process.env.DATABASE_URL))"
```

这次错误也提醒我：**测试 Memory 时，原有 RAG 模块仍可能因为 ESM 静态导入而被初始化。** 报错来自哪个模块，应该先看堆栈，而不是直接归咎于新增加的功能。

## 四、验证 Conversation History

先不考虑跨 Thread，只测试同一个 Thread 是否记住前一轮对话。

`src/scripts/test-memory2.ts`：

```typescript
import "dotenv/config";
import { mastra } from "../mastra/index.js";

const agent = mastra.getAgent("knowledgeAgent");
const memory = {
  resource: "user-001",
  thread: "thread-001",
};

const first = await agent.generate(
  "请记住一个测试信息：我的测试代号是 Blue-Penguin-729。只回复 OK。",
  { memory },
);
console.log("First Response:", first.text);

const second = await agent.generate(
  "我的测试代号是什么？不要调用知识库，只根据对话历史回答。",
  { memory },
);
console.log("Second Response:", second.text);
```

实际运行：

```text
First Response: 好的，我记住了：Blue-Penguin-729。
Second Response: 你的测试代号是 Blue-Penguin-729。
```

这证明在本次测试中，Agent 能够使用同一个 Thread 的先前对话。它还不能单独证明 PostgreSQL 持久化：同一进程的两次调用，仍需结合数据库记录与跨进程测试判断。

## 五、PostgreSQL 自动创建了什么？

启动后，我查询了数据库中以 `mastra` 开头的表：

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_name LIKE 'mastra%'
ORDER BY table_name;
```

这次项目中自动创建了 **39 张 `mastra_` 表**。其中能看到 `mastra_agents`、`mastra_agent_versions`、`mastra_ai_spans`、`mastra_datasets` 等。

39 张表不代表 Memory 需要 39 张表。Mastra Storage 还承担 Agent、Tracing、Dataset 等多类功能；表数量和名称也可能随版本改变。通常不需要自己编写 Memory 的 `CREATE TABLE`，但 PostgreSQL 数据库本身需要事先存在，连接用户也需要相应 DDL 权限。生产环境还应考虑迁移和权限管理。

在 `mastra_messages` 中，我看到了：

- `thread_id = thread-001`
- `resourceId = user-001`
- `role = user / assistant`
- `type = v2`
- `content` 为包含消息 Parts 的 JSON

除了文本消息，记录中还出现了 `tool-invocation`，其中包含工具执行结果。这与原来的 RAG Tool 调用链吻合，说明 Mastra 保存的不只是纯文本回答。

可以先用下面的 SQL 查表结构：

```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name IN ('mastra_messages', 'mastra_threads')
ORDER BY table_name, ordinal_position;
```

由于列名可能使用大小写混合（例如 `resourceId`），实际查询时要根据数据库中的字段名称正确加双引号。

## 六、第二次测试：跨 Node.js 进程恢复对话

如果只是同一进程内能回答测试代号，仍不足以说明数据从 PostgreSQL 被恢复。因此我关闭前一个脚本，再单独执行一个新脚本。

`src/scripts/test-memory3.ts`：

```typescript
import "dotenv/config";
import { mastra } from "../mastra/index.js";

const agent = mastra.getAgent("knowledgeAgent");

const response = await agent.generate(
  "我之前告诉过你的测试代号是什么？",
  {
    memory: {
      resource: "user-001",
      thread: "thread-001",
    },
  },
);

console.log("First Response:", response.text);
```

实际输出：

```text
First Response: 你的测试代号是 Blue-Penguin-729。
```

结合此前在 PostgreSQL 中查到的消息记录，可以确认：**这次项目的 Conversation History 已经具备跨进程持久化恢复能力**。它不依赖上一轮 Node.js 进程继续运行。

## 七、Working Memory：跨 Thread 保存用户偏好

Conversation History 解决了同一 Thread 的上下文连续性。但用户可能有多个对话窗口：一个聊 Go，一个聊 Kubernetes，还有一个聊 RAG。不能要求每个新 Thread 都附带之前所有对话。

Working Memory 的目标不是原样复制所有历史消息，而是维护后续仍然有用的信息，例如：

```markdown
# User Profile
- Favorite Programming Language: Rust
- Current Goal: Learning Kubernetes Operator
```

本次配置使用：

```typescript
workingMemory: {
  enabled: true,
  scope: "resource",
}
```

`scope: "resource"` 是关键：相同 Resource 的不同 Thread 可以共享 Working Memory。也可以使用模板约束结构，例如：

```typescript
workingMemory: {
  enabled: true,
  scope: "resource",
  template: `
# User Profile
- Favorite Programming Language:
- Current Learning Goal:
  `,
}
```

模板由开发者定义，具体值可以由 Agent 通过记忆机制更新，不需要业务代码为每条偏好手写 SQL。不过自动提取与更新的结果依赖模型和配置，不能当作严格一致的业务数据库。

### 用两个 Thread 验证

`src/scripts/test-memory4.ts`：

```typescript
import "dotenv/config";
import { mastra } from "../mastra/index.js";

const agent = mastra.getAgent("knowledgeAgent");
const resource = "user-001";

// Thread A：提供偏好
await agent.generate(
  "请记住，我最喜欢的编程语言是 Rust。",
  {
    memory: {
      resource,
      thread: "working-memory-a",
    },
  },
);

// Thread B：同一个 Resource，但换一个 Thread
const response = await agent.generate(
  "我最喜欢的编程语言是什么？",
  {
    memory: {
      resource,
      thread: "working-memory-b",
    },
  },
);

console.log(response.text);

// Thread C：换一个 Resource，测试隔离
const response2 = await agent.generate(
  "我最喜欢的编程语言是什么？",
  {
    memory: {
      resource: "user-002",
      thread: "working-memory-c",
    },
  },
);

console.log(response2.text);
```

第一次回答的关键内容是：

```text
你最喜欢的编程语言是 Rust。
```

第二次则表示：

```text
知识库中没有关于你个人偏好的信息，
因此无法判断你最喜欢的编程语言是什么。
```

同时，`mastra_threads` 中可以看到：

| id | resourceId |
| --- | --- |
| `thread-001` | `user-001` |
| `working-memory-a` | `user-001` |
| `working-memory-b` | `user-001` |

随后运行第三次请求还会创建 `working-memory-c`，对应 `user-002`。

这些结果说明：**在本次实验中，同一 Resource 的不同 Thread 可以取得先前提供的偏好，而另一个 Resource 没有取得该信息。** 这与 Resource 级 Working Memory 的预期一致。

但需要严谨一点：**回答正确不等于已经直接证明 Working Memory 表里保存了 Rust**。如果启用了 Semantic Recall、其他共享上下文或自定义工具，也可能提供跨 Thread 信息。Agent 自己声称“来自对话记录”也不能当作底层数据来源的证据。要验证具体来源，应关闭其他跨 Thread 机制，并直接读取 Working Memory 内容或检查对应持久化记录。

## 八、如何进一步检查 Working Memory 的存储位置

不要先假设它一定保存在 `mastra_threads.metadata` 或某个固定名称的表中。不同 Mastra 版本的实现可能不同。

先查相关表和字段：

```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'public'
  AND (
    table_name ILIKE '%memory%'
    OR table_name ILIKE '%resource%'
    OR table_name = 'mastra_threads'
  )
ORDER BY table_name, ordinal_position;
```

也可以用 Agent 的 Memory API 检查当前版本是否支持读取 Working Memory。方法签名需要根据实际安装版本确认，不应该直接猜测返回结构。

为了避免旧数据影响实验，最好为每次测试使用新的 Resource 和 Thread ID，并保证没有其他跨 Thread 检索通道。

## 九、为什么问个人偏好时，Agent 还去查 RAG？

本次输出里还有一个现象：用户问的是“我最喜欢的编程语言是什么”，但 Agent 仍然检索了知识库，随后说明文档里只有 Go/Argon2id 的技术文章，没有用户偏好。

这并不是 PostgreSQL Memory 失效，而是 **Agent 的工具选择策略** 仍然偏向 RAG。上一篇的 Agent 指令要求技术问题先检索，增加 Memory 后，需要重新划分信息来源：

```typescript
instructions: `
你是中文技术知识助手。
个人偏好、当前任务状态：优先使用可用的 Memory。
当前对话的追问：优先使用 Conversation History。
需要外部技术资料时：调用 knowledgeSearchTool。
不要将个人记忆编造成知识库引用。
信息不足时明确说明，不要猜测。
`,
```

这只是提示词层面的引导，不能严格保证工具一定不会调用。若需要确定性行为，可以在应用层对问题分类，再决定是否开放 RAG Tool，或使用明确的工作流控制调用顺序。

## 十、进一步扩展：Semantic Recall 与 Observational Memory

目前这篇文章主要验证 Conversation History 和 Working Memory。Mastra 还提供其他记忆能力：

**Semantic Recall**：使用 Embedding 检索相关历史消息。它与知识库 RAG 都会涉及向量，但检索对象不同。前者检索对话，后者检索文章。项目已有 SiliconFlow `BAAI/bge-m3`（1024 维）和 `PgVector`，后续可以复用相同服务商的 Embedding 能力，但应隔离记忆向量与知识库向量的索引，并确保维度匹配。

**Observational Memory**：面向长对话的观察、整理与压缩机制，适合研究如何减少长历史对上下文窗口和 Token 成本的压力。它与 Working Memory 的用途不同，是否同时启用应根据当前 Mastra 版本、模型支持和实际效果评估。

这两项本次没有做完整实测，因此不把它们写成已经验证通过的功能。

## 十一、生产环境还需要考虑什么？

实验中使用 `user-001`、`user-002` 作为 Resource 很方便，但真实应用不能直接信任客户端传来的 Resource ID。Resource 必须由服务端身份认证结果确定，否则用户可能通过伪造 ID 访问其他人的记忆。

此外，Working Memory 由模型参与维护，不适合用作权限、支付金额、订单状态等关键业务数据的唯一事实来源。还需要设计记忆删除、数据保留期限、隐私处理和用户纠错机制。数据库自动建表方便开发，但生产环境最好管理好迁移和 DDL 权限。

## 十二、总结

从上一篇 RAG MVP 到这次 Memory 实验，Agent 增加了两个重要能力：

1. **Conversation History**：同一 Thread 的历史消息保存在 PostgreSQL 中，新 Node.js 进程仍然可以恢复。
2. **Working Memory**：在 Resource 作用域下，不同 Thread 可以共享用户相关状态；本次 Rust 偏好测试观察到了预期的跨 Thread 行为和不同 Resource 的隔离效果。

这次最有价值的不是让 Agent 回答出一个测试代号，而是把“模型回答正确”拆解成了可验证的工程问题：消息是否入库、Thread 如何关联 Resource、进程重启后是否能恢复、跨 Thread 的信息究竟来自哪一种记忆机制。

RAG 解决“Agent 如何获取外部知识”，Memory 则解决“Agent 如何利用之前的交互”。两者结合，才让一个最小知识问答 Demo 开始具备持续使用的可能。

**相关资料**：

- [上一篇：从零构建 Mastra RAG MVP](https://blog.damingerdai.com/2026/10/18/ai/mastra/mastra-rag-mvp-deepseek-siliconflow-pgvector/)
- [Mastra Memory 官方文档](https://mastra.ai/docs/memory/overview)
- [Mastra 官方网站](https://mastra.ai/)
