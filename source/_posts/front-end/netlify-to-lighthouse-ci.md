---
title: 从 Netlify 部署失败到 Lighthouse CI：在 GitHub Actions 中建立前端质量门禁
date: 2026-10-24 15:30:00
description: 从 Netlify Lighthouse 插件与 Node.js 22 不兼容的真实部署故障出发，记录如何在 Vite + React + Bun 项目中独立运行 Lighthouse CI、控制检测范围、设置质量门槛、生成 GitHub Actions Summary，以及踩坑排查。
tags:
  - Lighthouse
  - GitHub Actions
  - CI/CD
  - Netlify
  - Vite
  - Bun
categories: [前端]
---

## 前言：Netlify 部署失败，问题竟然出在 Lighthouse 插件

最近一次 Netlify 部署失败，让我重新思考了前端项目的 CI/CD 质量检查应该放在哪里。

当时的错误日志非常明确：

```text
❯ Context
  production

❯ Installing plugins
   - @netlify/plugin-lighthouse@4

Configuration error

The Node.js version is 22.23.3 but the plugin
"@netlify/plugin-lighthouse" requires >=14.15 <20

Failed during stage 'building site':
Build script returned non-zero exit code: 2
```

Netlify 使用 Node.js `22.23.3`，但 `@netlify/plugin-lighthouse@4` 声明的版本范围是 `>=14.15 <20`。因此失败发生在**插件配置检查阶段**，而不是 Lighthouse 真正执行审计后发现页面分数不合格。

一个直接的办法是降级 Node.js，另一个办法是寻找兼容的新插件版本。但我不想为了一个质量检测插件，让整个项目的构建运行时受限于它的兼容范围。

更重要的是，这次故障让我意识到：**网站质量检查不必依附于某个部署平台的插件生命周期。**

于是我决定在另一个使用 React、Vite 和 Bun 的前端项目中，将 Lighthouse CI 独立接入 GitHub Actions，让 Pull Request 在合并前就能得到可读的质量报告。

目标很明确：

- 只检测 `/`、`/subscriptions/`、`/upgrade-plan/` 三个公开页面。
- 每个页面执行三次 Lighthouse，预期得到九份报告。
- 四个类别低于 70 分时使 CI 失败；70–84 分显示警告；85 分及以上显示通过。
- 不向 PR 自动评论，只在 GitHub Actions Summary 展示 Markdown 表格。
- 上传 HTML/JSON 报告，方便排查具体审计项。

这里需要区分：**Netlify 的兼容性错误发生在部署项目中；后续的 Lighthouse CI 实践以 `ai-seo-frontend` 为例。** 两者是一次故障引发的工程改进思路，并不需要假设它们使用相同的构建配置。

## 一、为什么把 Lighthouse 放进 CI？

`tsc` 通过、Vite 构建成功，只能证明代码在构建层面没有阻断性问题，并不能保证页面体验达标。

Lighthouse 提供四个常用审计类别：

| 类别 | 关注内容 |
| --- | --- |
| Performance | 页面加载与渲染性能 |
| Accessibility | 无障碍访问 |
| Best Practices | Web 开发最佳实践 |
| SEO | 搜索引擎优化基础检查 |

Chrome DevTools 可以手动运行 Lighthouse，但手动检测不容易形成稳定的交付标准。Lighthouse CI（LHCI）能够在自动化环境中执行审计、保存结果并进行断言，更适合 PR 检查。

我不打算一开始就要求所有指标达到 90 分。对于现有项目，先把最低门槛设为 70 分，允许 70–84 分以警告形式存在，再根据实际报告逐步优化，是更容易持续执行的策略。**70/85 是本项目自定义的质量策略，并非 Lighthouse 官方等级划分。**

## 二、项目构建背景：Vite + React + Bun + 预渲染

项目使用 Bun 管理依赖，生产构建还包含 TypeScript 检查、客户端构建、SSR 构建和静态预渲染。

核心构建流程类似：

```bash
 tsc -b \
   && vite build \
   && vite build --ssr src/entry-server.tsx --outDir dist-ssr \
   && node scripts/prerender.mjs \
   && node scripts/verify-prerender.mjs
```

预渲染后，`dist` 中除了公开页面，还存在辅助 HTML：

```text
dist/
├── index.html
├── subscriptions/
│   └── index.html
├── upgrade-plan/
│   └── index.html
├── offline.html
├── spa.html
└── assets/
```

这五个 HTML 文件与后面的第一个大坑直接相关。

## 三、安装并运行 Lighthouse CI

安装 CLI：

```bash
bun add -d @lhci/cli
```

在 `package.json` 中添加：

```json
{
  "scripts": {
    "lhci": "lhci autorun"
  }
}
```

### 坑一：`lhci` 不是完整的运行命令

