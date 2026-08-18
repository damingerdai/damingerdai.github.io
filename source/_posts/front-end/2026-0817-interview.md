---
title: 2026年08月17号面试汇总
date: 2026-08-18 15:51:42
tags: [高级前端工程师, 面试]
categories: [前端]
---

# HTTP里的缓存

HTTP 协议中的缓存机制主要分为两大类：**强缓存**（本地缓存）和**协商缓存**（对比缓存）。它们通过 HTTP 报文头部的字段来控制。

---

### 一、 强缓存（Strong Caching）

无需向服务器发送请求，浏览器直接从本地缓存（内存或磁盘）中读取资源，HTTP 状态码为 **200 OK (from disk cache / from memory cache)**。

* **Cache-Control**（HTTP/1.1，优先级最高）
* `max-age=<seconds>`：设置缓存的最大有效时间（秒）。
* `no-cache`：跳过强缓存，强制向服务器发起协商缓存。
* `no-store`：禁止任何缓存，每次都必须重新下载。
* `private`：仅允许终端用户的浏览器缓存（默认值）。
* `public`：允许中间代理服务器（如 CDN）和浏览器缓存。
* `s-maxage=<seconds>`：仅对公共代理/CDN 生效的缓存时长。


* **Expires**（HTTP/1.0，过时）
* 指定资源失效的具体绝对时间（GMT 格式）。受客户端本地时间不准确的影响。



---

### 二、 协商缓存（Validation Caching）

强缓存失效或配置为 `no-cache` 时，浏览器向服务器发送带有缓存标识的请求，由服务器决定是否使用本地缓存：

* 如果资源未修改，服务器返回 **304 Not Modified**，浏览器使用本地缓存。
* 如果资源已修改，服务器返回 **200 OK** 及最新资源。
* **ETag / If-None-Match**（推荐，基于内容指纹）
* **响应头 ETag**：服务器根据文件内容生成的唯一哈希/标识符。
* **请求头 If-None-Match**：浏览器再次请求时带上上次获得的 ETag 值，由服务器对比。


* **Last-Modified / If-Modified-Since**（基于最后修改时间）
* **响应头 Last-Modified**：资源在服务器上的最后修改时间。
* **请求头 If-Modified-Since**：浏览器再次请求时带上该时间，服务器对比文件时间戳是否更新。



---

### 三、 强缓存与协商缓存对比

| 特性 | 强缓存 (Cache-Control / Expires) | 协商缓存 (ETag / Last-Modified) |
| --- | --- | --- |
| **是否发请求** | 否（直接读取本地） | 是（发送轻量验证请求） |
| **返回状态码** | `200 OK` (from cache) | `304 Not Modified`（未修改） / `200 OK`（已修改） |
| **优先控制字段** | `Cache-Control` | `ETag` (If-None-Match) |
| **性能损耗** | 最低（无需网络开销） | 极低（仅产生 RTT，通常无响应体） |

---

### 四、 刷新操作对缓存的影响

* **地址栏输入 URL / 点击链接**：同时触发强缓存和协商缓存。
* **F5 / 点击刷新按钮**：跳过强缓存，强制向服务器发起协商缓存。
* **Ctrl + F5 / 强制刷新**：跳过强缓存和协商缓存，重新下载所有资源（请求头会自动带上 `Cache-Control: no-cache`）。

# React的渲染过程

React 的渲染过程主要分为**三大阶段**：**Render 阶段（计算变化）**、**Commit 阶段（更新 DOM）** 以及 **Browser Paint 阶段（浏览器绘制）**。在 React 16+（Fiber 架构）引入后，这个过程变成了异步可中断的调和过程。

---

### 一、 核心三阶段

```
[ Trigger 触发 ] ──> [ Render 计算 ] ──> [ Commit 提交 ] ──> [ Browser Paint 绘制 ]

```

#### 1. 触发阶段（Trigger）

渲染的起始点，常见触发条件：

* **初次渲染**：调用 `createRoot(container).render(<App/>)`。
* **状态更新**：调用 `setState` / `dispatch`（`useState`, `useReducer`）。
* **Context / 父组件重新渲染**：传递给子组件的 props 变化或祖先 Context 更新。

#### 2. 渲染阶段（Render / Reconciliation）

此阶段的主要任务是**找出需要更新的部分**，纯 JavaScript 计算，不涉及真实 DOM 操作（**可中断、可复用**）：

* **构建 WorkInProgress Fiber 树**：从根节点开始遍历组件树。
* **执行组件函数**：获取最新的 Virtual DOM（JSX）。
* **Diff 算法对比**：对比新旧 Fiber 树的节点差异：
* **单节点**：key 和 type 相同则复用，不同则销毁重建。
* **多节点**：优先复用旧节点，处理节点位置移动、插入或删除。


* **打上 Side-Effect 标记（Flags）**：如 `Placement`（新增）、`Update`（更新）、`Deletion`（删除）。

#### 3. 提交阶段（Commit）

此阶段将 Render 阶段计算出的变更**同步应用到真实 DOM 上**（**同步执行，不可中断**）：

