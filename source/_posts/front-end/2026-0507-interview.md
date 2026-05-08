---
title: 2026年5月7号面试总结
date: 2026-05-08 09:44:37
tags: [高级前端工程师, 面试]
categories: [前端]
---

# 前言

2026年5月7号有两场面试，本文记录了自己预先准备面试题和面试中遇到的问题。

# 笔试

## 防抖 (Debounce)

原理：在事件被触发 $n$ 秒后再执行回调，如果在这 $n$ 秒内又被触发，则重新计时。适用于：搜索框输入、窗口大小调整

```ts
function debounce<T extends (...args: any[]) => void>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: ReturnType<typeof setTimeout> | null = null;

  return function (this: any, ...args: Parameters<T>) {
    if (timeout) clearTimeout(timeout);

    timeout = setTimeout(() => {
      func.apply(this, args);
    }, wait);
  };
}
```

## 节流 (Throttle)

原理：规定在一个单位时间内，只能触发一次函数。如果这个单位时间内触发了多次函数，只有一次生效。适用于：滚动事件（scroll）、抢购按钮点击。

```ts
function throttle<T extends (...args: any[]) => void>(
  func: T,
  limit: number
): (...args: Parameters<T>) => void {
  let lastCall = 0;

  return function (this: any, ...args: Parameters<T>) {
    const now = Date.now();
    if (now - lastCall >= limit) {
      lastCall = now;
      func.apply(this, args);
    }
  };
}
```

## 深拷贝（Deep Copy）

### 现代最简法：structuredClone

```ts
const original = { a: 1, b: { c: 2 } };
const copy = structuredClone(original);
```

优点：原生支持、性能好、支持循环引用、支持 Date, RegExp, Map, Set 等类型。
缺点：不支持函数、不可扩展。

### 传统最简法：JSON.parse(JSON.stringify())

```ts
const original = {
  name: "Arthur",
  age: 25,
  date: new Date(),        // 特殊对象：Date
  reg: /abc/g,            // 特殊对象：RegExp
  und: undefined,         // 特殊值：undefined
  func: () => { console.log("hello") }, // 特殊值：函数
  nan: NaN,               // 特殊值：NaN
  infinity: Infinity      // 特殊值：正无穷
};

// 执行深拷贝
const copy = JSON.parse(JSON.stringify(original));

console.log("Original:", original);
console.log("Copy:", copy);

// 验证独立性
copy.name = "Damingerdai";
console.log(original.name); // 输出 "Arthur"，互不影响
```

优点：一行搞定，兼容性极强。

缺点：

丢失数据：会忽略 undefined、函数和 Symbol。

异常转换：Date 会变成字符串，RegExp 会变成空对象。

循环引用：如果对象内部自己引用自己，会直接报错

### 手写递归实现（面试高频）

```ts
function deepClone<T>(obj: T, hash = new WeakMap()): T {
  // 1. 处理 null 或基本类型
  if (obj === null || typeof obj !== "object") return obj;

  // 2. 处理日期和正则
  if (obj instanceof Date) return new Date(obj) as any;
  if (obj instanceof RegExp) return new RegExp(obj) as any;

  // 3. 处理循环引用：如果已经拷贝过，直接返回之前的结果
  if (hash.has(obj as object)) return hash.get(obj as object);

  // 4. 初始化克隆对象（保留数组或对象的形状）
  const result: any = Array.isArray(obj) ? [] : {};
  hash.set(obj as object, result);

  // 5. 递归拷贝所有属性
  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      result[key] = deepClone(obj[key], hash);
    }
  }

  return result;
}
```

## 二叉搜索树

### 定义节点类

```ts
class TreeNode {
    val: number;
    left: TreeNode | null;
    right: TreeNode | null;

    constructor(val: number, left: TreeNode | null = null, right: TreeNode | null = null) {
        this.val = val;
        this.left = left;
        this.right = right;
    }
}
```

### 插入节点 (Insert)

```ts
function insertIntoBST(root: TreeNode | null, val: number): TreeNode {
    // 如果当前节点为空，说明找到了插入位置，创建新节点返回
    if (!root) {
        return new TreeNode(val);
    }

    if (val < root.val) {
        // 递归向左子树插入
        root.left = insertIntoBST(root.left, val);
    } else {
        // 递归向右子树插入
        root.right = insertIntoBST(root.right, val);
    }

    return root;
}
```

### 删除节点 (Delete)

