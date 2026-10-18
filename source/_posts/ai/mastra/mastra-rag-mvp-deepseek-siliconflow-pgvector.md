---
title: 从零构建 Mastra RAG MVP：DeepSeek、SiliconFlow 与 PostgreSQL/pgvector 实战
date: 2026-10-18 10:30:00
description: 使用 Mastra、DeepSeek、SiliconFlow BGE-M3 和 PostgreSQL/pgvector 构建可运行的 RAG 知识库，记录文档切分、向量入库、检索、Agent 工具调用及实际排错过程。
tags:
  - TypeScript
  - Mastra
  - RAG
  - DeepSeek
  - PostgreSQL
  - AI Agent
categories:
  - AI
---

最近我在尝试使用 [Mastra](https://mastra.ai/) 构建一个简单的 RAG（Retrieval-Augmented Generation，检索增强生成）应用。目标并不是一开始就做完整的知识库平台，而是先打通一条最小可用链路：**把自己的 Markdown 技术文章导入向量数据库，然后用自然语言提问，让模型根据文章内容回答问题。**

这次选用的技术组合是：

- **Mastra**：组织 Agent 和 Tool。
- **DeepSeek**：负责理解问题、决定调用检索工具并生成答案。
- **SiliconFlow**：提供 `BAAI/bge-m3` Embedding 模型，将文本转换为向量。
- **PostgreSQL + pgvector**：保存并检索文档向量。
- **TypeScript**：实现导入脚本、检索服务和命令行问答。

这篇文章记录 MVP 的实现思路、关键代码，以及真正遇到的几次错误。代码按这次项目的实现方式整理；Mastra 和 AI SDK 的具体类型、方法签名可能随版本调整，复制时应以项目安装版本为准。

---

## 一、先理解要实现的 RAG 链路

普通 LLM 并不知道我最近在 GitHub 上修改了什么，也不一定知道 Health Master 项目为什么从 MD5 迁移到 Argon2id。RAG 的思路是先检索外部知识，再将检索到的上下文提供给模型。

```text
Markdown 技术文章
       │
       ▼
   MDocument
       │
       ▼
  Chunking（文本切分）
       │
       ▼
SiliconFlow / BAAI/bge-m3
       │
       ▼
Embedding（1024 维）
       │
       ▼
PostgreSQL / pgvector
       │
       ├──────────────────────────────┐
       │                              │
       ▼                              │
 用户自然语言提问                      │
       │                              │
       ▼                              │
  Mastra Agent（DeepSeek）            │
       │                              │
       ▼                              │
 knowledgeSearchTool                  │
       │                              │
       ▼                              │
 问题 Embedding → 相似度检索 ───────────┘
       │
       ▼
 返回相关 Chunk + source
       │
       ▼
 DeepSeek 组织答案
```

这里有两个阶段：**Ingestion（知识入库）** 和 **Retrieval + Generation（检索与生成）**。Embedding 模型和生成模型可以来自不同服务商；两者不需要是同一个模型。

## 二、项目初始化

在一个 Node.js / TypeScript 项目中安装依赖：

```bash
npm install @mastra/core @mastra/rag @mastra/pg @ai-sdk/openai-compatible ai dotenv
npm install -D typescript tsx
```

项目按职责拆分：

```text
mastra-rag-app/
├── src/
│   ├── mastra/
│   │   ├── agents/
│   │   │   └── knowledge.agent.ts
│   │   ├── lib/
│   │   │   ├── embedding.ts
│   │   │   └── vector-store.ts
│   │   ├── services/
│   │   │   ├── knowledge.service.ts
│   │   │   └── retrieval.service.ts
│   │   └── tools/
│   │       └── knowledge-search.tool.ts
│   └── scripts/
│       ├── init-vector-db.ts
│       ├── ingest.ts
│       ├── search.ts
│       └── chat.ts
├── .env
└── package.json
```

环境变量示例：

```dotenv
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/mastra_rag
SILICONFLOW_API_KEY=your-siliconflow-api-key
SILICONFLOW_BASE_URL=https://api.siliconflow.cn/v1
DEEPSEEK_API_KEY=your-deepseek-api-key
```

不要把真实密钥提交到 Git 仓库。建议将 `.env` 加入 `.gitignore`，另提供不含密钥的 `.env.example`。

### PostgreSQL 的准备工作

数据库需要安装 `pgvector` 扩展。确保使用的是支持 pgvector 的 PostgreSQL 实例，然后在目标数据库执行：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

注意：安装数据库扩展与创建 Mastra 使用的向量表是两件不同的事。

## 三、配置 SiliconFlow Embedding

创建 `src/mastra/lib/embedding.ts`：

```ts
import { createOpenAICompatible } from "@ai-sdk/openai-compatible";

const apiKey = process.env.SILICONFLOW_API_KEY;
if (!apiKey) {
  throw new Error("SILICONFLOW_API_KEY is missing");
}

export const siliconflow = createOpenAICompatible({
  name: "siliconflow",
  baseURL: process.env.SILICONFLOW_BASE_URL ?? "https://api.siliconflow.cn/v1",
  apiKey,
});

export const embeddingModel = siliconflow.embeddingModel("BAAI/bge-m3");
```

`BAAI/bge-m3` 在这次调用中返回 **1024 维向量**。这决定了后续创建 pgvector 索引时的 `dimension` 参数。

这里踩过两个坑。

### 坑一：Embedding 请求返回 404

一开始配置的 `baseURL` 没有包含 `/v1`，请求最终命中了错误的路径。改为：

```text
https://api.siliconflow.cn/v1
```

之后才进入正确的 Embedding 接口。对于 OpenAI-compatible Provider，**Base URL 是否包含版本前缀**必须核对服务商文档。

### 坑二：Embedding 请求返回 401

当时一度怀疑 `.env` 没有加载、API Key 无效，或者 SDK 的请求格式有问题。于是分别用原生 `fetch` 和 SDK 做最小化测试：

```ts
import "dotenv/config";
import { embed } from "ai";
import { embeddingModel } from "../mastra/lib/embedding";

const { embedding } = await embed({
  model: embeddingModel,
  value: "测试 Embedding 是否正常",
});

console.log("Dimensions:", embedding.length);
```

原生 HTTP 请求返回 `200`，SDK 测试也能输出 `Dimensions: 1024`，说明服务本身和模型调用没有问题。最终发现实际项目中的配置把 **`SILICONFLOW` 拼成了 `SILOCONFLOW`**，导致读取了错误的环境变量名。

这个排错过程的经验是：**先用最小请求证明第三方 API 可用，再逐层检查业务代码**，不要一开始就把所有问题归因于框架。

## 四、配置 PostgreSQL 向量存储

创建 `src/mastra/lib/vector-store.ts`：

```ts
import { PgVector } from "@mastra/pg";

export const INDEX_NAME = "knowledge_base";

export const vectorStore = new PgVector({
  id: "knowledge-vectors",
  connectionString: process.env.DATABASE_URL!,
});
```

这里的 `INDEX_NAME` 是后续写入和查询时使用的名称，必须保持一致。

### 坑三：Vector table 不存在

Embedding 调用成功后，导入时遇到：

```text
MastraError: Vector table "knowledge_base" does not exist
id: MASTRA_VECTOR_PG_DESCRIBE_INDEX_FAILED
```

原因很直接：**配置 PgVector 实例并不会自动替我们创建 `knowledge_base` 表**。需要先创建索引对应的存储结构。

创建 `src/scripts/init-vector-db.ts`：

```ts
import "dotenv/config";
import { vectorStore, INDEX_NAME } from "../mastra/lib/vector-store";

async function main() {
  await vectorStore.createIndex({
    indexName: INDEX_NAME,
    dimension: 1024,
  });

  console.log(`Vector index "${INDEX_NAME}" initialized`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

首次运行前先执行初始化脚本。若索引已经存在，是否允许重复执行取决于所安装的 Mastra 版本；不应假设 `createIndex()` 一定是幂等的。

## 五、将 Markdown 切分并导入向量库

创建 `src/mastra/services/knowledge.service.ts`：

```ts
import { MDocument } from "@mastra/rag";
import { embedMany } from "ai";
import { embeddingModel } from "../lib/embedding";
import { vectorStore, INDEX_NAME } from "../lib/vector-store";

export async function ingest(text: string, source: string) {
  const document = MDocument.fromText(text);

  const chunks = await document.chunk({
    strategy: "recursive",
    maxSize: 512,
    overlap: 50,
  });

  if (chunks.length === 0) {
    return { count: 0 };
  }

  const { embeddings } = await embedMany({
    model: embeddingModel,
    values: chunks.map((chunk) => chunk.text),
  });

  await vectorStore.upsert({
    indexName: INDEX_NAME,
    vectors: embeddings,
    metadata: chunks.map((chunk, index) => ({
      text: chunk.text,
      source,
      chunkIndex: index,
    })),
  });

  return { count: chunks.length };
}
```

`source` 保存文档来源，例如 GitHub Raw Markdown URL。后面检索到 Chunk 时，可以用它展示引用来源。

### 编写导入脚本

下面是一个适合本地 Markdown 文件的 CLI 示例，创建 `src/scripts/ingest.ts`：

```ts
import "dotenv/config";
import { readFile } from "node:fs/promises";
import { resolve } from "node:path";
import { ingest } from "../mastra/services/knowledge.service";

const file = process.argv[2];
if (!file) {
  throw new Error("Usage: npm run ingest -- <markdown-file> [source-url]");
}

const text = await readFile(resolve(file), "utf8");
const source = process.argv[3] ?? file;
const result = await ingest(text, source);

console.log(`Ingested ${result.count} chunks from ${source}`);
```

运行示例：

```bash
npm run ingest -- ./articles/golang-argon2id-password-hashing.md \
  https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/refs/heads/master/source/_posts/back-end/golang-argon2id-password-hashing.md
```

这篇 Argon2id 文章有 YAML Front Matter、标题、代码块和表格。当前 MVP 直接使用递归切分策略，能够工作，但不是专门针对 Markdown 语义结构优化的切分方式。后续可以考虑按标题层级分段，并避免将代码块截断。

> **当前限制**：上面的 `upsert` 没有提供稳定的 Chunk ID。重复导入同一篇文章时，可能产生重复向量。幂等导入、文档更新和删除尚未实现。

## 六、先不接 Agent，验证向量检索

将检索独立成服务有一个好处：出现问题时，可以明确区分是 **Embedding/向量检索不准确**，还是 **Agent/LLM 没有正确使用检索结果**。

创建 `src/mastra/services/retrieval.service.ts`：

```ts
import { embed } from "ai";
import { embeddingModel } from "../lib/embedding";
import { vectorStore, INDEX_NAME } from "../lib/vector-store";

export async function retrieve(query: string, topK = 5) {
  const { embedding } = await embed({
    model: embeddingModel,
    value: query,
  });

  const results = await vectorStore.query({
    indexName: INDEX_NAME,
    queryVector: embedding,
    topK,
  });

  return results.map((result) => ({
    id: result.id,
    score: result.score,
    text: String(result.metadata?.text ?? ""),
    source: String(result.metadata?.source ?? ""),
  }));
}
```

创建 `src/scripts/search.ts`：

```ts
import "dotenv/config";
import { retrieve } from "../mastra/services/retrieval.service";

const query = process.argv.slice(2).join(" ") || "为什么使用 Argon2id？";
const results = await retrieve(query, 3);

for (const [index, result] of results.entries()) {
  console.log(`\n--- Result ${index + 1} ---`);
  console.log("Score:", result.score);
  console.log("Source:", result.source);
  console.log("Text:", result.text);
}
```

测试：

```bash
npm run search -- "Health Master 为什么使用 Argon2id？"
```

这里的 `score` 是检索相似度得分，**不是模型答案的正确率，也不能直接当作可信度百分比**。

## 七、将检索包装成 Mastra Tool

创建 `src/mastra/tools/knowledge-search.tool.ts`：

```ts
import { createTool } from "@mastra/core/tools";
import { z } from "zod";
import { retrieve } from "../services/retrieval.service";

export const knowledgeSearchTool = createTool({
  id: "knowledge-search",
  description:
    "Search the knowledge base for relevant information about stored documents.",
  inputSchema: z.object({
    query: z.string().describe("Search query"),
  }),
  outputSchema: z.object({
    results: z.array(
      z.object({
        source: z.string(),
        text: z.string(),
        score: z.number(),
      }),
    ),
  }),
  execute: async ({ query }) => {
    const results = await retrieve(query, 5);
    return {
      results: results.map(({ source, text, score }) => ({
        source,
        text,
        score,
      })),
    };
  },
});
```

这里没有直接使用封装好的向量查询工具，而是把自己的 `retrieve()` 包装成 Mastra Tool。这样未来要添加权限过滤、去重或 Reranking，可以只修改检索服务。

## 八、使用 DeepSeek 创建 Knowledge Agent

创建 `src/mastra/agents/knowledge.agent.ts`：

```ts
import { Agent } from "@mastra/core/agent";
import { createOpenAICompatible } from "@ai-sdk/openai-compatible";
import { knowledgeSearchTool } from "../tools/knowledge-search.tool";

const deepseek = createOpenAICompatible({
  name: "deepseek",
  baseURL: "https://api.deepseek.com/v1",
  apiKey: process.env.DEEPSEEK_API_KEY!,
});

export const knowledgeAgent = new Agent({
  id: "knowledge-agent",
  name: "Knowledge Agent",
  instructions: `
你是一个基于私人知识库的中文技术问答助手。
回答知识库相关问题时先调用 knowledgeSearchTool 检索。
根据检索到的材料回答，避免编造项目细节。
如果资料不足，请明确说明。
尽可能注明文档来源；不要编造来源链接。
直接回答问题，不要输出“我现在去搜索”等过程说明。
`,
  model: deepseek.chatModel("deepseek-chat"),
  tools: { knowledgeSearchTool },
});
```

这里采用的是 **Agent 自主调用 Tool** 的方式。即使在指令中要求先检索，也不意味着从程序逻辑上强制每次执行检索。若未来要求严格的“先检索再回答”，应该把 `retrieve()` 放到 Agent 调用之前，由应用程序控制检索流程。

创建 `src/scripts/chat.ts`：

```ts
import "dotenv/config";
import { knowledgeAgent } from "../mastra/agents/knowledge.agent";

const question =
  process.argv.slice(2).join(" ") ||
  "Health Master 为什么从 MD5 迁移到 Argon2id？";

const response = await knowledgeAgent.generate(question);
console.log(response.text);
```

运行：

```bash
npm run chat -- "Health Master 为什么从 MD5 迁移到 Argon2id？"
```

在实际测试中，Agent 能找到文章中与 PR #333 相关的内容，并回答出具体的工程决策，例如：

- 引入 `pkg/pwd` 统一密码哈希与验证逻辑。
- 对旧 MD5 密码采用**要求用户重置密码**，而不是登录成功后静默升级哈希的方案。
- 使用 `errcode.ErrLegacyPasswordResetRequired` 表示旧密码需要重置。
- 将 `users.password` 字段扩展为 `character varying(255)`，容纳 Argon2id 的 PHC 字符串。

这些都是知识库文章里的项目细节，比仅仅回答“Argon2id 比 MD5 更安全”更能证明检索增强的实际价值。

## 九、调试 Agent 的 Tool Calls

初次调试时，我尝试这样打印工具调用：

```ts
console.log(call.toolName);
console.log(call.input);
```

结果全是 `undefined`。这并不是 Tool 执行失败，而是**读取错了返回对象的字段路径**。

我随后直接使用 Node.js 的 `inspect()` 打印原始结构：

```ts
import { inspect } from "node:util";

for (const [index, step] of response.steps.entries()) {
  console.log(`\n========== STEP ${index + 1} ==========`);
  console.log("Tool Calls:", inspect(step.toolCalls, { depth: 6, colors: true }));
  console.log("Tool Results:", inspect(step.toolResults, { depth: 6, colors: true }));
  console.log("Step Text:", step.text);
}
```

这次安装版本中，Tool Call 的实际结构是：

```ts
{
  type: "tool-call",
  payload: {
    toolCallId: "call_...",
    toolName: "knowledgeSearchTool",
    args: { query: "Health Master MD5 迁移 Argon2id 原因" }
  }
}
```

Tool Result 的关键结构则是：

```ts
{
  type: "tool-result",
  payload: {
    toolName: "knowledgeSearchTool",
    result: {
      results: [
        { source: "https://...", text: "...", score: 0.7279 }
      ]
    }
  }
}
```

因此要读取：

```ts
call.payload.toolName;
call.payload.args;
result.payload.result.results;
```

### 实际运行结果

这次测试有两个 Step：

**Step 1：两次工具调用**

```text
knowledgeSearchTool
  query: Health Master MD5 迁移 Argon2id 原因

knowledgeSearchTool
  query: 密码哈希算法选择 Argon2id
```

两次调用分别返回 5 个相关 Chunk。第一组最高分约为 `0.728`，第二组最高分约为 `0.731`。返回片段来自同一篇 Markdown 文章，包含迁移原因、密码验证逻辑、Memory-hard、Salt 和旧 MD5 密码处理等内容。

**Step 2：生成答案**

这一步不再调用工具，而是直接输出中文回答，并在末尾列出原始 Markdown URL。最终 `response.text` 与 Step 2 的回答内容一致。

前一次测试曾出现英文开头：

```text
I'll search the knowledge base for information about this migration.
```

说明模型有时会把检索前的说明文本也放进响应中。后续运行中这段英文没有再出现，但这属于模型行为，不应该把提示词约束视为绝对保证。

### 关于来源引用

当前实现将 `source` 存在向量 metadata 中，模型会尝试在回答中引用来源。不过，**最终答案中的引用文字仍然由模型生成**。更严格的做法是由应用程序直接汇总工具返回的 `source` 和 Chunk ID，再与答案一起展示，避免出现模型编造或错误拼接来源的情况。

## 十、package.json 脚本汇总

可以添加以下脚本：

```json
{
  "scripts": {
    "db:init": "tsx src/scripts/init-vector-db.ts",
    "ingest": "tsx src/scripts/ingest.ts",
    "search": "tsx src/scripts/search.ts",
    "chat": "tsx src/scripts/chat.ts"
  }
}
```

建议按以下顺序验证：

```bash
# 1. 确保 PostgreSQL 已启动，并创建 pgvector 扩展

# 2. 首次创建向量表
npm run db:init

# 3. 导入一篇 Markdown
npm run ingest -- ./articles/golang-argon2id-password-hashing.md \
  https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/refs/heads/master/source/_posts/back-end/golang-argon2id-password-hashing.md

# 4. 先验证纯向量检索
npm run search -- "为什么 Argon2id 需要 Salt？"

# 5. 最后测试 Agent
npm run chat -- "Health Master 为什么从 MD5 迁移到 Argon2id？"
```

## 十一、MVP 已经完成什么，还缺什么？

目前已经验证的能力：

| 能力 | 状态 |
| --- | --- |
| Markdown 文本切分 | 已实现 |
| SiliconFlow BGE-M3 Embedding | 已验证，1024 维 |
| PostgreSQL/pgvector 向量存储 | 已验证 |
| 问题向量化与相似度检索 | 已验证 |
| DeepSeek + Mastra Agent | 已验证 |
| Agent 多次调用检索 Tool | 已在日志中确认 |
| 返回文章来源 | 初步实现 |
| 幂等导入 / 增量更新 | 未实现 |
| Markdown 语义切分 | 未实现 |
| Reranking / 检索去重 | 未实现 |
| 引用真实性的程序化校验 | 未实现 |
| 自动化 RAG Evaluation | 未实现 |
| Web UI / 流式聊天 | 未实现 |

下一步我打算优先做 **Document Lifecycle（文档生命周期管理）**：

1. 为每篇文档及 Chunk 生成稳定 ID。
2. 使用内容 Hash 判断文档是否变化，避免无意义的重复 Embedding。
3. 文档更新时替换旧 Chunk；文档删除时清理对应向量。
4. 改进 Markdown 切分，保留标题层级和代码块上下文。
5. 再考虑 Reranking、质量评估和 Web UI。

## 十二、总结

这次实现最大的收获，不是“成功调用了一个大模型 API”，而是把 RAG 拆成了几个能够独立验证的环节：

- Embedding 失败时，先检查 API URL、环境变量和最小 SDK 调用。
- 向量入库失败时，检查数据库扩展和 Mastra 对应的索引/表是否存在。
- Agent 回答不准确时，先看检索结果，再看 Tool Calls，最后才分析提示词。
- 不要把相似度分数误认为正确率，也不要把模型输出的引用当成天然可信的来源证明。

对于个人技术博客知识库而言，这已经是一个可运行的 RAG MVP。下一阶段的重点不应急着堆叠功能，而是让文档导入、更新、检索和引用变得稳定、可重复、可观察。

## 参考资料

- [Mastra 官方文档](https://mastra.ai/docs)
- [DeepSeek API 文档](https://api-docs.deepseek.com/)
- [SiliconFlow 文档](https://docs.siliconflow.cn/)
- [pgvector](https://github.com/pgvector/pgvector)
- [Health Master PR #333](https://github.com/damingerdai/health-master/pull/333)
- [从密码哈希迁移到 Argon2id：一次 Go 项目中的密码安全实践（Markdown 原文）](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/refs/heads/master/source/_posts/back-end/golang-argon2id-password-hashing.md)

## 源代码

[mastra-rag-app](https://github.com/damingerdai/mastra-rag-app)