* **Before Mutation（突变前）**：执行 `getSnapshotBeforeUpdate`，处理 DOM 突变前的状态捕捉。
* **Mutation（突变）**：执行真实 DOM 的增删改操作；更新 DOM 引用（ref）。
* **Layout（突变后）**：DOM 已更新但浏览器未绘制。**同步执行 `useLayoutEffect**` 的回调函数。

---

### 二、 浏览器绘制（Browser Paint）

Commit 阶段结束后，主线程交还给浏览器，浏览器根据更新后的真实 DOM 树和 CSSOM 树计算布局并绘制像素到屏幕上。

绘制完成后，React 会**异步触发 `useEffect**` 的回调函数，避免阻塞页面的渲染。

---

### 三、 总结：生命周期与 Hook 的执行时机

| 阶段 | 对应执行的代码/Hook | 特性 |
| --- | --- | --- |
| **Render** | 函数组件体、`useMemo`、`useCallback` | 纯计算、可多次执行/中断、无 DOM 操作 |
| **Commit (Mutation)** | 真实 DOM 变更操作 | 同步、不可中断 |
| **Commit (Layout)** | `useLayoutEffect` | 同步执行，会阻塞页面绘制 |
| **Post-Paint** | `useEffect` | 异步执行，页面绘制完成后调用，不阻塞 UI |

---

# 前端的状态管理

前端状态管理，就是管理“数据在什么时候发生变化，以及数据变化后哪些 UI 需要同步更新”。

状态管理的核心是把应用中的状态集中或有规则地组织起来，并建立“状态变化 → UI 更新”的可预测数据流。

--

# RAG（检索增强生成）知识库的准确率

要提高 RAG（检索增强生成）知识库的准确率，核心在于解决“检索不到/检索不准”（Retrieval）**以及**“幻觉/未能忠实回答”（Generation）这两个阶段的问题。可以从以下 5 个关键环节进行针对性优化：

---

### 一、 数据清洗与预处理（清洗源头）

* **高质量数据提取**：过滤掉 HTML 标签、格式错乱字符以及大量重复信息；对于 PDF/表格等结构化/半结构化文档，使用专业的解析工具（如 Unstructured、MinerU）将表格提取为 Markdown 格式，避免丢失逻辑关联。
* **富化文档元数据（Metadata）**：在文档切片时保留文件名、章节层级、创建时间、适用人群等字段，方便在检索时结合元数据进行精确过滤。

---

### 二、 优化切片策略（Chunking）

* **语义切片（Semantic Chunking）**：避免简单按照固定字符数（如 500 字）强行截断，改为依据段落、标点或语义相似度变化进行动态切分，保证单个 Chunk 包含完整独立的语义。
* **父子切片 / 小块检索大块返回（Parent-Child / Small-to-Big Retrieval）**：
* 将文档切分成极小的 Chunk（如 100~200 Tokens）用于高精度向量匹配。
* 匹配成功后，将该小 Chunk 对应的更大上下文（如包含完整上下文的父级段落或整个章节）提交给 LLM 组装回答。



---

### 三、 检索与重排序优化（Retrieval & Rerank）

* **混合检索（Hybrid Search）**：同时使用**向量检索（Dense Retrieval，理解语义）**和**关键词检索（Sparse Retrieval/BM25，精确匹配专有名词、型号、ID）**，并将两者结果融合（如通过 RRF 算法）。
* **引入重排序（Rerank）**：先拉取前 20~50 个候选 Chunk，再使用 Cross-Encoder 重排序模型（如 BGE-Reranker、Cohere Rerank）根据 Query 与 Chunk 的细粒度相关性重新评分，仅选取 Top 3~5 进最终 Prompt。这是提升准确率性价比最高的方式。
* **Query 改写与膨胀**：
* **HyDE（假设性文档嵌入）**：让 LLM 先针对用户问题生成一个“假设性回答”，再用该回答去检索相似文档。
* **Query 重构/扩展**：利用 LLM 纠正用户的错别字、补充指代不清的上下文，或将复杂问题拆解为多个子问题分别检索。



---

### 四、 Prompt 工程与生成控制（Generation）

* **严格防幻觉指令（Guardrails）**：在 Prompt 中设置明确边界，如：
> “请严格仅根据提供的上下文（Context）回答问题。如果上下文不足以回答该问题，请直接回答‘知识库中暂未包含相关信息’，切勿自行发挥或引入外部知识。”


* **引用溯源（Source Citation）**：要求模型在输出结论的同时标注引用的具体文档片段序号或页码，便于人工复核与追溯。

---

### 五、 引入评估与反馈闭环（Evaluation）

* **使用 RAG Triad（RAG 三元组）评估框架**：利用 Ragas 或 TruLens 等工具对系统进行持续打分：
* **Context Relevance**：检索到的 Chunk 和 Query 的相关程度。
* **Groundedness**：生成的回答是否全部源于检索到的上下文（防幻觉）。
* **Answer Relevance**：生成的回答是否准确响应了用户的真实意图。


* **Bad Case 归因与迭代**：建立用户反馈机制（点赞/点踩），收集答非所问的具体案例，归因是“没检索到”、“检索到了但重排序靠后”还是“模型理解偏差”，从而倒逼索引与参数调优。

---