```ts
function deleteNode(root: TreeNode | null, key: number): TreeNode | null {
    if (!root) return null;

    if (key < root.val) {
        // 去左子树删除
        root.left = deleteNode(root.left, key);
    } else if (key > root.val) {
        // 去右子树删除
        root.right = deleteNode(root.right, key);
    } else {
        // 命中了要删除的节点

        // 情况 1: 叶子节点或只有一个子节点
        if (!root.left) return root.right;
        if (!root.right) return root.left;

        // 情况 2: 有两个子节点
        // 找到右子树的最小节点（后继节点）
        const minNode = getMin(root.right);
        // 用后继节点的值覆盖当前节点
        root.val = minNode.val;
        // 在右子树中删除那个重复的后继节点
        root.right = deleteNode(root.right, minNode.val);
    }

    return root;
}

// 辅助函数：获取子树中的最小节点
function getMin(node: TreeNode): TreeNode {
    let curr = node;
    while (curr.left) {
        curr = curr.left;
    }
    return curr;
}
```


# JavaScript

## 闭包

闭包是指一个函数能够记住并访问它的“词法作用域”（Lexical Scope），即使这个函数在它被定义的作用域之外执行。
闭包是 JavaScript 静态作用域规则的自然产物。它的本质是函数保存了对创建时作用域的引用

应用场景：

1. 数据私有化（模拟私有变量）；
2. 柯里化（Currying）；
3. 回调函数与异步处理。

代价：
1. 可能会导致内存泄露
2. 存在一定的性能开销

## 原型链（Prototype Chain）

在 JavaScript 中，原型链（Prototype Chain） 是实现对象之间 属性共享 和 继承 的核心机制

```ts
function Animal(name) {
  this.name = name;
}
Animal.prototype.eat = function() {
  console.log(this.name + " is eating.");
};

function Dog(name) {
  Animal.call(this, name); // 借用构造函数继承属性
}

// 核心：原型链继承方法
Dog.prototype = Object.create(Animal.prototype); 
Dog.prototype.constructor = Dog;

const myDog = new Dog("Buddy");
myDog.eat(); // 沿着原型链找到了 Animal.prototype 上的方法
```

# React/Nextjs

## 关于 Server Actions 的错误处理

在企业内网环境下，稳定性和用户反馈的准确性是第一位的。

### 统一响应结构

不要让前端直接处理 throw new Error，而是通过 return 返回一个标准对象。这样前端可以使用 useActionState 轻松获取错误信息，并保持 UI 类型安全

### 业务级脱敏与捕获

在 Action 内部使用 try-catch 包裹数据库（如 Drizzle）操作。

- 对外： 将原始错误（如 Foreign key constraint failed）转换为用户友好的文案（如“关联数据校验失败”）。
- 对内： 同时将原始堆栈信息记录到企业内部监控日志，方便排查由于内网环境特有的网络或权限导致的故障。

### 全局防御 (error.js)

在文件夹中配置 error.js 文件，作为 Server Action 以外的最后防线。

- 作用： 捕捉那些没被 try-catch 住的致命错误（如数据库连接断开、服务器 OOM）。
- 体验： 确保用户看到的是一个带有“重试”按钮的报错页面，而不是浏览器默认的白屏或 JSON 源码。

## useTransition 和 useDeferredValue 

问题：React 18/19 中，useTransition 和 useDeferredValue 是如何利用并发特性（Concurrent Mode）来优化长列表或复杂计算的

回答如下：

在 React 18/19 中，useTransition 和 useDeferredValue 的核心逻辑都是基于 Fiber 架构的可中断渲染（Interruptible Rendering）。它们不再像以前那样“一把梭”更新所有组件，而是将更新分成了高优先级和低优先级。
以前 React 的渲染是阻塞式的（Blocking），一旦开始渲染 10000 条数据，浏览器就没法响应用户输入了。引入并发特性后，useTransition 允许我们将渲染任务标记为 Non-urgent（非紧急）。

React 此时会利用 Fiber 架构进行时间切片：它每计算一小段列表，就停下来看看有没有用户输入。如果有，就先去跑输入的逻辑。这种‘可中断’的特性，让长列表的计算不再导致页面假死。而 useDeferredValue 则是这种机制在数据流层面的应用，它确保了高频变动的值不会立刻触发重型计算，从而保障了主线程的绝对优先权。

## React Fiber保证一致性

首先，在架构层面，Fiber 采用了双缓存（Double Buffering）。所有的并发渲染都在内存中的 workInProgress 树上进行，只有当整棵树协调完毕，才会一次性挂载到真实的 DOM 上。这就保证了用户看到的 UI 永远是完整的。

其次，针对中断期间的状态变化，React 利用了 Lanes 优先级模型。如果一个正在进行的低优先级渲染因为新的状态更新被中断，React 并不会强行接着渲染，而是会根据情况丢弃当前的中间结果。它会重新获取最新的状态快照，从头（或从受影响的节点）开始一轮新的渲染。

简而言之，Fiber 允许渲染过程‘断断续续’，但它通过**‘丢弃重做’和‘原子提交’**，确保了渲染结果要么不出来，出来就一定是基于当前时刻最新状态的完整镜像。

## useLayoutEffect 和 useEffect

