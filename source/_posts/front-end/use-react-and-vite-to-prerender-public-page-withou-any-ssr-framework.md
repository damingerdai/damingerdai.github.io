---
title: 不使用 SSR 框架，如何给 React + Vite 实现 Prerender
date: 2026-10-05 23:46:18
tags: [react, react-dom, vite, react-router, prerender]
categories: [前端]
---

# 不使用 SSR 框架，如何给 React + Vite 实现 Prerender

最近在优化一个 React + Vite 项目的 SEO。

项目本身使用 React Router Data Mode，是一个标准的 SPA。公开页面主要有：

```text
/
/subscriptions
/upgrade-plan
```

而 `/dashboard` 等页面属于登录后的应用页面，并不需要搜索引擎抓取。

最开始考虑过迁移到 React Router Framework Mode，通过 SSR 或 SSG 解决这个问题。但对于一个已经存在的 Vite SPA 来说，为了三个公开页面迁移整个应用架构，成本有些过高。

最终采用了一个更轻量的方案：

> 保持 React + Vite SPA 架构不变，在构建阶段额外运行一次 React Static Prerender，只为指定公开页面生成静态 HTML。

最终架构可以概括为：

```text
React + Vite SPA
       +
Build-time SSR Renderer
       +
React prerender()
       ↓
Selective SSG
```

也就是说：

- 公开页面：Prerender + Hydration
- 私有页面：传统 CSR
- 不需要运行 Node SSR Server
- 不需要迁移到 Next.js、React Router Framework Mode 等框架

## 1. 为什么普通 Vite SPA 对 SEO 不够理想

普通 Vite SPA 构建完成后的 `index.html` 基本是：

```html
<div id="root"></div>
<script type="module" src="/assets/index-xxx.js"></script>
```

访问：

```text
/subscriptions
```

实际过程是：

```text
HTML
 ↓
空 #root
 ↓
下载 JavaScript
 ↓
React 启动
 ↓
React Router 匹配 /subscriptions
 ↓
渲染页面
```

也就是说初始 HTML 本身没有页面正文。

我们希望公开页面变成：

```html
<div id="root">
  <main>
    <h1>...</h1>

    ...
  </main>
</div>
```

即使关闭 JavaScript，也可以直接看到公开页面的主体内容。

但同时又不希望 `/dashboard` 等页面也变成 SSR。

所以这里真正需要的是 **Selective Prerender**。

---

# 2. 整体构建流程

最终的 build command 是：

```json
{
  "scripts": {
    "build": "tsc -b && vite build && vite build --ssr src/entry-server.tsx --outDir dist-ssr && node scripts/prerender.mjs && node scripts/verify-prerender.mjs"
  }
}
```

乍一看比较长，其实可以拆成四步：

```text
tsc -b
   ↓
TypeScript check

vite build
   ↓
Client Build
   ↓
dist/

vite build --ssr src/entry-server.tsx
   ↓
Build-time Renderer
   ↓
dist-ssr/

scripts/prerender.mjs
   ↓
调用 Renderer
   ↓
生成静态 HTML

scripts/verify-prerender.mjs
   ↓
验证生成结果
```

其中最容易产生误解的是：

```bash
vite build --ssr
```

这里虽然出现了 `SSR`，但生产环境实际上并没有 SSR Server。

---

# 3. dist 和 dist-ssr 到底有什么区别？

普通：

```bash
vite build
```

生成：

```text
dist/
├── index.html
└── assets/
```

这些文件最终会部署给浏览器。

而：

```bash
vite build --ssr src/entry-server.tsx --outDir dist-ssr
```

生成的是：

```text
dist-ssr/
└── entry-server.js
```

它不是给浏览器使用的。

它的作用是：

> 把 React 应用编译成 Node.js 在构建阶段可以执行的 Renderer。

因为 Node 不能直接运行：

```text
entry-server.tsx
App.tsx
routes.tsx
JSX / TSX
Vite alias
...
```

所以先通过 Vite SSR Build：