最初我把脚本写成 `"lhci": "lhci"`，执行 `bun run lhci` 后只得到 CLI 帮助和错误：

```text
Not enough non-option arguments
```

原因是 `lhci` 还需要子命令。这里使用 `lhci autorun`，让 LHCI 根据配置依次采集、断言并保存结果。

## 四、先用 staticDistDir，为什么跑出了 15 份报告？

项目已经生成静态产物，因此最初使用最简单的方式：

```javascript
// lighthouserc.cjs（初始方案节选）
module.exports = {
  ci: {
    collect: {
      staticDistDir: "./dist",
      numberOfRuns: 3,
      settings: {
        chromeFlags: "--no-sandbox",
      },
    },
    upload: {
      target: "filesystem",
      outputDir: "./lhci-reports",
    },
  },
};
```

`staticDistDir` 的优点是省心：LHCI 为静态目录启动服务，无须额外维护预览服务。

但运行结束时出现了：

```text
All results processed!
Dumping 15 reports to disk at .../lhci-reports...
Done writing reports to disk.
Done running autorun.
```

原本预期是 `3 个页面 × 3 次 = 9 份报告`，实际却是 `5 个 HTML × 3 次 = 15 份报告`。

查看结果才发现，`staticDistDir` 自动发现了 `index.html`、`subscriptions/index.html`、`upgrade-plan/index.html`，也包含 `offline.html` 和 `spa.html`。

这不是 LHCI 出错，而是**自动发现全部 HTML 的行为与我只检测三个公开页面的需求不一致**。

### 还可以继续用 staticDistDir 吗？

当然可以。它非常适合希望检测整个静态站点的场景。

但本项目更强调明确、固定的检测范围。为了避免依赖自动发现行为，最终的配置方案选择启动 Vite Preview，再显式传入三个 URL。

需要说明：下方 Preview 方案是为九次检测设计的**目标配置**；已经观察到的 15 份报告来自前面的 `staticDistDir` 方案。本文不把尚未展示的 GitHub Actions 运行结果写成已验证成功。

## 五、明确检测 URL 和质量门槛

根目录创建 `lighthouserc.cjs`：

```javascript
module.exports = {
  ci: {
    collect: {
      startServerCommand:
        "bunx vite preview --host 127.0.0.1 --port 4173 --strictPort",
      startServerReadyPattern: "Local:",
      url: [
        "http://127.0.0.1:4173/",
        "http://127.0.0.1:4173/subscriptions/",
        "http://127.0.0.1:4173/upgrade-plan/",
      ],
      numberOfRuns: 3,
      settings: {
        chromeFlags: "--no-sandbox",
      },
    },
    assert: {
      assertions: {
        "categories:performance": ["error", { minScore: 0.7 }],
        "categories:accessibility": ["error", { minScore: 0.7 }],
        "categories:best-practices": ["error", { minScore: 0.7 }],
        "categories:seo": ["error", { minScore: 0.7 }],
      },
    },
    upload: {
      target: "filesystem",
      outputDir: "./lhci-reports",
    },
  },
};
```

这里用 `--strictPort` 保证 Vite 不会在 `4173` 被占用时悄悄切换端口；`url` 则让 LHCI 只审计三个目标路径。

注意，`vite preview` 检测的是本地构建产物，不等同于真实生产环境的 CDN、缓存、网络条件与第三方服务表现。它适合作为合并前的质量门禁，但不能代替线上监控。

### 坑二：断言规则写成嵌套数组

最初尝试把失败阈值和警告阈值同时写入同一条断言：

```javascript
// 错误示例
"categories:performance": [
  ["error", { minScore: 0.7 }],
  ["warn", { minScore: 0.85 }],
]
```

结果出现：

```text
TypeError: levelA.localeCompare is not a function
```

LHCI 这里期待的断言形式是一个级别字符串加参数对象：

```javascript
"categories:performance": ["error", { minScore: 0.7 }]
```

因此我把职责拆开：**LHCI 只负责 70 分以下失败；70–84 分的黄色警告由自定义 Summary 呈现。**

| 分数 | 展示状态 | CI 结果 |
| --- | --- | --- |
| 85–100 | 🟢 Pass | 通过 |
| 70–84 | 🟡 Warning | 通过 |
| 0–69 | 🔴 Fail | 失败 |

需要注意，LHCI 的断言选择与自定义脚本逐类别计算的中位数不一定完全相同；Summary 是方便阅读的汇总，不应被误认为断言引擎内部算法的逐项复刻。

## 六、编写 GitHub Actions Summary 脚本

Lighthouse 的 HTML 报告很详细，但每次下载 Artifacts 才能看到总分并不方便。

GitHub Actions 支持通过环境变量 `GITHUB_STEP_SUMMARY` 指向的文件输出 Markdown。因此我写了一个独立脚本：

