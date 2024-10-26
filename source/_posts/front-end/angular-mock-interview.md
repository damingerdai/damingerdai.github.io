---
title: Angular的模拟面试
date: 2024-10-26 16:16:38
tags:
---

# 前言

本文是针对angular前端的模拟面试

# 问题

## 前端CSS的优先级

在 CSS 中，选择器用于选择 HTML 元素并应用样式。不同选择器的优先级（权重）决定了样式的应用顺序。以下是常用选择器类型及其优先级规则：

1. CSS 选择器类型
    - 元素选择器：直接选择 HTML 元素，如 div、p、h1。
    - 类选择器：通过类名选择元素，使用 . 前缀，如 .box。
    - ID 选择器：通过 ID 名称选择元素，使用 # 前缀，如 #header。
    - 属性选择器：通过元素属性选择元素，如 [type="text"]。
    - 伪类选择器：选择特定状态的元素，如 :hover、:focus。
    - 伪元素选择器：选择元素的一部分，如 ::before、::after。
    - 组合选择器：通过多种选择器的组合，选择满足特定条件的元素。常见的组合符有：
        - 后代选择器：div p，选择 div 中的所有 p 元素。
        - 子选择器：div > p，选择 div 的直接子元素 p。
        - 相邻兄弟选择器：h1 + p，选择紧跟在 h1 之后的 p 元素。
        - 通用兄弟选择器：h1 ~ p，选择 h1 之后的所有 p 元素。

2. CSS 选择器优先级（权重）计算规则
CSS 中的优先级使用权重值进行计算，以确定样式的应用顺序。权重值的计算通常按四位数来表示（a, b, c, d），不同选择器类型的权重值如下：

    - 内联样式：1000 权重，最高优先级（a=1，如 \<div style="color: red;"\>）。
    - ID 选择器：0100 权重，每个 ID 选择器会为权重 b 加 1。
    - 类选择器、属性选择器、伪类选择器：0010 权重，每个类、属性、伪类为权重 c 加 1。
    - 元素选择器、伪元素选择器：0001 权重，每个元素或伪元素为权重 d 加 1。
    - 通配选择器 *：没有权重，不影响优先级。

优先级计算示例：

```css
#header .menu-item a 的优先级：0100 + 0010 + 0001 = 0111
div > p 的优先级：0001 + 0001 = 0002
h1::before 的优先级：0001 + 0001 = 0002
.nav .link:hover 的优先级：0010 + 0010 = 0020
```

3. 层叠规则（重要性）
如果多个选择器的优先级相同，那么后定义的样式将覆盖先定义的样式。如果样式带有 !important 标记，则会直接覆盖常规样式的优先级。多个 !important 样式冲突时，遵循优先级和源顺序（后者覆盖前者）。

### 总结
优先级从高到低：内联样式 > ID 选择器 > 类选择器、属性选择器、伪类选择器 > 元素选择器、伪元素选择器。
使用 !important 可以提高样式优先级，但应谨慎使用，以免降低代码可维护性。

## CSS主题切换的思路，代码怎么做全局的变量

方案一：

在 Angular 中实现 CSS 主题切换的思路，通常利用CSS 变量和全局样式，并结合 Angular 的服务来管理和切换主题。可以通过动态切换 data-theme 属性，实现全局的主题切换。

方案二：

使用scss和切换link的方式。

方案三：

使用css in js的思路，不一定支持angular

## 网页的暗黑模式怎么实现

实现网页的暗黑模式，通常可以使用**CSS 变量**、**媒体查询**、**JavaScript** 等方法来进行切换。以下是几种常见的实现方式：

### 1. 使用 CSS 变量实现暗黑模式

这种方式通过定义 CSS 变量来控制主题颜色，再使用 JavaScript 切换 CSS 变量值。

#### 1.1 定义 CSS 变量

在 `:root` 中定义亮色模式的 CSS 变量，并为暗色模式设置一个 `data-theme="dark"` 属性时的变量值。

```css
/* 定义亮色模式 */
:root {
  --background-color: #ffffff;
  --text-color: #000000;
}

/* 定义暗黑模式 */
[data-theme="dark"] {
  --background-color: #1e1e1e;
  --text-color: #ffffff;
}

/* 使用变量应用到样式 */
body {
  background-color: var(--background-color);
  color: var(--text-color);
  transition: background-color 0.3s, color 0.3s;
}
```

#### 1.2 使用 JavaScript 切换主题

使用 JavaScript 监听用户的点击事件，在页面根元素上添加或移除 `data-theme="dark"` 属性，切换暗黑和亮色模式。

```javascript
// 切换暗黑模式
function toggleDarkMode() {
  const currentTheme = document.documentElement.getAttribute("data-theme");
  const newTheme = currentTheme === "dark" ? "light" : "dark";
  document.documentElement.setAttribute("data-theme", newTheme);
  localStorage.setItem("theme", newTheme); // 保存用户选择
}

// 初始化主题
const savedTheme = localStorage.getItem("theme") || "light";
document.documentElement.setAttribute("data-theme", savedTheme);
```