```text
src/entry-server.tsx
        ↓
   Vite SSR Build
        ↓
dist-ssr/entry-server.js
```

然后 `prerender.mjs` 才可以：

```js
import { render } from "../dist-ssr/entry-server.js";
```

所以：

```text
dist/
```

是**产品**。

而：

```text
dist-ssr/
```

更像是**生产这个产品的工具**。

整个 build 完成以后，`dist-ssr` 甚至可以删除。

---

# 4. Client 和 Server 必须共享 Routes

原来的 Router 定义在 `Router.tsx`。

为了让浏览器和构建阶段使用完全相同的 Route Tree，将 route configuration 单独拆成：

```text
routes.tsx
```

然后导出：

```ts
export const routes: RouteObject[] = [
  // ...
];
```

浏览器使用：

```ts
createBrowserRouter(routes);
```

Prerender 使用：

```ts
createStaticHandler(routes);
```

形成：

```text
                routes.tsx
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
createBrowserRouter    createStaticHandler
          ↓                   ↓
       Browser              Node
```

这点非常重要。

否则 Server Render 出来的 React Tree 和 Browser Hydration 的 React Tree 不一致，很容易产生 hydration mismatch。

---

# 5. entry-server.tsx：在 Node 中模拟一次路由访问

Server Entry 的核心逻辑类似：

```ts
const handler = createStaticHandler(routes);

export async function render(path: string) {
  const context = await handler.query(
    new Request(`https://prerender.invalid${path}`),
  );

  const router = createStaticRouter(handler.dataRoutes, context);

  // ...
}
```

比如：

```ts
render("/upgrade-plan");
```

React Router 会在 Node 环境中模拟：

```text
Request /upgrade-plan
        ↓
React Router route matching
        ↓
匹配 /upgrade-plan
        ↓
执行 loader
        ↓
createStaticRouter()
```

例如这个 route 有：

```ts
{
  path: 'upgrade-plan',
  loader: planCatalogLoader,
  element: ...
}
```

那么 prerender 时 `planCatalogLoader` 也会执行。

---

# 6. StaticRouterProvider 是做什么的？

Server Render 使用：

```tsx
<StaticRouterProvider router={router} context={context} />
```

它除了渲染页面之外，还有一个非常重要的作用：

**把 React Router 的 hydration data 输出到 HTML。**

最终 HTML 中会出现类似：

```html
<script>
  window.__staticRouterHydrationData = ...
</script>
```

其中包含 React Router hydration 所需要的 route state，例如 loader data。

所以可以粗略理解：

```text
Server

loader()
   ↓
StaticRouterProvider
   ↓
__staticRouterHydrationData
   ↓
HTML
   ↓
Browser
   ↓
createBrowserRouter()
   ↓
恢复 Router state
```

这也是为什么 prerender 脚本会检查：

```js
markup.includes("__staticRouterHydrationData");
```

它是在确认：

> React Router 的 hydration 数据确实被生成了。

---

# 7. 使用 React prerender()

构造完整 React Tree：

```tsx
const app = (
  <StrictMode>
    <QueryProvider>
      <App>
        <StaticRouterProvider router={router} context={context} />
      </App>
    </QueryProvider>
  </StrictMode>
);
```

然后使用 React Static API：

```ts
const { prelude } = await prerender(app, {
  onError(error) {
    renderError = error;
  },
  signal: AbortSignal.timeout(30_000),
});
```

`prerender()` 和传统 `renderToString()` 有一个很重要的区别：

它是为 Static Generation 设计的，可以等待 Suspense 内容完成。

这对于项目里大量存在：

```ts
React.lazy(() => import(...))
```

以及：

```tsx
<Suspense fallback={...}>
```

非常重要。

整体过程变成：

```text
React Tree
   ↓
lazy component
   ↓
Suspense
   ↓
prerender()
   ↓
等待内容 ready
   ↓
Static HTML
```

## 关于“两遍 Render”

最初的实现为了确保 lazy component 和 Suspense 都已经完成，采用了：

```ts
const { prelude } = await prerender(app);

await new Response(prelude).text();