```text
scripts/lighthouse-summary.mjs
```

它负责读取本轮 JSON 报告、按 URL 分组、计算中位数、格式化红黄绿状态并输出 Markdown。

### 坑三：报告明明存在，却提示找不到 JSON

第一次运行：

```bash
bun scripts/lighthouse-summary.mjs
```

出现：

```text
Error: No Lighthouse JSON reports found
```

但 `.lighthouseci/` 目录里确实有报告。检查后发现两个目录的命名规则不同：

```text
.lighthouseci/
├── lhr-1792824862717.json
├── lhr-1792824876271.json
└── ...

lhci-reports/
├── localhost-index_html-....report.json
└── ...
```

脚本读取的是 `.lighthouseci/`，却错误地只匹配 `.report.json` 后缀。修改为匹配 `lhr-*.json` 后解决。

### 坑四：页面显示为 /index.html

静态构建产物的 URL 可能显示为：

```text
/index.html
/subscriptions/index.html
/upgrade-plan/index.html
```

但用户真正访问的是：

```text
/
/subscriptions/
/upgrade-plan/
```

因此 Summary 需要规范化路径，避免把文件名当成公开路由。

### 完整的 Summary 脚本

```javascript
import { readdir, readFile, appendFile } from "node:fs/promises";
import path from "node:path";

const reportDir = ".lighthouseci";
const pages = ["/", "/subscriptions/", "/upgrade-plan/"];
const categories = [
  "performance",
  "accessibility",
  "best-practices",
  "seo",
];

function normalizePathname(url) {
  const pathname = new URL(url).pathname;
  return pathname.replace(/\/index\.html$/, "/").replace(/\/+$/, "/");
}

function median(values) {
  const sorted = [...values].sort((a, b) => a - b);
  const mid = Math.floor(sorted.length / 2);
  return sorted.length % 2
    ? sorted[mid]
    : (sorted[mid - 1] + sorted[mid]) / 2;
}

function formatScore(score) {
  if (score == null) return "N/A";
  const value = Math.round(score * 100);
  return `${value >= 85 ? "🟢" : value >= 70 ? "🟡" : "🔴"} ${value}`;
}

async function main() {
  const filenames = (await readdir(reportDir)).filter((name) =>
    /^lhr-\d+\.json$/.test(name),
  );

  if (filenames.length === 0) {
    throw new Error("No Lighthouse JSON reports found");
  }

  const results = new Map(
    pages.map((page) => [page, Object.fromEntries(categories.map((c) => [c, []]))]),
  );

  for (const filename of filenames) {
    const report = JSON.parse(
      await readFile(path.join(reportDir, filename), "utf8"),
    );
    const url = report.requestedUrl || report.finalUrl;
    if (!url) continue;

    const pathname = normalizePathname(url);
    const scores = results.get(pathname);
    if (!scores) continue;

    for (const category of categories) {
      const score = report.categories?.[category]?.score;
      if (typeof score === "number") scores[category].push(score);
    }
  }

  const lines = [
    "## Lighthouse CI Report",
    "",
    "| Page | Performance | Accessibility | Best Practices | SEO |",
    "|---|---:|---:|---:|---:|",
  ];

  for (const [pathname, scores] of results) {
    const values = categories.map((category) => {
      const runs = scores[category];
      return runs.length ? formatScore(median(runs)) : "N/A";
    });
    lines.push(`| ${pathname} | ${values.join(" | ")} |`);
  }

  lines.push(
    "",
    "**Quality thresholds**",
    "",
    "- 🟢 85–100: Pass",
    "- 🟡 70–84: Warning",
    "- 🔴 Below 70: Fail",
    "",
    "_Scores are medians of Lighthouse runs._",
  );

  const markdown = `${lines.join("\n")}\n`;
  console.log(markdown);

  if (process.env.GITHUB_STEP_SUMMARY) {
    await appendFile(process.env.GITHUB_STEP_SUMMARY, markdown);
  }
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

脚本只负责显示指定页面。如果旧报告还留在 `.lighthouseci/`，统计会混入历史数据，因此 CI 执行前必须清理目录。

## 七、完整 GitHub Actions 工作流

在 `.github/workflows/lighthouse.yml` 中配置：

```yaml
name: Lighthouse CI