#### 1.3 为按钮添加事件监听

在 HTML 中添加一个按钮来切换主题。

```html
<button onclick="toggleDarkMode()">切换暗黑模式</button>
```

### 2. 使用媒体查询自动匹配系统主题

CSS 提供了一个 `prefers-color-scheme` 媒体查询，可以自动检测用户系统设置，匹配亮色或暗黑模式。

```css
/* 默认亮色模式 */
:root {
  --background-color: #ffffff;
  --text-color: #000000;
}

/* 暗黑模式 */
@media (prefers-color-scheme: dark) {
  :root {
    --background-color: #1e1e1e;
    --text-color: #ffffff;
  }
}

body {
  background-color: var(--background-color);
  color: var(--text-color);
}
```

此方法会根据用户的系统主题设置自动应用亮色或暗黑模式，用户可以在系统设置中调整。

### 3. 使用 CSS 和 JavaScript 结合用户首选项

可以结合媒体查询和 JavaScript 自动检测用户系统设置，同时提供按钮允许用户自定义选择。

#### 3.1 检测用户系统主题设置

当用户首次访问网站时，检测系统主题，优先使用用户系统的主题偏好。如果用户选择了特定的主题，则将其偏好保存到 `localStorage` 中。

```javascript
function initTheme() {
  const savedTheme = localStorage.getItem("theme");

  if (savedTheme) {
    document.documentElement.setAttribute("data-theme", savedTheme);
  } else if (window.matchMedia("(prefers-color-scheme: dark)").matches) {
    document.documentElement.setAttribute("data-theme", "dark");
  } else {
    document.documentElement.setAttribute("data-theme", "light");
  }
}

initTheme();
```

#### 3.2 切换主题并保存用户选择

```javascript
function toggleDarkMode() {
  const currentTheme = document.documentElement.getAttribute("data-theme");
  const newTheme = currentTheme === "dark" ? "light" : "dark";
  document.documentElement.setAttribute("data-theme", newTheme);
  localStorage.setItem("theme", newTheme);
}
```

#### 3.3 HTML 示例

```html
<button onclick="toggleDarkMode()">切换暗黑模式</button>
```

### 总结

- **CSS 变量法**：定义变量并通过 `data-theme` 切换，非常灵活。
- **媒体查询**：自动适应用户系统主题设置，便于实现无缝暗黑模式。
- **JavaScript 保存用户偏好**：在 `localStorage` 中记录用户选择，提供更个性化的体验。

## Angular 哪些方式能获取dom节点

Angular 提供了多种方式来获取 DOM 节点，每种方式都有其适用场景。选择哪种方式取决于具体的需求和场景。

- ViewChild 和 ViewChildren: 用于获取模板中定义的元素或指令。
- ElementRef: 用于获取组件的根元素。
- @HostListener: 用于监听宿主元素的事件。
- Renderer2: 用于在组件中直接操作 DOM。
- 原生JavaScript。

一般情况下，推荐使用 ViewChild、ViewChildren 和 ElementRef 来获取 DOM 元素，因为它们与 Angular 的变更检测机制集成得更好。

## Angular里面service的实现，service是应用在全生命周期里吗，为什么？

在 Angular 中，服务（Service）是一种用于提供特定功能、数据和业务逻辑的单例对象。服务的设计和实现使得它们在整个应用程序的生命周期内可以被共享和重用。下面是关于 Angular 服务的实现及其在全生命周期中的应用原因的详细说明。

### 1. 服务的基本实现

服务通常是一个使用 `@Injectable` 装饰器的类，允许将其注入到组件、其他服务或指令中。下面是一个简单的服务示例：

```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root' // 这里的 root 表示该服务在整个应用的生命周期内是单例的
})
export class ExampleService {
  private data: string;

  constructor() {
    this.data = 'Hello, World!';
  }

  getData(): string {
    return this.data;
  }

  setData(newData: string): void {
    this.data = newData;
  }
}
```

### 2. 服务的生命周期管理

服务的生命周期与应用程序的生命周期相同。这是通过提供服务的模块作用域来实现的。在 Angular 中，服务的实例通常是在根注入器（root injector）中创建的，从而使其在整个应用程序中共享。

#### 2.1 单例模式

使用 `@Injectable` 装饰器并将 `providedIn` 设置为 `root`，服务会被注册为单例，这意味着在应用的生命周期内只会创建一个服务实例。所有注入了该服务的组件和其他服务都将共享同一个实例。这样做的原因有以下几点：

- **数据共享**：多个组件可以共享和访问同一个服务实例的数据，便于进行状态管理和跨组件通信。
  
- **性能优化**：由于只创建一个实例，减少了内存使用和性能开销。