return renderToString(app);
```

也就是：

```text
prerender()
   ↓
等待 Suspense
   ↓
丢弃第一遍 HTML
   ↓
renderToString()
   ↓
生成最终 HTML
```

这个方案可以工作，但进一步分析之后，我认为没有必要。

`prerender()` 本身就是 React 提供的 Static Prerender API，因此更自然的写法是直接消费并使用它的输出：

```ts
const { prelude } = await prerender(app, {
  onError(error) {
    renderError = error;
  },
  signal: AbortSignal.timeout(30_000),
});

const html = await new Response(prelude).text();

if (renderError) throw renderError;

return html;
```

避免重复渲染整棵 React Tree。

也避免组件中存在时间、随机值或其他非 deterministic 内容时，两次 render 得到不同结果。

---

# 8. prerender.mjs：真正生成静态 HTML

有了：

```ts
render("/subscriptions");
```

以后，还没有真正生成一个 HTML 文件。

这一步由：

```text
scripts/prerender.mjs
```

负责。

首先读取 Vite 原始模板：

```js
const template = await readFile(resolve("dist", "index.html"), "utf8");
```

里面原本是：

```html
<div id="root"></div>
```

然后：

```js
const markup = await render(path);
```

拿到 React 输出：

```html
<main>...</main>

<script>
  window.__staticRouterHydrationData = ...
</script>
```

再替换：

```js
template.replace(
  '<div id="root"></div>',
  () => `<div id="root" data-prerendered="${path}">${markup}</div>`,
);
```

最终 `/subscriptions` 变成：

```html
<div id="root" data-prerendered="/subscriptions">
  <main>...</main>

  <script>
    window.__staticRouterHydrationData = ...
  </script>
</div>
```

然后写入：

```text
dist/subscriptions/index.html
```

---

# 9. 为什么需要 data-prerendered？

这里专门增加了：

```html
data-prerendered="/subscriptions"
```

它不是 React 或 React Router 的 API。

这是我们自己的一个 marker。

作用是告诉 Browser：

> 当前 `#root` 里面的 HTML 是专门为 `/subscriptions` prerender 出来的。

浏览器启动时：

```ts
const path = window.location.pathname.replace(/\/+$/, "") || "/";

if (root.dataset.prerendered === path) {
  hydrateRoot(root, app);
} else {
  createRoot(root).render(app);
}
```

例如：

```text
URL
/subscriptions

HTML marker
data-prerendered="/subscriptions"
```

匹配：

```ts
"/subscriptions" === "/subscriptions";
```

于是：

```ts
hydrateRoot(root, app);
```

React 不需要重新创建整个 DOM，而是直接 hydrate 已经存在的静态 HTML。

---

# 10. 非 Prerender 页面继续使用 createRoot

我们并不希望所有页面都 prerender。

例如：

```text
/dashboard
```

依然应该是传统 SPA。

所以构建时先保存原始 Vite shell：

```js
await writeFile(resolve(out, "spa.html"), template);
```

其中仍然是：

```html
<div id="root"></div>
```

访问 `/dashboard` 时：

```ts
root.dataset.prerendered;
```

是：

```ts
undefined;
```

因此：

```ts
createRoot(root).render(app);
```

最终形成：

```text
Public page
/subscriptions
      ↓
Static HTML
      ↓
hydrateRoot()


Private / CSR page
/dashboard
      ↓
spa.html
      ↓
empty #root
      ↓
createRoot()
```

这就是整个 Selective Prerender 的核心。

---

# 11. 为什么 Router.tsx 还要删除 Hydration Data？

Router 初始化还有一个容易忽略的细节：

```ts
const root = document.getElementById("root");

const path = window.location.pathname.replace(/\/+$/, "") || "/";

if (root?.dataset.prerendered !== path) {
  Reflect.deleteProperty(window, "__staticRouterHydrationData");
}

const router = createBrowserRouter(routes);
```

为什么？

因为：

```ts
createBrowserRouter(routes);
```

会读取 React Router 已经存在的 hydration data。

