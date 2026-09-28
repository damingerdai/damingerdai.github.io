---
title: Mastra 入门：从零搭建旅行助手 Agent
date: 2026-09-28 17:19:14
tags: [AI, Mastra]
categories: [AI]
---

# Mastra 入门：从零搭建旅行助手 Agent

本文面向熟悉 TypeScript、刚接触 Mastra 的开发者。完成后，你会有一个能追问旅行需求、生成行程，并按需调用**模拟天气工具**的 Agent。示例使用 Bun 和 DeepSeek Flash；天气数值固定，仅用于学习工具调用。

## 1. 先理解三个概念

| 概念 | 在本例中的职责 |
| --- | --- |
| Agent | 配置模型、指令和可用工具，处理用户对话 |
| Tool | 接受经过 Schema 约束的输入，执行代码，返回结构化结果 |
| Studio | 本地可视化调试界面，试聊 Agent、查看工具调用与运行过程 |

Agent 可以决定是否请求调用工具；工具的 `execute` 才负责真正执行代码。当前示例没有持久化 Memory、实时天气、车票和酒店查询，也没有保证跨新会话记住偏好。更完整的 Workflow、Memory、RAG、MCP 可以以后逐步加入。

## 2. 初始化项目

确认 Node.js 满足当前 Mastra 文档的最低要求（教程所列为 22.13.0），并检查 Bun：

```bash
node -v
bun -v
```

最可控的入门方式是手动安装。这样项目目录不会随脚手架安装超时而消失：

```bash
mkdir travel-agent
cd travel-agent
bun init -y
bun add @mastra/core zod
bun add -d mastra typescript @types/node
mkdir -p src/mastra/agents src/mastra/tools
```

在 `package.json` 的 `scripts` 中加入 `"dev": "mastra dev"`。不要覆盖 Bun 初始化生成的其他字段。例如：

```json
{
  "scripts": {
    "dev": "mastra dev"
  }
}
```

如果 `bun add` 长时间停在依赖解析，可以尝试 `bun add ... --registry https://registry.npmjs.org`，或根据自己网络情况改用镜像。另一个选择是 `bun create mastra@latest`，但脚手架的自动安装在你的环境曾于 60 秒超时并移除了目录；本文因此采用手动创建。

项目完成后的核心结构：

```text
travel-agent/
├── .env
├── .gitignore
├── package.json
└── src/mastra/
    ├── index.ts
    ├── agents/travel-agent.ts
    └── tools/city-weather-tool.ts
```

## 3. 配置模型密钥

在 DeepSeek 开放平台申请 API Key，在项目根目录创建 `.env`：

```dotenv
DEEPSEEK_API_KEY=替换成你的密钥
```

确保 `.gitignore` 至少包含 `.env` 与 `node_modules/`，不要提交真实密钥。DeepSeek 官方当前给 Flash 的 API 模型名是 `deepseek-flash`；Mastra 的模型标识使用供应商前缀，因此示例写作 `deepseek/deepseek-flash`。若安装版本报告无法识别该标识，先查 Mastra 模型目录并升级相关包，不要猜测旧模型名。

## 4. 创建模拟天气工具

文件 `src/mastra/tools/city-weather-tool.ts`：

```ts
import { createTool } from '@mastra/core/tools'
import { z } from 'zod'

export const cityWeatherTool = createTool({
  id: 'get-city-weather',
  description: '返回演示用模拟天气；不是真实天气或预报，仅用于测试工具调用',
  inputSchema: z.object({
    city: z.string().describe('中文或英文城市名'),
  }),
  outputSchema: z.object({
    city: z.string(),
    summary: z.string(),
    temperatureCelsius: z.number(),
    travelAdvice: z.string(),
    isMock: z.boolean(),
  }),
  execute: async ({ city }) => ({
    city,
    summary: '晴，微风',
    temperatureCelsius: 24,
    travelAdvice: '模拟情境下适合步行；实际出行请核对天气预报。',
    isMock: true,
  }),
})
```

