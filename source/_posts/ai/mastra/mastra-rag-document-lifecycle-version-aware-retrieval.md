---
title: Mastra RAG 进阶：使用 Drizzle ORM 和 pgvector 实现文档版本管理与 Version-aware Retrieval
date: 2026-10-20 16:30:00
description: 在 Mastra RAG MVP 基础上，通过 Drizzle ORM、SHA-256 内容去重和当前版本指针实现文档版本管理与版本感知检索，附真实测试与工程边界。
tags:
  - Mastra
  - RAG
  - TypeScript
  - Drizzle ORM
  - PostgreSQL
  - pgvector
categories:
  - AI
---

# Mastra RAG 进阶：使用 Drizzle ORM 和 pgvector 实现文档版本管理与 Version-aware Retrieval

## 前言

在上一篇文章《[从零构建 Mastra RAG MVP：DeepSeek、SiliconFlow 与 PostgreSQL/pgvector 实战](https://blog.damingerdai.com/2026/10/18/ai/mastra/mastra-rag-mvp-deepseek-siliconflow-pgvector/)》中，我使用 Mastra、DeepSeek、SiliconFlow BGE-M3 和 PostgreSQL/pgvector 搭建了最小可运行的 RAG 知识库。

最初的链路是：Markdown → Chunking → Embedding → pgvector → Similarity Search → Agent。它能够完成基础检索，却没有解决文档更新后的版本问题。

**当一篇文章从 v1 更新为 v2，旧版向量仍在 pgvector 中时，怎样保证 RAG 不会把旧内容当成最新知识？**

这次在原有 POC 上引入 Drizzle ORM，增加三张文档元数据表，实现内容去重、版本创建、发布指针和 Version-aware Retrieval。本文基于实际 TypeScript 代码与命令输出，同时明确没有实现的生产级能力。

## 一、问题：相似度不等于有效性

假设第一次导入一篇 Go Argon2id 文章，生成 v1 的 35 个 Chunk；更新文章后，生成 v2 的 36 个 Chunk。若两个版本的向量都留在同一个索引，pgvector 可能将 v1 的某个 Chunk 排在 v2 前面。

向量检索只负责判断**哪些内容与问题相似**，无法单凭相似度判断**哪些内容属于当前发布版本**。

因此需要独立的 Document Registry：

```text
Document A
├── v1 (35 Chunks)
└── v2 (36 Chunks) ← currentVersionId

PgVector: v1 和 v2 的向量可以同时存在
Retrieval: 只允许当前发布版本的 Chunk 通过
```

## 二、技术栈与数据模型

| 组件 | 技术 | 职责 |
|---|---|---|
| RAG / Agent | Mastra | 文档处理与工具调用 |
| Embedding | SiliconFlow `BAAI/bge-m3` | 生成 1024 维向量 |
| Vector Store | Mastra `PgVector` | 相似度检索 |
| 数据库 | PostgreSQL + pgvector | 保存向量和文档元数据 |
| ORM | Drizzle ORM | 管理文档、版本、Chunk |
| 内容 | Markdown | 知识库输入 |

三张表的关系如下：

```text
documents
    │ 1:N
    ▼
document_versions
    │ 1:N
    ▼
document_chunks ── vector_id ──► PgVector
```

### 2.1 documents：逻辑文档与当前发布指针

```ts
export const documents = pgTable("documents", {
  id: uuid("id").primaryKey().default(sql`uuidv7()`),
  sourceUri: text("source_uri").notNull().unique(),
  title: text("title"),
  status: text("status").notNull().default("pending"),
  currentVersionId: uuid("current_version_id"),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow(),
  deletedAt: timestamp("deleted_at", { withTimezone: true }),
});
```

同一 `sourceUri` 始终代表同一篇逻辑文档。`currentVersionId` 指向当前**已发布**版本，而不是版本号最大的记录。例如 v3 正在索引时，指针仍可指向 v2。

> `uuidv7()` PostgreSQL18的新特性，如果你的pg版本不支持，可以把*default(sql`uuidv7()`)*,使用改成*defaultRandom()*

### 2.2 document_versions：内容、处理管线和模型版本

```ts
export const documentVersions = pgTable("document_versions", {
  id: uuid("id").primaryKey().default(sql`uuidv7()`),
  documentId: uuid("document_id").notNull().references(() => documents.id),
  versionNumber: integer("version_number").notNull(),
  contentHash: text("content_hash").notNull(),
  pipelineVersion: text("pipeline_version").notNull(),
  embeddingModel: text("embedding_model").notNull(),
  status: text("status").notNull().default("pending"),
  chunkCount: integer("chunk_count").default(0),
  errorMessage: text("error_message"),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
  publishedAt: timestamp("published_at", { withTimezone: true }),
}, (table) => [unique().on(table.documentId, table.versionNumber)]);
```

这里保留三个决定是否需要重新索引的维度：`contentHash`、`pipelineVersion`、`embeddingModel`。即使文章没有变化，修改切分策略或 Embedding 模型也可能需要重新索引。

### 2.3 document_chunks：Vector ID 到版本的关联

```ts
export const documentChunks = pgTable("document_chunks", {
  id: text("id").primaryKey(),
  documentVersionId: uuid("document_version_id")
    .notNull().references(() => documentVersions.id),
  chunkIndex: integer("chunk_index").notNull(),
  contentHash: text("content_hash").notNull(),
  vectorId: text("vector_id").notNull().unique(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
}, (table) => [unique().on(table.documentVersionId, table.chunkIndex)]);
```

`document_chunks` 不需要重复保存 Chunk 正文，而是维护 `vectorId → documentVersionId → documentId` 的关系。

数据库迁移使用 Drizzle Kit：

```bash
npm run db:generate -- --name describe_change
npm run db:migrate
```

## 三、导入：内容去重与版本创建

### 3.1 通过 sourceUri 识别文档

```ts
await db.insert(documents)
  .values({ sourceUri, title })
  .onConflictDoNothing({ target: documents.sourceUri });
```

来源地址具有唯一约束。重复导入时先找到同一个 Document，再判断是否需要生成新版本。

### 3.2 SHA-256 内容哈希

```ts
import { createHash } from "node:crypto";

export function sha256(content: string) {
  return createHash("sha256").update(content, "utf8").digest("hex");
}
```

导入时比较当前发布版本：

```ts
if (
  currentVersion?.contentHash === contentHash &&
  currentVersion.pipelineVersion === PIPELINE_VERSION &&
  currentVersion.embeddingModel === EMBEDDING_MODEL &&
  currentVersion.status === "active"
) {
  return {
    status: "unchanged" as const,
    documentId: document.id,
    versionId: currentVersion.id,
  };
}
```

重复内容不进入重新 Embedding 的分支。这里验证的是顺序执行下的去重；完整并发幂等还需要额外设计。

### 3.3 版本号分配

版本创建放在 Drizzle 事务中，并对 Document 行加锁：

```ts
return db.transaction(async (tx) => {
  const [document] = await tx.select()
    .from(documents)
    .where(eq(documents.id, input.documentId))
    .for("update");

  if (!document || document.deletedAt) {
    throw new Error("Document not available");
  }

  const [latest] = await tx.select({
    versionNumber: documentVersions.versionNumber,
  }).from(documentVersions)
    .where(eq(documentVersions.documentId, input.documentId))
    .orderBy(desc(documentVersions.versionNumber))
    .limit(1);

  const [version] = await tx.insert(documentVersions).values({
    documentId: input.documentId,
    versionNumber: (latest?.versionNumber ?? 0) + 1,
    contentHash: input.contentHash,
    pipelineVersion: input.pipelineVersion,
    embeddingModel: input.embeddingModel,
    status: "pending",
  }).returning();

  if (!version) throw new Error("Failed to create version");
  return version;
});
```

`FOR UPDATE` 用来序列化同一 Document 的版本号分配；它不等于完整的 Ingestion 分布式锁。

## 四、索引：Mastra Chunking、Embedding 和元数据

```ts
const doc = MDocument.fromMarkdown(content);
const chunks = await doc.chunk({
  strategy: "recursive",
  maxSize: 512,
  overlap: 50,
});
const texts = chunks.map((chunk) => chunk.text);

const { embeddings } = await embedMany({
  model: embeddingModel,
  values: texts,
});

if (
  embeddings.length !== texts.length ||
  embeddings.some((vector) => vector.length !== 1024)
) {
  throw new Error("Invalid embedding dimensions");
}

const ids = texts.map((_, index) => makeVectorId(versionId, index));

await vectorStore.upsert({
  indexName: INDEX_NAME,
  ids,
  vectors: embeddings,
  metadata: texts.map((text, index) => ({
    docId: documentId,
    versionId,
    chunkIndex: index,
    text,
    source,
  })),
});

await saveDocumentChunks({ versionId, texts, vectorIds: ids });
```

`versionId` 会被写入向量 Metadata，但**Metadata 里的版本标识不是版本是否有效的权威依据**。真正的发布状态由 Drizzle 管理的 `documents.currentVersionId` 决定。

## 五、发布：先索引，再切换当前版本

流程如下：

```text
Create Version (pending)
    ↓
Mark processing
    ↓
Chunk + Embedding + Vector Upsert
    ↓
Save document_chunks
    ↓
Publish Version (active)
    ↓
Update documents.currentVersionId
```

发布时在同一个数据库事务中更新版本状态和文档指针：

```ts
await tx.update(documentVersions).set({
  status: "active",
  chunkCount,
  publishedAt: new Date(),
}).where(eq(documentVersions.id, versionId));

await tx.update(documents).set({
  currentVersionId: versionId,
  status: "active",
  updatedAt: new Date(),
}).where(eq(documents.id, documentId));
```

这样 v2 尚未完成索引时，`currentVersionId` 仍然可以指向 v1。**最新创建版本**与**当前发布版本**是两个不同概念。

需要注意：这里的原子性只覆盖元数据发布事务；Embedding API、Vector Upsert 与数据库元数据写入并不共享一个端到端事务。

## 六、Version-aware Retrieval：核心设计

当前 POC 使用两阶段检索：

```text
User Query
    ↓
BGE-M3 Query Embedding
    ↓
PgVector Top 50 Candidates
    ↓
Drizzle JOIN + currentVersionId 校验
    ↓
过滤过期版本
    ↓
有效 Top 5
```

### 6.1 生成 Query Embedding

```ts
const { embedding } = await embed({
  model: embeddingModel,
  value: query,
});
```

`embed()` 只生成向量，不负责查询数据库，也不负责生成最终回答。

### 6.2 pgvector 检索候选

```ts
const candidates = await vectorStore.query({
  indexName: INDEX_NAME,
  queryVector: embedding,
  topK: candidateLimit,
});

const candidateIds = candidates.map((item) => item.id);
```

此时得到的是按相似度排序的候选 Chunk，其中可能同时包含 v1 和 v2。

### 6.3 Drizzle 验证当前发布版本

```ts
const validChunks = await db
  .select({
    vectorId: documentChunks.vectorId,
    documentId: documents.id,
    versionId: documentVersions.id,
  })
  .from(documentChunks)
  .innerJoin(
    documentVersions,
    eq(documentChunks.documentVersionId, documentVersions.id),
  )
  .innerJoin(
    documents,
    and(
      eq(documentVersions.documentId, documents.id),
      eq(documents.currentVersionId, documentVersions.id),
    ),
  )
  .where(and(
    inArray(documentChunks.vectorId, candidateIds),
    eq(documents.status, "active"),
    isNull(documents.deletedAt),
    eq(documentVersions.status, "active"),
  ));
```

最关键的条件是：

```ts
eq(documents.currentVersionId, documentVersions.id)
```

这句不是在比较 `versionNumber`，而是在比较**当前发布指针**和**Chunk 所属版本 ID**。

| Vector | 所属版本 | 当前版本 | 结果 |
|---|---|---|---|
| vector-001 | v1 | v2 | 排除 |
| vector-002 | v1 | v2 | 排除 |
| vector-003 | v2 | v2 | 保留 |
| vector-004 | v2 | v2 | 保留 |

### 6.4 Map 过滤并保持相似度顺序

```ts
const validMap = new Map(
  validChunks.map((chunk) => [chunk.vectorId, chunk]),
);

const results: SearchResult[] = [];
for (const candidate of candidates) {
  const valid = validMap.get(candidate.id);
  if (!valid) continue;

  const score = candidate.score ?? 0;
  if (score < minScore) continue;

  const metadata = candidate.metadata ?? {};
  if (
    typeof metadata.text !== "string" ||
    typeof metadata.source !== "string" ||
    typeof metadata.chunkIndex !== "number"
  ) continue;

  results.push({
    id: candidate.id,
    text: metadata.text,
    score,
    documentId: valid.documentId,
    versionId: valid.versionId,
    source: metadata.source,
    chunkIndex: metadata.chunkIndex,
  });

  if (results.length >= topK) break;
}
return results;
```

批量 JOIN 避免了逐个 Vector ID 查询的 N+1 问题；Map 过滤则保留 pgvector 返回的原始排序。

## 七、可复现实验：v1 → unchanged → v2

### 7.1 首次导入

```bash
npx tsx src/scripts/ingest.ts
```

实际输出：

```json
{
  "status": "indexed",
  "documentId": "01a1585d-2b04-7d12-bbcb-8cd5710a166e",
  "versionId": "01a1585d-2b0a-7fd0-b143-80c64fe8aa59",
  "versionNumber": 1,
  "chunkCount": 35
}
```

### 7.2 重复导入

```bash
npx tsx src/scripts/ingest.ts
```

实际输出：

```json
{
  "status": "unchanged",
  "documentId": "01a1585d-2b04-7d12-bbcb-8cd5710a166e",
  "versionId": "01a1585d-2b0a-7fd0-b143-80c64fe8aa59"
}
```

### 7.3 修改内容，生成 v2

为了不修改线上博客，可以在测试脚本中为抓取的 Markdown 追加内容：

```ts
const originalContent = await response.text();
const content = originalContent + `

## Version 2 Test

This document version introduces an updated
Argon2id implementation for lifecycle testing.
`;
```

执行：

```bash
npx tsx src/scripts/ingest2.ts
```

实际输出：

```json
{
  "status": "indexed",
  "documentId": "01a1585d-2b04-7d12-bbcb-8cd5710a166e",
  "versionId": "01a1586b-ed40-7a87-947c-b1bab677c6fd",
  "versionNumber": 2,
  "chunkCount": 36
}
```

Document ID 不变，Version ID 更新，说明系统识别了同一逻辑文档的新版本。

### 7.4 检索当前版本

```bash
npx tsx src/scripts/test-retrieval.ts
```

测试 Query：`Go 语言中如何使用 Argon2id 进行密码哈希？`

| Result | Score | Version |
|---|---:|---|
| 1 | 0.770230952973452 | v2 |
| 2 | 0.7123205539320796 | v2 |
| 3 | 0.7062564598416385 | v2 |
| 4 | 0.6941330516975417 | v2 |
| 5 | 0.6928984748194693 | v2 |

五条结果的 `versionId` 均为：

```text
01a1586b-ed40-7a87-947c-b1bab677c6fd
```

这与 v2 导入返回的 Version ID 一致。**在当前测试数据上，最终结果没有混入 v1。**

还可以补充自动化断言：

```ts
import assert from "node:assert/strict";

const expectedVersionId = "01a1586b-ed40-7a87-947c-b1bab677c6fd";
const results = await searchKnowledge(
  "Go 语言中如何使用 Argon2id 进行密码哈希？",
  { topK: 5, candidateLimit: 100 },
);

assert.ok(results.length > 0, "No results found");
assert.ok(
  results.every((result) => result.versionId === expectedVersionId),
  "Outdated version detected",
);
```

这段断言是可补充的测试代码；前面的实际检索输出来自手动运行的脚本。要长期稳定复现，最好使用隔离测试数据库与固定的 Markdown 测试数据。

## 八、明确的工程边界

### 8.1 先 Top K 再过滤可能漏召回

当前先检索 50 个候选，再通过 Drizzle 过滤。如果前 50 个恰好全部来自旧版本，最终可能返回空结果，即使有效版本中存在相关 Chunk。

更严格的方案是在 PostgreSQL 查询中同时完成向量相似度排序与当前版本约束。实际落地时需要检查 Mastra 向量表结构、查询计划和 pgvector 近似索引的过滤行为。

### 8.2 索引与发布不是端到端原子操作

Embedding API、向量写入、Chunk 注册和版本发布跨越多个步骤。当前失败版本可以标记为 `failed`，但残留向量清理、可靠重试和并发导入去重尚未完整实现。

### 8.3 历史版本向量仍占空间

v1 与 v2 的向量可以同时保留。这有利于历史追踪，但也意味着存储成本随版本增长。后续可以引入版本保留策略、垃圾回收或未变化 Chunk 复用。

### 8.4 Markdown Front Matter 影响检索内容

实际结果中，Hexo YAML Front Matter 也参与了 Chunk 切分。后续可以先解析 `title`、`tags`、`categories`、`date`，把它们保存为文档元数据，再仅对正文生成 Embedding。

### 8.5 未实现的功能

本次 POC 没有实现 Soft Delete/Restore、完整失败恢复、Vector Garbage Collection、跨版本增量索引，也没有验证高并发或大规模历史版本下的召回率。

## 九、总结

本次实现并不是再做一个 RAG Demo，而是为已有 Mastra RAG MVP 增加了明确的版本语义：

```text
v1 导入 → 35 Chunks
相同内容再次导入 → unchanged
修改内容 → v2 / 36 Chunks
Version-aware Retrieval → Top 5 全部来自 v2
```

最重要的设计选择，是使用 `documents.currentVersionId` 表示当前已发布版本，而不是直接信任向量 Metadata 或最大版本号。

**RAG 中的向量相似度负责回答“哪些内容相关”，而 Document Registry 负责回答“哪些内容现在有效”。**

对于当前 POC，这已经足够。等真正遇到可靠性、规模或运维需求时，再逐步引入数据库侧版本过滤、失败恢复与向量垃圾回收。