只有：

```text
data-prerendered
        ===
current pathname
```

才能证明这份 hydration data 属于当前页面。

如果不匹配，就应该删除：

```ts
window.__staticRouterHydrationData;
```

让 React Router 正常按照 CSR 模式启动。

注意顺序也很重要：

```ts
delete hydration data

        ↓

createBrowserRouter()
```

不能反过来。

因为 Router 创建时就可能已经读取 hydration data。

---

# 12. React Hydration 和 Router Hydration 是两回事

这是整个实现中一个很容易混淆的地方。

`main.tsx`：

```ts
hydrateRoot(root, app);
```

处理的是：

> React DOM 是否复用已有 HTML。

而 Router 中：

```ts
window.__staticRouterHydrationData;
```

处理的是：

> React Router 是否复用构建阶段产生的 route / loader state。

所以实际上有两个 hydration：

```text
data-prerendered === current path
              │
       ┌──────┴──────┐
       ↓             ↓
 React DOM       React Router
       ↓             ↓
hydrateRoot()    保留 hydration data
```

如果不是 prerender 页面：

```text
data-prerendered !== current path
              │
       ┌──────┴──────┐
       ↓             ↓
 React DOM       React Router
       ↓             ↓
createRoot()     删除 hydration data
```

两者职责不同，但判断依据相同。

---

# 13. 不要用 `<h1>` 判断 Prerender 是否成功

最初的 prerender script 中还有：

```js
if (
  !markup.includes("<h1") ||
  !markup.includes("__staticRouterHydrationData")
) {
  throw new Error(`Missing content or hydration data: ${path}`);
}
```

这里：

```js
__staticRouterHydrationData;
```

是合理的 hydration contract。

但是：

```js
markup.includes("<h1");
```

只是一个比较脆弱的 heuristic。

它实际上是在说：

> 如果 HTML 里存在 `<h1>`，就认为页面成功 render 了。

但 `<h1>` 属于页面语义和 SEO 规则，并不是 prerender 的技术要求。

更合理的是：

```js
if (!markup.trim() || !markup.includes("__staticRouterHydrationData")) {
  throw new Error(`Missing content or hydration data: ${path}`);
}
```

分别验证：

```text
markup.trim()
      ↓
React 确实输出了内容


__staticRouterHydrationData
      ↓
React Router hydration data 存在
```

然后最终 HTML 再通过：

```html
data-prerendered="/subscriptions"
```

确认它属于哪个 path。

至于：

```html
<h1></h1>
```

可以继续放在 `verify-prerender.mjs` 中作为 SEO / semantic verification：

```js
assert.equal(doc.querySelectorAll("h1").length, 1);
```

这两个层次最好分开：

```text
Prerender correctness
        ↓
有内容
有 hydration data
有 data-prerendered


SEO correctness
        ↓
一个 h1
足够正文内容
价格存在
...
```

---

# 14. 为什么还需要 verify-prerender.mjs？

只生成文件还不够。

构建完成后会自动执行：

```bash
node scripts/verify-prerender.mjs
```

验证包括：

- `spa.html` 的 root 仍然为空
- SPA shell 不包含 hydration data
- prerender 页面存在正确的 `data-prerendered`
- 页面只有一个 `<h1>`
- `<main>` 中存在足够的正文内容
- 存在 `__staticRouterHydrationData`
- 没有 fallback 到 `Switch to client rendering`
- CSS / JS assets 和原始 Vite shell 一致
- Vercel rewrite 指向正确 HTML
- `/subscriptions` 与 `/subscriptions/` 都能匹配
- API/BFF rewrite 没有被破坏

这一步其实很有价值。

因为 prerender 最危险的情况不是 build 失败，而是：

> Build 成功了，但是生成的是 Loading Skeleton 或空页面。

Verification 可以直接在 CI 阶段阻止这种错误部署。

---

# 15. Vercel Routing

最终静态目录类似：

```text
dist/
├── index.html
├── spa.html
├── subscriptions/
│   └── index.html
├── upgrade-plan/
│   └── index.html
└── assets/
```

