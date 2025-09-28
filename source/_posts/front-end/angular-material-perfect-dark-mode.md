---
title: 为 Angular Material 应用添加完美深色模式支持
date: 2025-09-28 21:47:38
tags: [ angular, angular material, dark mode]
categories: [前端]
---


# 💡 为 Angular Material 应用添加完美深色模式支持

深色模式（Dark Mode）是现代应用不可或缺的功能。它不仅能提升用户在低光环境下的舒适度，还能让应用看起来更专业、更时尚。

如果你正在使用 **Angular Material**，实现深色模式可以非常优雅和高效。本文将分享我如何通过一个独立的 **`ThemePickerComponent`**，结合 **Angular Signals** 和 **系统偏好检测**，为我的应用添加深色模式的完整过程。


## 🛠️ 核心思路概览

我的深色模式解决方案基于以下几个关键机制：

1.  **CSS 变量与 `color-scheme`**: 利用 Angular Material 基于 CSS 变量的主题机制，并通过在 `<html>` 标签上切换 `color-scheme` 属性来控制主题。
2.  **Angular Signals**: 使用 **`signal()`** 存储和管理当前的主题模式 (`'light'` 或 `'dark'`)。
3.  **持久化与偏好**: 通过 `localStorage` 记住用户的选择，同时使用 `window.matchMedia` 监听用户操作系统的偏好设置。


## 💻 关键代码解析与实现步骤

### 1\. 配置主题和 `color-scheme` (`styles.scss`)

在全局样式文件中，我们确保应用能响应 `color-scheme` 变化，并设置 Material 主题。

```scss
/* ui/src/styles.scss */

@use '@angular/material' as mat;

html, body {
  font-family: Roboto, "Helvetica Neue", sans-serif;
}

html {
  background-color: var(--mat-sys-surface);
  color: var(--mat-sys-on-surface);
  // ✨ 关键：允许浏览器知道页面支持两种颜色方案
  color-scheme: light dark;

  @include mat.theme((
        color: (
          // Material 会根据 color-scheme 自动应用 light/dark 调色板
          primary: mat.$rose-palette, 
          tertiary: mat.$red-palette,
      ),
      typography: Roboto,
      density: 0,
  ));
}
```

我们通过 `background-color: var(--mat-sys-surface)` 来使用 Material **系统级 CSS 变量**，确保背景颜色随着主题切换而正确变化。

### 2\. `ThemePickerComponent`：主题切换器

这是整个功能的核心。它负责初始化、监听和切换主题。

#### A. HTML 模板：根据模式切换图标

我们使用 `mat-icon-button` 和 `@if` 语法根据 `mode()` 的值显示太阳或月亮图标。

```html
<button class="theme-button" mat-icon-button
    [matTooltip]="'Change Theme'"
    (click)="changeMode()"
  >
     @if (mode() === 'dark') {
       <mat-icon>dark_mode</mat-icon>
     } @else {
       <mat-icon>light_mode</mat-icon>
     }
</button>
```

#### B. TypeScript 逻辑：Signals, Effects, 和持久化

```typescript
// ui/src/app/components/theme-picker/theme-picker.component.ts

import { afterNextRender, Component, effect, inject, OnInit, Renderer2, signal } from '@angular/core';
// ... 导入 MatButtonModule, MatIconModule, MatTooltipModule

@Component({...})
export class ThemePickerComponent implements OnInit {
  static storageKey = 'batcher-ui-theme';
  mode = signal('light');
  private renderer = inject(Renderer2);

  constructor() {
    // 1. 响应式更新：当 mode() 变化时，更新 <html> 元素的 color-scheme
    effect(() => {
      const mode = this.mode();
      this.renderer.setStyle(document.documentElement, 'color-scheme', mode);
    });

    // 2. 初始化：客户端渲染后，加载存储或系统偏好
    afterNextRender(() => {
      // 优先加载 localStorage 存储的模式，否则检测系统偏好
      const systemPreference = window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
      const storedMode = this.getStoredMode() || systemPreference;
      
      if (storedMode) {
        this.mode.set(storedMode);
      }
    });
  }

  ngOnInit(): void {
    // 3. 实时监听：如果用户未设置过偏好，跟随系统主题的实时变化
    window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', e => {
      const storedMode = this.getStoredMode();
      if (!storedMode) {
        this.mode.set(e.matches ? 'dark' : 'light');
      }
    });
  }

  changeMode(): void {
    const newMode = this.mode() === 'light' ? 'dark' : 'light';
    this.mode.set(newMode);
    this.storeMode(newMode);
  }

  // 4. 持久化方法 (storeMode, getStoredMode)
  storeMode(mode: string): void {
    try {
      localStorage.setItem(ThemePickerComponent.storageKey, mode);
    } catch { /* ignore */ }
  }

  getStoredMode(): string | null {
    try {
      return localStorage.getItem(ThemePickerComponent.storageKey);
    } catch { return null; }
  }
}
```

#### C. SCSS处理(可选)

修改icon的颜色去适配material 3的配色

```css
.theme-button {
    color: var(--mat-primary);
}

```

### 3\. 集成到导航栏 (`navigation.component.html`)

我们将 `app-theme-picker` 组件放置在应用导航栏（`mat-toolbar`）的右侧。

```html
<mat-toolbar>
    <div class="flex-grow-1"></div>
    <app-theme-picker class="theme-picker"></app-theme-picker>
</mat-toolbar>
```

**导航栏样式调整**

我们还调整了导航栏的背景色，使其使用更适合作为容器背景的 Material 系统变量。

```scss
/* ui/src/app/components/navigation/navigation.component.scss */

:host {
  color: var(--mat-sys-primary);
  // 更改为使用 surface-container-low，这通常更适合作为页面/容器的背景
  background: var(--mat-sys-surface-container-low);
}

// ... 省略其他样式

.flex-grow-1 {
  flex-grow: 1;
}
```


## ✨ 总结

通过上述实现，我们的 Angular Material 应用获得了：

1.  **用户自定义**: 用户可以随时通过点击按钮切换光亮/深色模式。
2.  **偏好记忆**: 应用会记住用户的选择，即使重新打开浏览器也不会丢失。
3.  **尊重系统设置**: 如果用户从未手动切换过主题，应用会默认跟随操作系统的偏好设置。

使用 Angular Signals 和 Effects 极大地简化了状态管理和响应式更新，让深色模式的实现既强大又简洁！

## 🔗 完整的代码变更
你可以查看这个 GitHub Commit 来了解所有相关的代码修改：

[fix(ui): add dark mode](https://github.com/damingerdai/batcher/commit/facf4c29833b549cf756aa2c711a0d329210bc79)