问题：useLayoutEffect 和 useEffect有什么区别

回答如下：

useEffect 是在屏幕更新后异步执行的，它是非阻塞的，适合处理绝大多数业务逻辑，如 API 请求。

而 useLayoutEffect 是在 DOM 更新后、浏览器绘图前同步执行的。在我之前的项目中，比如实现一个自动根据内容调整位置的 Tooltip 浮层时，我会选择 useLayoutEffect。因为我需要在渲染的第一时间获取 Tooltip 的物理尺寸，如果不加干预直接渲染，用户会看到浮层先在默认位置闪现，然后才跳到正确位置；而 useLayoutEffect 能确保浏览器在显示第一帧时，浮层就已经在计算好的位置了。

## setState的同步异步问题

问题：React的setstate是同步的还是异步的

回答如下：

setState 本身的执行过程是同步的，但它触发的状态更新行为在 React 的不同版本和场景下表现不同。
在 React 18 之后，为了性能优化，React 引入了自动批处理（Automatic Batching），这使得 setState 在绝大多数场景下看起来都是“异步”的。

setState 的同步或异步表现，本质上是 React ‘调度（Scheduling）’ 的结果。在 React 18 之后，由于引了 ‘自动批处理（Automatic Batching）’，setState 在所有场景下（包括 Promise 和 setTimeout）都表现为异步。这主要是为了减少不必要的重绘，提升渲染性能。如果我们需要基于当前状态进行多次更新，我会使用函数式更新（updater function）。如果需要在状态更新并渲染完成后执行逻辑，我会将其放在 useEffect 中。值得注意的是，React 18 的 并发模式（Concurrent Mode） 进一步强化了这种‘异步感’，因为它允许 React 根据任务优先级（Lanes 模型）来中断和恢复渲染，从而保证高优先级任务（如用户输入）的即时响应。”

- React 为了避免频繁的重新渲染（Re-render），会将同一个事件处理函数中的多个 setState 合并为一个更新任务

## React Hooks

### React Hooks的作用

- 在 Hooks 出现之前，React 复用逻辑主要靠 HOC（高阶组件） 和 Render Props。Hooks 允许你在不改变组件结构的情况下，将逻辑提取到可重用的函数（Custom Hooks）中。逻辑变得像插拔式插件一样简单
- 在类组件中，同一个业务逻辑的代码往往被拆分在不同的生命周期里，useEffect 允许你根据业务逻辑而不是生命周期来组织代码。你可以把订阅和取消订阅写在同一个 Effect 里，让相关代码“聚合”
- 函数组件通过闭包访问状态，彻底抛弃了 this。这更符合函数式编程的直觉，代码也更加简洁易读。
- 函数对压缩工具（如 Terser）更友好，更利于减小最终的 Bundle Size，提升首屏加载速度。

### React Hooks的顺序问题

问题：既然 Hooks 这么完美，那为什么它会有‘不能在条件语句中调用’这种限制？这个限制的背后是怎样的存储数据结构

回答如下：
React 必须依靠 Hook 调用的 “物理顺序” 来确定每个 useState 或 useEffect 对应的是哪一个状态。
在 React 内部，每一个组件实例（Fiber 节点）上都有一个属性叫 memoizedState。它并不是一个对象，而是一个按顺序排列的链表。

- 初次渲染（Mount）： React 按照 Hook 出现的顺序，依次在内存中创建节点。
- 后续渲染（Update）： React 再次执行函数组件，它从头开始依次读取链表中的节点

Hooks 必须写在顶层，根本原因在于 React 内部使用 单向链表 来存储 Hook 的状态。
每一个 Fiber 节点的 memoizedState 属性记录了这个链表的头节点。由于 React 并不记录 Hook 的名称，它唯一标识 Hook 的方式就是 执行顺序。
如果我们在 if 语句中调用 Hook，就会导致在不同次渲染之间，Hook 的调用次数或顺序发生变化。这会造成‘链表索引偏移’，使得 React 无法正确地将之前的状态恢复到对应的变量上。为了保证状态的稳定性（State Stability），React 规定了 Hook 必须在每轮渲染中以相同的顺序被调用

# 云原生

## Docker 镜像体积优化

问题：你曾与 DevOps 团队协作将 Docker 镜像体积缩减了 2/3 。请分享一下具体的优化策略。

- 采用多阶段构建 (Multi-stage Builds)：将构建过程分为“构建环境”和“运行环境”，第一阶段（Build） 安装所有依赖（包括 devDependencies），进行 TypeScript 编译、代码混淆或前端构建（如 Angular/React 构建）。在 第二阶段（Runtime） 只从第一阶段拷贝编译后的产物（如 dist 文件夹）以及生产环境必需的 node_modules。

- 精选基础镜像 (Base Image Selection)：切换到 node:alpine：Alpine Linux 极其精简，通常只有 5MB 左右，整个 Node 环境可以控制在 100MB 出头