Vercel 中显式配置：

```text
/                         → /index.html

/subscriptions            → /subscriptions/index.html
/subscriptions/           → /subscriptions/index.html

/upgrade-plan             → /upgrade-plan/index.html
/upgrade-plan/            → /upgrade-plan/index.html

其他页面
                           → /spa.html
```

同时：

```text
/api/*
/bff/*
```

继续保持原来的 Serverless Function / Worker rewrite。

这意味着 prerender 完全没有改变应用原来的 API 架构。

---

# 16. 最终架构

完整构建过程：

```text
                     Source Code
                         │
             ┌───────────┴───────────┐
             │                       │
        vite build            vite build --ssr
             │                       │
             ▼                       ▼
           dist/                  dist-ssr/
        Client Assets          entry-server.js
             │                       │
             │                 render(path)
             │                       │
             │                React prerender()
             │                       │
             └───────────┬───────────┘
                         ▼
                  prerender.mjs
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
             /    /subscriptions  /upgrade-plan
             │           │           │
             ▼           ▼           ▼
          Static       Static      Static
           HTML         HTML        HTML
                         │
                         ▼
               verify-prerender.mjs
                         │
                         ▼
                    Deploy dist/
```

浏览器端则是：

```text
                   Browser Request
                         │
             ┌───────────┴───────────┐
             │                       │
        Public Route              CSR Route
             │                       │
        Static HTML                spa.html
             │                       │
data-prerendered matches          empty root
             │                       │
      hydrateRoot()              createRoot()
             │                       │
             └───────────┬───────────┘
                         ▼
                      React App
```

---

# 17. 这个方案适合什么场景？

这个方案比较适合：

- 已经存在的 React + Vite SPA
- 不想迁移 SSR Framework
- 只有少量公开页面需要 SEO
- 大量后台页面不需要 SSR
- 希望继续使用现有 Vercel Static Deployment
- 希望最小化架构改造

例如：

```text
Marketing pages       → Prerender
Pricing pages         → Prerender
Public product pages  → Prerender

Dashboard             → CSR
Settings              → CSR
Admin                  → CSR
Authenticated pages   → CSR
```

如果未来需要 prerender：

```text
/products/:id
/blog/:slug
/categories/:id
```

几百甚至几千个动态页面，那么继续维护自己的 SSG Pipeline 的复杂度会逐渐增加。

到那个阶段再考虑 React Router Framework Mode、Next.js 等完整框架会更加合理。

但对于只有几个公开页面需要 SEO 的成熟 Vite SPA，为了 SSR 重构整个应用往往并不划算。

---

# 18. Prerender 不等于完整的 SEO

前面的实现解决了一个重要问题：

```text
普通 SPA

<div id="root"></div>

        ↓

Prerender

<div id="root">
  <main>
    页面正文
  </main>
</div>
```

搜索引擎和其他不执行 JavaScript 的客户端现在可以直接读取页面正文。

但这并不意味着 SEO 工作已经完成。

一个公开页面通常还需要正确输出：

```html
<title>...</title>

<meta
  name="description"
  content="..."
>

<link
  rel="canonical"
  href="..."
>

<meta
  property="og:title"
  content="..."
>

<meta
  property="og:description"
  content="..."
>

<meta
  property="og:url"
  content="..."
>

<meta
  property="og:image"
  content="..."
>
```

如果使用结构化数据，还可能包含：

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization"
}
</script>
```

因此，一个真正可用于 SEO 的 prerender 页面至少需要考虑两个部分：

```text
HTML Document
│
├── <head>
│   ├── title
│   ├── description
│   ├── canonical
│   ├── Open Graph
│   └── structured data
│
└── <body>
    └── #root
        └── 页面正文