- **全局访问**：任何组件或服务都可以随时访问和调用该服务的方法。

### 3. 服务的使用场景

- **业务逻辑处理**：将复杂的业务逻辑从组件中抽离到服务中，保持组件的简洁和可维护性。

- **数据共享和状态管理**：在多个组件之间共享数据，避免了 prop drilling（层层传递数据）。

- **HTTP 请求**：通常使用服务来处理 HTTP 请求，这样可以集中管理 API 调用，方便处理错误和响应。

- **配置和环境变量**：服务可以用于存储全局配置和环境变量，供各个组件使用。

### 4. 具体示例

下面是一个具体示例，展示如何在组件中使用服务：

```typescript
import { Component, OnInit } from '@angular/core';
import { ExampleService } from './example.service';

@Component({
  selector: 'app-example',
  template: `
    <h1>{{ data }}</h1>
    <button (click)="changeData()">改变数据</button>
  `
})
export class ExampleComponent implements OnInit {
  data: string;

  constructor(private exampleService: ExampleService) {}

  ngOnInit(): void {
    this.data = this.exampleService.getData(); // 获取服务中的数据
  }

  changeData(): void {
    this.exampleService.setData('新数据'); // 修改服务中的数据
    this.data = this.exampleService.getData(); // 更新组件中的数据
  }
}
```

在这个示例中，`ExampleComponent` 使用 `ExampleService` 来获取和设置数据。无论有多少个 `ExampleComponent` 实例，它们都会共享同一个 `ExampleService` 实例。

### 5. 总结

- Angular 服务是应用程序的单例对象，具有全局共享的特性。
- 服务的生命周期与整个应用程序相同，使得跨组件数据共享和业务逻辑管理变得简单而高效。
- 通过使用服务，能够提高代码的可维护性和可重用性，同时简化组件的结构。

## 图片优化

1. 使用 srcset 和 sizes 属性
2. 使用NgOptimizedImage
3. 使用picture结合webp格式

## Angular做到多少版本，要从14升级到18怎么做

使用https://angular.dev/update-guide

## 敏捷开发相较于之前的瀑布流的区别，敏捷开发的时间估算，多久一个sprint

敏捷开发与传统的瀑布流开发方法相比，具有一些显著的区别，尤其是在流程、灵活性和团队协作方面。以下是一些主要区别和敏捷开发中时间估算的相关信息。

### 1. 敏捷开发与瀑布流的区别

- **开发流程**：
  - **瀑布流**：线性流程，依次进行需求分析、设计、开发、测试和维护。每个阶段完成后再进入下一个阶段，通常不允许返回。
  - **敏捷开发**：迭代和增量的流程，强调短周期的开发和频繁交付，每个迭代（Sprint）都会进行需求、设计、开发、测试等步骤。

- **灵活性和适应性**：
  - **瀑布流**：对需求变化的适应性较差，需求一旦确定，后续阶段难以调整。
  - **敏捷开发**：鼓励频繁反馈和适应变化，团队可以根据用户反馈和市场变化灵活调整需求。

- **用户参与**：
  - **瀑布流**：用户通常在初期提供需求，后续参与较少，直到最终交付。
  - **敏捷开发**：用户和利益相关者持续参与，通过迭代反馈确保产品符合用户需求。

- **团队协作**：
  - **瀑布流**：团队角色较为分明，沟通相对较少，依赖文档。
  - **敏捷开发**：强调团队的跨职能协作，定期召开会议（如站会、回顾会）促进沟通和协作。

### 2. 敏捷开发的时间估算

在敏捷开发中，时间估算通常涉及以下几个方面：

- **Sprint 的周期**：
  - 一个 Sprint 通常持续 **1 到 4 周**，常见的做法是 2 周。团队可以根据项目需求和团队成员的工作节奏选择合适的周期。

- **时间盒（Time Box）**：
  - 敏捷方法强调使用时间盒来控制工作节奏。在每个 Sprint 中，团队会定义要完成的任务，并在规定的时间内努力完成。

- **故事点和任务估算**：
  - 使用故事点（Story Points）来评估工作量，通常基于相对复杂度和完成时间。团队可以通过回顾历史数据来进行更准确的时间估算。

### 3. Sprint 计划

- **Sprint 计划会议**：在每个 Sprint 开始时，团队会召开计划会议，确定要在该 Sprint 中完成的任务。团队会根据优先级和容量进行任务分配。

- **每日站会**：在 Sprint 进行中，团队每天进行短暂的站会，讨论进展、遇到的问题和计划。

### 总结

- **敏捷开发**：强调迭代、灵活性和团队协作，能够快速响应变化。
- **Sprint 周期**：通常为 1 到 4 周，2 周为常见做法。
- **时间估算**：使用故事点、任务估算和时间盒的方法来提高准确性和效率。

这些特点使敏捷开发适合快速变化的项目需求和高动态环境。