- 生产环境依赖瘦身 (Pruning Dependencies)：将node_modules里的markdown文件，测试代码，typescript文件等不需要的文件清除。（实测效果不佳）

- 优化镜像层 (Layer Optimization)；将多个 RUN 命令合并为一个，并在同一层中清理缓存；合理的 .dockerignore： 确保 .git、本地 node_modules、日志文件和环境配置文件不被复制进镜像镜像，这不仅能缩减体积，还能加速构建。


# 业务

## WO

### 引入golang技术栈

问题：你提到为了处理百万级用户和千万级消息，果断引入了 Go 语言栈 处理异步消息 。请详细聊聊当时 Node.js 遇到了什么样的瓶颈？在引入 Go 之后，你是如何设计跨语言（Node.js 与 Go）的服务通信和数据一致性保障的？

回答：

- Node.js 当时遇到了什么样的瓶颈： 在 Workforce Orchestrator (WO) 项目的早期，我们使用 Node.js 承载了所有的业务逻辑，包括高并发的消息分发和异步数据处理。随着用户量和消息量冲向百万与千万级，Node.js 暴露出以下核心瓶颈：CPU 密集型任务阻塞事件循环（Event Loop）；高并发下的内存暴涨（OOM）。

- 引入 Go 后，如何设计跨语言服务通信： 我们并没有完全废弃 Node.js，而是采用了 “前轻后重”的微服务架构：Node.js 继续负责面向用户侧、业务逻辑复杂、多变的前端 Portal/Admin 服务和 GraphQL BFF 层；而 Go 专门负责高并发、CPU 密集的异步消息处理与数据同步。

```
[前端/客户端] 
      │ (HTTPS/WSS)
      ▼
[Node.js BFF (GraphQL/Express)] 
      │ 
      ├─► (高效数据写入) ──► [PostgreSQL] ◄── (读取/分析) ──┐
      │                                                  │
      ├─► (发布消息) ──► [Google Pub/Sub 消息队列]          │
                                │                        │
                                ▼                        │
                        [Go 异步数据处理服务] ──────────────┘
```


## BCP

### 使用patch-package修改next auth

问题：在 BCP 项目中，你提到使用 patch-package 深度定制了 Next-Auth 以解决企业代理环境下的认证问题 。能具体描述一下那个“坑”是什么吗？你是如何定位到 Next-Auth 源码问题的，以及最终的补丁逻辑是如何绕过或兼容代理的

回答如下：

在企业内网环境下，所有的外网请求（甚至是部分内网跨域请求）都必须经过 Corporate Proxy（企业代理）。当集成 Next-Auth 进行 Azure AD（或内部 OIDC 服务）认证时，服务端在交换 Token 的阶段（即服务器对服务器的后端通信）出现访问超时的问题。

定位过程如下：浏览器可以访问 Azure AD的服务 -> 终端不可以ping通Azure AD的服务 -> 配置http_proxy环境变量之后终端可以ping通Azure AD的服务 -> 配置http_proxy环境变量之后next auth还是无法访问配置Azure AD的服务。

解决思路：使用[Add support for HTTP Proxy](https://next-auth.js.org/tutorials/corporate-proxy)

解决方案：为了不破坏项目的整体架构，同时确保 CI/CD 流程能自动应用修复，我选择了 patch-package引入对next auth的源代码改动。

经验：

- 要迷信流行库的开箱即用。在受限的企业内网（High-Security Environment）中，很多开源库的默认行为（如网络请求、证书校验）都会失效
- 平衡‘改源码’与‘可维护性’。直接修改 node_modules 是大忌，但通过 patch-package 将修复作为代码版本的一部分提交，既解决了 BCP PoC 的紧急上线问题，又为团队留下了清晰的文档记录，直到官方在后续版本中通过 httpOptions 彻底修复了这个扩展性问题。

## 积分引擎

无

## GSS Data Store

### 使用Podman部署

问题：你提到在虚拟机上通过 Podman systemd 实现了环境的自动化自愈 。在有了 Kubernetes 的情况下，为什么还需要在虚拟机上做这套方案？

回答如下：

- 当时公司的 Kubernetes 生产环境还在建设或调优阶段，不具备上生产的能力。
- 虚拟机部署经验成熟可靠。
- 利用 podman generate systemd 命令将容器的生命周期挂载到系统的 systemd 服务中。
- 在 systemd 的配置中设置 Restart=always 和 StartLimitInterval。这样即使容器因为宿主机每周末定期重启，宿主机的操作系统内核会自动检测并重新拉起服务。
- 相比于简单的脚本，systemctl 提供了成熟的状态检查（Status Check）和日志回溯（Journald），能确保在没有 K8s 控制平面监控的情况下，服务依然具备基本的健壮性

