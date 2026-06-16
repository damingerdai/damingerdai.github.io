---
title: 在 Next.js 16 (Turbopack) 中集成 RDKit.js 渲染 SMILES 的工程实践与踩坑指南
date: 2026-06-16 20:54:34
tags: [nexjts,rdkitjs, smiles]
categories: [前端]
---

# 在 Next.js 16 (Turbopack) 中集成 RDKit.js 渲染 SMILES 的工程实践与踩坑指南

## 前言

在生物制药数字化（AIDD、LIMS 等系统）的前端开发中，SMILES（简化分子线性输入规范）是表示分子结构最常用的文本格式。为了在前端实现“输入文本，实时预览 2D 分子结构”，我们需要引入化学信息学界的工业级开源工具包：RDKit.js。

由于 RDKit.js 底层依赖重型的 WebAssembly (Wasm) 编译产物，当它遇到 Next.js 16 默认的 Turbopack 构建流以及 SSR（服务端预渲染） 架构时，会引发一系列经典的打包与运行时崩溃。

本文将完整记录基于 Next.js 16 + Bun + TypeScript 栈集成 RDKit.js 的踩坑心路历程，并分享最终的离线化解决方案。

## 一、 初次尝试与经典的 “fs” 编译炸弹

按照常规的前端模块化思维，我们首先会通过 Bun 安装依赖：

```bash
bun add @rdkit/rdkit
```

然后在一个标准的 Client Component 中尝试动态加载：

```typescript
// 尝试通过全局或动态 import 载入
import initRDKitModule from "@rdkit/rdkit";
```

然而，一旦启动开发服务器，Next.js 16 会直接抛出无法编译的红屏错误:

```bash
Module not found: Can't resolve 'fs'
./node_modules/@rdkit/rdkit/dist/RDKit_minimal.js (7:655)

> 7 | ...if(ENVIRONMENT_IS_NODE){var fs=require("fs");...
    |                                 ^^^^^^^^^^^^^
```

### 为什么会报错？

RDKit.js 的底层胶水代码（由 Emscripten 编译生成）为了同时兼容 Node.js 和浏览器环境，其混淆代码中包含了 `if(ENVIRONMENT_IS_NODE){ var fs = require("fs"); }` 这样的环境判别逻辑。

虽然我们在代码中可能加了 `typeof window !== 'undefined'` 的运行时拦截，但 Next.js 16 默认的 Turbopack 编译器在静态编译打包阶段是非常死板的。只要它在扫描依赖树时看到了 `require("fs")`，就会固执地尝试在浏览器捆绑包里去打包 Node.js 的原生 fs（文件系统）模块。由于浏览器端根本没有文件系统，编译直接宣告崩溃。

### 为什么不能走 SSR 或 Server Action？

有人会想，既然浏览器端打包卡住，那把 RDKit 移到服务端运行，通过 SSR 或 Server Action 渲染好再返回不行吗？这条路同样是死胡同：

- **缺乏 Canvas 环境**：RDKit 渲染的核心方法（如 draw_to_canvas_with_highlights）需要接收一个真实的 HTMLCanvasElement 节点，利用浏览器的渲染上下文画图。服务端压根没有 DOM 和 Canvas 实例。
- **打字交互的网络雪崩**：用户在输入框手敲 SMILES 时是逐字输入的。如果每次按键都触发一次 Server Action 请求去服务端开辟 Wasm 内存、解析并返回，网络延迟会造成极其严重的卡顿和闪烁。分子结构的解析与预览必须留在客户端本地，实现零延迟响应。

## 二、 破局思维：静态映射与全局单例

既然不能通过常规的 Bundler 编译流去 import 它，我们就必须让编译器“闭眼”——将 RDKit 相关的 Wasm 和 JS 胶水代码作为纯静态资产对待，避开 Turbopack 的依赖扫描，利用浏览器的 window 对象进行纯客户端外挂加载。

### 1. 自动化构建：利用 postinstall 自动映射

我们不需要手动去复制物理文件。为了保证团队协作和 CI/CD 自动构建时资源不丢失，我们可以在 package.json 的 scripts 中配置一个 postinstall 钩子。这样每次 bun install 结束后，脚本会自动把需要的包文件同步到 Next.js 的 public 静态资源池中:

```json


{
  "scripts": {
    "postinstall": "mkdir -p public/rdkit && cp node_modules/@rdkit/rdkit/dist/RDKit_minimal.js public/rdkit/ && cp node_modules/@rdkit/rdkit/dist/RDKit_minimal.wasm public/rdkit/"
  }
}
```

执行一次 bun run postinstall，文件就会安静地躺在 public/rdkit/ 下，Turbopack 再也不会去扫描它们。

## 三、 完美的 TypeScript 类型驯服术

为了让这个“外挂”在全局 window 上的单例不报 TS 错误，我们需要在项目中补全它的类型声明。

### 1. 补充全局声明：types/rdkit.d.ts

在项目根目录下创建类型定义，明确全局缓存和初始化函数的签名：