`inputSchema` 约束模型传入的参数；`outputSchema` 描述返回数据的结构。`execute` 是实际运行的函数。这里所有城市都返回同一组数值，因此必须让用户知道它是模拟数据。

## 5. 创建 Agent 并注册

文件 `src/mastra/agents/travel-agent.ts`：

```ts
import { Agent } from '@mastra/core/agent'
import { cityWeatherTool } from '../tools/city-weather-tool'

export const travelAgent = new Agent({
  id: 'travel-agent',
  name: 'Travel Agent',
  description: '帮助用户规划城市短途旅行',
  instructions: `
    你是一名严谨的中文旅行规划助手。
    收集出发地、目的地、旅行天数、预算和旅行偏好。
    缺少关键信息时提问，不重复询问已有信息。
    信息充分时给出每日行程、交通、住宿、餐饮、景点和预算拆分。
    不要编造实时票价、酒店价格、交通班次或景点开放时间。
    用户明确要测试模拟天气或要求根据模拟天气安排活动时，使用 cityWeatherTool。
    工具返回的天气只是模拟数据，必须在回复中明确说明，不能当成真实天气预报。
    用户询问真实天气时，说明当前没有实时天气工具。
    全程使用中文。
  `,
  model: 'deepseek/deepseek-flash',
  tools: { cityWeatherTool },
})
```

文件 `src/mastra/index.ts`：

```ts
import { Mastra } from '@mastra/core'
import { travelAgent } from './agents/travel-agent'

export const mastra = new Mastra({
  agents: { travelAgent },
})
```

`Agent` 定义行为，`Mastra` 将它注册到项目。工具注册在 Agent 的 `tools` 属性中；只在指令里提及一个工具并不能让模型调用它。

## 6. 启动与验收

```bash
bun run dev
```

按终端输出打开本地 Studio，通常是 <http://localhost:4111>。进入 Agents → Travel Agent，逐条测试：

| 输入 | 预期观察 |
| --- | --- |
| `你好` | 正常对话，不调用天气工具 |
| `我想从上海去杭州玩两天` | 追问预算和偏好 |
| `帮我测试杭州的模拟天气` | 工具调用参数包含 `city: "杭州"`；回答注明模拟 |
| `上海出发，杭州两天一夜，预算 1800 元，喜欢安静路线` | 给出行程与预算，价格属于估算，不能伪装为实时价格 |

观察 Studio 中的工具调用记录：是否出现 `get-city-weather`、输入城市是否正确、输出是否包含 `isMock: true`。只看最终文本不能确认工具确实被调用。模型选择工具具有非确定性；如果第三条没有调用，先检查 Agent 是否注册了工具、指令是否明确以及模型是否支持工具调用。

## 7. 常见问题与下一步

- **脚手架安装超时并删除项目：**使用上面的手动安装步骤；命令行直接执行 `bun add`，不要依赖脚手架的自动安装超时。
- **提示缺少密钥或模型不可用：**检查 `.env` 是否位于项目根目录、密钥名称及模型标识；参考 Mastra 模型目录核对当前支持情况。
- **连续对话忘记先前信息：**先确认自己在同一个 Studio 会话内；跨会话长期保存偏好需要另行配置 Memory 与存储。
- **想做真实旅行助手：**替换模拟工具为可靠的天气 API，处理请求失败、时间范围和城市消歧，再接入交通、住宿数据；区分实时结果、估算和模型建议。

## 参考资料

1. [Mastra 官方手动安装](https://mastra.ai/reference/manual-install)
2. [Mastra 官方工具文档](https://mastra.ai/docs/agents/tools)
3. [Mastra 模型目录](https://mastra.ai/models)
4. [DeepSeek 官方模型说明](https://api-docs.deepseek.com/quick_start/pricing/)
5. [Mastra 中文教程：动手构建](https://beamuswayne.github.io/Mastra-Tutorial/part-2-build/)；[教程源码与旅行助手示例](https://github.com/BeamusWayne/Mastra-Tutorial)