```

## React 19 的 Metadata Hoisting

如果使用 React 19，可以直接在组件中声明：

```tsx
function SubscriptionsPage() {
  return (
    <>
      <title>Plans & Pricing | Intelligent Brand</title>

      <meta
        name="description"
        content="Choose the Intelligent Brand plan that fits your business."
      />

      <meta
        property="og:title"
        content="Plans & Pricing | Intelligent Brand"
      />

      <meta
        property="og:description"
        content="Choose the Intelligent Brand plan that fits your business."
      />

      <main>
        ...
      </main>
    </>
  );
}
```

React 会处理这些 metadata 元素，而不需要为了 `<title>` 单独操作 `document.title`。

这点对于 prerender 尤其重要。

如果 SPA 使用：

```ts
useEffect(() => {
  document.title = 'Plans & Pricing';
}, []);
```

标题只能在浏览器 JavaScript 执行之后修改。

而 prerender 的目标恰恰是：

> 在 JavaScript 执行之前，HTML Document 本身就已经包含 SEO 信息。

因此对于需要 prerender 的公开页面，SEO metadata 最好也进入 React 的静态渲染过程。

## Prerender Verification 也应该检查 `<head>`

这也意味着 `verify-prerender.mjs` 不应该只验证：

```text
✓ 页面正文存在
✓ <h1> 存在
✓ hydration data 存在
✓ data-prerendered 正确
```

还应该验证 SEO metadata。

例如：

```text
/subscriptions

✓ <title> 非空
✓ meta[name="description"] 非空
✓ canonical URL 正确
✓ og:title 存在
✓ og:description 存在
✓ og:url 正确
✓ 必要时检查 og:image
✓ 必要时检查 JSON-LD
```

最终可以把 verification 分成三个层次：

```text
1. Prerender Correctness
   ├── React 输出非空
   ├── data-prerendered 正确
   └── 没有 client-render fallback

2. Hydration Correctness
   ├── __staticRouterHydrationData 存在
   ├── React hydrateRoot 正常
   └── Router hydration 正常

3. SEO Correctness
   ├── title
   ├── description
   ├── canonical
   ├── Open Graph
   ├── structured data
   └── 页面语义，例如唯一 h1
```

这样 `<h1>` 的定位也更加清楚。

它不应该被用来证明：

```text
Prerender 成功
```

而应该被用来验证：

```text
页面的 HTML 语义 / SEO 规则符合预期
```

## 最终目标

因此，这套方案真正要达到的效果不是简单地：

```text
SPA → HTML 有内容
```

而应该是：

```text
              Prerendered HTML
                     │
          ┌──────────┴──────────┐
          │                     │
        <head>                 <body>
          │                     │
    SEO Metadata            页面正文
          │                     │
    title                   semantic HTML
    description             h1
    canonical               main
    Open Graph              links
    JSON-LD                 content
```

这样生成出来的才是一个真正适合作为公开 SEO Landing Page 的静态 HTML。

因此更准确地说：

> **Prerender 解决的是 SPA SEO 的“初始 HTML 不包含页面内容和 metadata”问题，而不是 SEO 的全部问题。**

关键词研究、内容质量、内部链接、Sitemap、robots.txt、Core Web Vitals、结构化数据质量等，仍然属于 Prerender 之外的 SEO 工作。

# 总结

整个方案最重要的并不是 `vite build --ssr` 本身，而是把不同职责拆开：

```text
Vite Client Build
→ 负责浏览器运行

Vite SSR Build
→ 负责生成构建阶段可执行的 Renderer

React prerender()
→ 负责把 React Tree 转成静态 HTML

StaticRouterProvider
→ 负责 Server Router 和 hydration data

data-prerendered
→ 标识 HTML 属于哪个 route

hydrateRoot()
→ 接管已经生成的静态 HTML

createRoot()
→ 保持其他页面传统 CSR

spa.html
→ 保留原来的 SPA fallback

verify-prerender
→ 防止错误的静态 HTML 被部署
```

最终得到的并不是一个 SSR Application，而仍然是一个静态部署的 React SPA。

只不过构建阶段额外做了一件事：

> **提前把搜索引擎需要看到的几个页面渲染好了。**

这是一种介于纯 SPA 和完整 SSR Framework 之间的方案：

**React + Vite + Build-time Prerender + Selective Hydration。**