```typescript
// 假设你已经将官方提供的 RDKitModule 等基础接口存放于 definitions.d.ts 中
import { RDKitModule, RDKitLoader } from './definitions';

declare global {
  interface Window {
    // 静态脚本加载后，挂载在 window 上的全局初始化函数
    initRDKitModule?: RDKitLoader;
    // 用于在客户端浏览器生命周期内缓存实例，实现全局单例
    _rdkitInstance?: RDKitModule;
  }
}
```

## 四、 实战：编写实时渲染组件

接下来，我们利用 Next.js 16 原生的 next/script 组件异步加载静态资源，并在组件中处理好高频打字时的内存释放。

```tsx
// components/molecule-preview.tsx
'use client';

import React, { useEffect, useRef, useState } from 'react';
import Script from 'next/script';
import type { RDKitModule, JSMol } from '../types/definitions';

interface MoleculePreviewProps {
  smiles: string;
  width?: number;
  height?: number;
}

export const MoleculePreview: React.FC<MoleculePreviewProps> = ({
  smiles,
  width = 240,
  height = 180,
}) => {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const [rdkit, setRdkit] = useState<RDKitModule | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isScriptLoaded, setIsScriptLoaded] = useState(false);

  // 1. 静态脚本加载完毕后的初始化逻辑（全局单例模式）
  const handleRDKitLoad = async () => {
    setIsScriptLoaded(true);
    if (typeof window === 'undefined' || !window.initRDKitModule) return;

    // 命中缓存，直接复用
    if (window._rdkitInstance) {
      setRdkit(window._rdkitInstance);
      return;
    }

    try {
      // 显式指定本地 public 下的 wasm 路径
      const instance = await window.initRDKitModule({
        locateFile: () => '/rdkit/RDKit_minimal.wasm',
      });
      window._rdkitInstance = instance; // 写入全局缓存
      setRdkit(instance);
    } catch (err) {
      console.error('RDKit Wasm 初始化失败:', err);
      setError('引擎初始化失败');
    }
  };

  // 2. 监听 SMILES 输入变化，实时绘制 2D 图形
  useEffect(() => {
    if (!rdkit || !canvasRef.current) return;

    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');

    // 输入为空时清空画布
    if (!smiles.trim()) {
      ctx?.clearRect(0, 0, width, height);
      setError(null);
      return;
    }

    let mol: JSMol | null = null;
    try {
      mol = rdkit.get_mol(smiles);

      if (mol && mol.is_valid()) {
        setError(null);
        ctx?.clearRect(0, 0, width, height);
        
        // 执行 2D 绘图
        mol.draw_to_canvas_with_highlights(
          canvas,
          JSON.stringify({ width, height, bondLineWidth: 1.8 })
        );
      } else {
        setError('无效的 SMILES 语法');
      }
    } catch (err) {
      // 捕获用户打字过程中，括号未闭合等临时的非法中间状态
      setError('结构解析中...');
    } finally {
      // 【极度重要】：WebAssembly 的 C++ 堆内存无法被 JS 的垃圾回收(GC)自动清理。
      // 必须在 finally 块中显式调用 delete()，否则高频打字时会导致浏览器内存泄漏甚至卡死。
      if (mol) {
        mol.delete();
      }
    }
  }, [rdkit, smiles, width, height]);

  return (
    <>
      {/* 载入本地 public 目录下的静态胶水代码 */}
      <Script
        src="/rdkit/RDKit_minimal.js"
        strategy="afterInteractive"
        onLoad={handleRDKitLoad}
        onError={() => setError('化学脚本加载失败')}
      />

      <div className="relative border border-slate-200 bg-white rounded-lg flex items-center justify-center min-h-[180px] p-2">
        <canvas
          ref={canvasRef}
          width={width}
          height={height}
          className={error ? 'opacity-20 grayscale' : 'opacity-100 transition-opacity'}
        />

        {/* 状态提示 */}
        {!rdkit && !error && (
          <span className="absolute text-xs text-slate-400 animate-pulse">
            {!isScriptLoaded ? '正在加载本地脚本...' : '正在初始化本地 Wasm...'}
          </span>
        )}

        {error && (
          <span className="absolute bottom-2 text-xs text-red-600 bg-red-50 px-2 py-0.5 rounded border border-red-100">
            {error}
          </span>
        )}
      </div>
    </>
  );
};
```

## 总结

针对 RDKit.js 这种横跨 Node 和浏览器端的重型 WebAssembly 库，在 Next.js 16 体系下，一味地去死磕 Bundler 的编译 fallback 配置往往事倍功半。

通过本次实践我们发现，采用“构建期利用 postinstall 剥离静态文件 $\rightarrow$ 运行时利用 next/script 绕过 Turbopack 扫描 $\rightarrow$ 全局 window 实现单例懒加载”的曲线救国路线，反而带来了更多的工程优势：
1. 彻底离线化：完全脱离远端 CDN 的网络依赖，适合内网或实验室环境部署。
2. 绝对稳定：锁定了本地 node_modules 的版本，避免生产环境发生非预期升级。
3. 首屏编译无感：静态资产零加工，完美保留了 Next.js 16 的极致开发启动速度。