on:
  pull_request:
    branches:
      - dev
      - main
    paths:
      - "src/**"
      - "public/**"
      - "scripts/**"
      - "index.html"
      - "package.json"
      - "bun.lock"
      - "vite.config.*"
      - "tsconfig*.json"
      - "lighthouserc.*"
      - ".github/workflows/lighthouse.yml"
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: lighthouse-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lighthouse:
    name: Lighthouse Audit
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7

      - name: Setup Bun
        uses: oven-sh/setup-bun@v2
        with:
          bun-version: "1.4.2"

      - name: Setup Node.js
        uses: actions/setup-node@v7
        with:
          node-version: "24"

      - name: Install dependencies
        run: bun ci

      - name: Build application
        run: bun run build

      - name: Clean Lighthouse reports
        run: rm -rf .lighthouseci lhci-reports

      - name: Run Lighthouse CI
        run: bun run lhci

      - name: Generate Lighthouse Summary
        if: always()
        shell: bash
        run: |
          if compgen -G ".lighthouseci/lhr-*.json" > /dev/null; then
            bun scripts/lighthouse-summary.mjs
          else
            echo "## Lighthouse CI Report" >> "$GITHUB_STEP_SUMMARY"
            echo "" >> "$GITHUB_STEP_SUMMARY"
            echo "No Lighthouse JSON reports were generated." >> "$GITHUB_STEP_SUMMARY"
          fi

      - name: Upload Lighthouse reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: lighthouse-reports
          path: lhci-reports/
          if-no-files-found: ignore
          retention-days: 14
```

这里有几个刻意做出的取舍：

1. **`paths` 过滤**：只在可能影响页面或 Lighthouse 配置的文件变更时执行，避免每次修改文档都启动浏览器。
2. **`concurrency`**：同一个 PR 连续推送新提交时，取消旧的运行，节省 CI 时间。
3. **`if: always()`**：即使 Lighthouse 断言失败，也尽量保留 Summary 和报告，方便定位原因。
4. **最小权限**：只需要 `contents: read`，不自动向 PR 发表评论。
5. **运行前清理**：避免旧的 Lighthouse JSON 污染当前检测的中位数。

有一点需要额外注意：如果 Lighthouse 工作流被设置为分支保护中的 *required check*，`paths` 过滤可能导致某些 PR 根本不触发该工作流，从而使必需检查一直处于等待状态。是否保留路径过滤，应与仓库的分支保护策略一起决定。

## 八、实际结果：哪些指标值得继续优化？

在最初的 `staticDistDir` 本地检测中，三个公开页面的 Summary 结果如下：

| Page | Performance | Accessibility | Best Practices | SEO |
| --- | ---: | ---: | ---: | ---: |
| `/` | 🟡 84 | 🟢 92 | 🟡 79 | 🟢 100 |
| `/subscriptions/` | 🟢 86 | 🟢 92 | 🟡 79 | 🟢 100 |
| `/upgrade-plan/` | 🟢 86 | 🟢 92 | 🟡 79 | 🟢 100 |

这份表格的路径已做规范化，**分数来自之前的本地 15 份报告，而不是后续九次检测方案的重新测量**。

三个主要页面的 Best Practices 都是 79 分，值得优先查看 HTML 报告中的具体审计失败项。相同分数可能意味着共享资源或公共脚本存在共同问题，但仅凭分数还不能下结论。

Performance 方面也不应只盯着总分。下一步更有意义的是结合 LCP、CLS、TBT 等具体指标，确定优化顺序。

## 九、总结：把质量检查从部署平台解耦

这次实践起点并不是“我要接入 Lighthouse CI”，而是一次真实的 Netlify 部署失败：`@netlify/plugin-lighthouse@4` 无法在 Node.js 22.23.3 环境中运行。

它让我重新思考了两个问题：

**第一，部署工具和质量检查工具是否一定要绑定在一起？**

不一定。把 Lighthouse CI 放在 GitHub Actions 中，能够独立控制 Node.js、Bun、LHCI 版本和质量规则，降低对部署平台插件的依赖。

**第二，CI/CD 的绿色状态到底意味着什么？**

构建成功很重要，但并不等于页面体验合格。通过独立的 Lighthouse 审计，可以为关键页面增加一层自动化质量门禁。

整个过程还遇到了几个真实的小坑：

- `lhci` 少了 `autorun` 子命令。
- LHCI 断言规则被错误写成嵌套数组，触发 `levelA.localeCompare` 异常。
- `staticDistDir` 自动发现五个 HTML，生成了 15 份报告，而不是预期的九份。
- Summary 脚本误判 JSON 文件名，提示 `No Lighthouse JSON reports found`。
- 预渲染产物的 `/index.html` 路径需要规范化为公开路由。

这些问题并不复杂，但它们体现了工程实践中比“工具能跑起来”更重要的部分：**检测范围是否准确、失败策略是否合理、报告是否容易理解，以及长期维护成本是否可控。**

下一步，我会在 GitHub Actions 中验证九次检测方案的实际运行结果，并根据报告逐步优化 Best Practices 和 Performance。等质量门禁稳定后，再考虑提高阈值、加入历史趋势分析，以及结合生产环境的真实用户性能数据。

对我来说，这比一开始就追求 Lighthouse 全绿更有价值。
