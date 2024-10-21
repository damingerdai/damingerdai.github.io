---
title: 模拟面试
date: 2024-10-20 20:45:06
tags: []
categories: [前端,nodejs, 敏捷开发,  面试]
---

## 敏捷开发

敏捷开发是一种以**迭代**和**增量**的方式来开发软件的流程，它注重**团队协作**、**快速反馈**以及**应对变化**的能力。在敏捷开发中，**Sprint（冲刺）** 是一个核心概念。下面我为你总结了敏捷开发流程中的**Sprint**、迭代节奏、和时间周期：

### 1. **Sprint（冲刺）**
- **Sprint** 是敏捷开发的基础单位，通常是一个**固定时间周期**，通常为**1到4周**。
- 每个 Sprint 都是一个**小型迭代周期**，在这个周期中，团队需要完成一定数量的工作（通常是具体的功能、用户故事或任务）。
- **目标**是在每个 Sprint 结束时交付一个**可工作的产品增量**，即使不一定是完整产品，也应该是能够演示或发布的可用功能。

### 2. **迭代节奏**
敏捷开发的迭代节奏是基于 Sprint 的。团队会按照以下步骤循环进行开发：

1. **Sprint 计划会议（Sprint Planning）**：
   - 在每个 Sprint 开始时，团队会召开计划会议，确定当前 Sprint 的**目标**和**待完成的任务**。这些任务一般从**产品待办事项列表（Product Backlog）**中选取，并拆分成更小的**用户故事（User Stories）**或**任务（Tasks）**。
   - 团队会评估每个任务的工作量，并确保任务的总量符合 Sprint 的工作周期。

2. **Sprint 执行**：
   - 团队根据计划开始执行任务，过程中可能会有**每日站会（Daily Standup）**，团队成员汇报进展、讨论问题以及调整任务优先级。
   - 每个 Sprint 中，团队尽量保持快速迭代，在尽可能短的时间内交付有价值的功能。

3. **Sprint 评审会议（Sprint Review）**：
   - 在 Sprint 结束时，团队会召开评审会议，展示在 Sprint 中完成的工作成果，通常是一个**可演示的产品功能**。
   - 团队与利益相关者一起**检验产品增量**，并对下一个 Sprint 的工作进行讨论和调整。

4. **Sprint 回顾会议（Sprint Retrospective）**：
   - 在 Sprint 结束后，团队还会召开回顾会议，讨论 Sprint 中的**问题**和**改进点**。
   - 这个会议帮助团队在每个 Sprint 后不断改进工作流程、沟通方式和工具。

### 3. **时间周期**
- 每个 Sprint 的时间周期通常是**固定的**，并且根据团队和项目的需求而定，常见的周期为：
  - **1周**：适用于需求非常紧急的项目，团队需要快速反馈和迭代，但要求团队有很强的适应能力。
  - **2周**：这是最常见的周期，能让团队有足够的时间完成可交付的功能，同时保持灵活应对变化。
  - **3到4周**：适用于更复杂的功能开发，需要较长时间来完成，但仍然遵循敏捷的短周期迭代原则。

### 4. **持续交付和反馈**
敏捷开发的一个核心目标是尽早交付可用的产品并从用户处获得反馈。因此，每个 Sprint 结束时，团队都会交付**可工作的产品增量**，并依据**用户反馈**调整下一阶段的开发工作。这个流程反复进行，直到项目完成或需求停止。

### 总结：
敏捷开发流程是通过多个 Sprint 迭代实现的，每个 Sprint 通常持续 1-4 周。在每个 Sprint 中，团队会计划任务，进行开发和测试，并交付可用的产品增量。通过频繁的反馈和不断优化，团队能够应对需求变化并持续改进开发效率。

## 技术

### javascript的闭包是什么，能不能写一个例子

闭包是指函数可以记住并访问它所在的词法作用域，即使这个函数在词法作用域之外执行。这是 JavaScript 强大的功能之一，常用于隐藏数据、模拟私有变量、创建回调函数等场景。

闭包的三个特性：
- 函数嵌套：闭包通常是内部函数访问外部函数的变量。
- 作用域链：函数在定义时绑定了其所在的词法作用域。
- 外部函数执行后，内部函数依然能够访问外部函数的变量。

例子：

```JavaScript
function createCounter() {
  let count = 0; // 外部函数的局部变量

  return function() { // 返回一个内部函数，形成闭包
    count++; // 内部函数能够访问外部函数的变量
    console.log(count);
  };
}

const counter = createCounter(); // 调用外部函数，返回内部函数

counter(); // 输出: 1
counter(); // 输出: 2
counter(); // 输出: 3

```

### javascript的原型链是什么

JavaScript的原型链是用于实现继承的一种机制。

- 每个对象都有一个**__proto__**属性，指向它的原型对象。
- 如果在对象上找不到某个属性或方法，JavaScript 会沿着原型链向上查找，直到找到该属性或方法，或者到达原型链的顶端 null。
- 函数的 prototype 属性是其实例对象的原型。通过这个机制，JavaScript 实现了属性和方法的共享和继承。
- 原型链的顶端是 Object.prototype，它的 __proto__ 指向 null，表示原型链的结束

例子：

```javascript
function Person(name) {
  this.name = name;
}

Person.prototype.sayHello = function() {
  console.log(`Hello, my name is ${this.name}`);
};

const person = new Person('John');
person.sayHello(); // Hello, my name is John

console.log(person.__proto__ === Person.prototype); // true
console.log(Person.prototype.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__ === null); // true
```

### javascript中new运算符做了什么

当你在 JavaScript 中使用 new 运算符时，它会执行以下四个步骤：

- 创建一个新的空对象。
- 将这个新对象的**__proto__**属性设置为构造函数的prototype对象，从而实现原型链继承。
- 调用构造函数，并将新对象作为函数内部的 this，让构造函数能够为新对象添加属性。
- 如果构造函数返回的是对象类型的值，则返回这个对象；否则，返回新创建的对象

### let和const有什么区别

let 允许重新赋值，而 const 不允许重新赋值。const 必须在声明时初始化，而 let 可以在声明后再赋值。

- 使用 let：适合需要在后续修改变量值的场景。
- 使用 const：适合声明不需要重新赋值的常量，尤其是用来声明不可变的原始类型（如数字、字符串）。对于对象和数组，虽然引用不能改变，但内部内容可以修改。

### interface和type的区别， interface能不能继承type？

interface 和 type 都可以定义对象的形状，但 interface 更适合面向对象的场景，而 type 更灵活，适合定义联合类型、元组等复杂类型。
interface 可以继承 type，反过来也可以通过交叉类型实现扩展。
interface 支持合并声明，这在库开发和扩展时非常有用，而 type 则无法合并。

### apply, call, bind 的区别

apply、call 和 bind 是 JavaScript 中用来改变函数内部 this 指向的方法，它们的核心功能是类似的，但在使用方式和行为上有所不同。

核心区别总结：
1. apply：传递参数时使用数组形式，立即调用函数。
2. call：传递参数时使用逗号分隔，立即调用函数。
3. bind：返回一个绑定了 this 的新函数，但不会立即执行。

书写apply：

```javascript
Function.prototype.myApply = function(context, argsArray) {
  // 1. 如果没有提供 context，则默认设置为全局对象 (在浏览器中是 window，在 Node.js 中是 global)
  context = context || globalThis;

  // 2. 给 context 添加一个临时方法，指向当前的函数 (this)
  const fnSymbol = Symbol();  // 使用 Symbol 避免属性冲突
  context[fnSymbol] = this;

  // 3. 执行函数，并传入参数数组
  let result;
  if (Array.isArray(argsArray)) {
    result = context[fnSymbol](...argsArray);  // 解构数组传参
  } else {
    result = context[fnSymbol]();  // 如果没有参数数组，则直接调用
  }

  // 4. 删除临时属性
  delete context[fnSymbol];

  // 5. 返回执行结果
  return result;
};
```

书写call：

```javascript
Function.prototype.myCall = function(context, ...args) {
  // 1. 如果没有提供 context，则默认设置为全局对象 (在浏览器中是 window，在 Node.js 中是 global)
  context = context || globalThis;

  // 2. 给 context 添加一个临时方法，指向当前的函数 (this)
  const fnSymbol = Symbol();  // 使用 Symbol 避免属性冲突
  context[fnSymbol] = this;

  // 3. 执行函数，并传入参数
  const result = context[fnSymbol](...args);  // 通过展开运算符传入参数

  // 4. 删除临时属性
  delete context[fnSymbol];

  // 5. 返回执行结果
  return result;
};

```

手写bind：

```javascript
Function.prototype.myBind = function(context, ...args) {
  // 保存原函数
  const originalFunc = this;

  // 返回一个新函数
  const boundFunction = function(...newArgs) {
    // 判断是否作为构造函数调用
    // 如果是构造函数调用，`this` 指向实例对象，应该忽略绑定的 `context`
    // 否则使用绑定的 `context`
    const isNew = this instanceof boundFunction;
    const thisArg = isNew ? this : context;

    // 使用 apply 绑定 `this`，合并 args 和 newArgs
    return originalFunc.apply(thisArg, [...args, ...newArgs]);
  };

  // 继承原函数的 prototype，确保绑定后的函数是原函数的实例
  if (originalFunc.prototype) {
    boundFunction.prototype = Object.create(originalFunc.prototype);
  }

  return boundFunction;
};
```

### react的事件合成

在 React 中，事件是通过合成事件（SyntheticEvent）机制处理的，它是对原生 DOM 事件的一种跨浏览器的封装，确保事件在所有浏览器中都有一致的行为。React 使用事件委托的方式，将所有事件统一绑定在根元素上，通过事件冒泡来捕捉子元素的事件触发。这种机制可以显著提高性能，因为它避免了在每个 DOM 节点上绑定独立的事件处理程序。

SyntheticEvent 的接口与浏览器的原生事件一致，像 preventDefault() 和 stopPropagation() 都可以直接使用。但需要注意的是，SyntheticEvent 是可回收的，React 会在事件处理后自动销毁事件对象，减少内存占用。如果需要在异步操作中使用事件对象，可以使用 event.persist() 方法防止事件被回收。

总的来说，React 的事件合成机制提供了性能优化和跨浏览器兼容性，是事件处理的基础工具。在 React 18 中，合成事件先于原生事件处理是为了确保性能优化、一致性和更好的用户体验。

要获取合成事件中的原生事件，你可以通过 event.nativeEvent 属性来访问。这允许你获取浏览器的原生事件对象，并直接与之进行交互。

在 React 中，阻止合成事件（SyntheticEvent）冒泡与阻止原生 DOM 事件冒泡的方法类似。你可以使用合成事件对象的 stopPropagation() 方法来阻止事件的冒泡行为。

当你调用 event.stopPropagation() 时，React 会阻止该事件继续向上传播到父级元素的事件处理程序。

### react的vdom是什么，有什么优势


React 的 Virtual DOM（虚拟 DOM）是 React 用来优化 UI 更新的一种技术。它是实际 DOM 的轻量级副本，通过在内存中模拟 DOM 树来进行高效的 UI 渲染和更新。

Virtual DOM 工作原理：

1. 当 React 组件首次渲染时，React 会在内存中创建一个 Virtual DOM 树。这个虚拟 DOM 是 React 元素的对象表示，它轻量且易于操作
2. 当组件的状态或属性发生变化时，React 会重新计算新的 Virtual DOM 树，但不会立即更新实际的 DOM
3. React 使用高效的差异算法（reconciliation）来比较新旧 Virtual DOM 树。它会找出新旧 Virtual DOM 树之间的差异（diff），即需要更新的部分
4. React 将这些差异应用到实际的 DOM 上，通过一次性最小化的更新来提升性能

Virtual DOM 的优势：

1. **性能优化**: 直接操作真实 DOM 是比较昂贵的，因为 DOM 的操作会导致浏览器的重新布局和重绘。而 Virtual DOM 是在内存中操作的，速度快得多。通过批量更新 DOM 的方式，React 将频繁的小改动合并为一次大的更新，减少不必要的重排和重绘，极大地提高了性能
2. **跨平台一致性**: Virtual DOM 提供了一种抽象的 DOM 结构，它不依赖于特定的平台。因此，React 可以用于浏览器环境、移动应用（React Native）、甚至服务器端渲染等不同平台，增强了代码的复用性
3. **可预测的 UI 更新**: 通过 Virtual DOM 和 React 的状态驱动（declarative）设计，React 可以确保 UI 的更新是可预测的，开发者只需专注于描述 UI 应该“如何呈现”，而不是手动控制具体的 DOM 更新过程
4. **简化复杂操作**: 在复杂的应用中，直接操作 DOM 可能会导致难以维护和调试的问题。React 的 Virtual DOM 屏蔽了复杂的 DOM 操作，让开发者只需要定义组件的状态和结构，React 自动处理 DOM 操作

Virtual DOM 的局限性：

1. **不总是最快的**：虽然 Virtual DOM 提供了高效的更新机制，但在某些非常简单或特定的场景下，直接操作 DOM 可能比使用 Virtual DOM 更快
2. **学习成本**： 需要理解 React 的状态管理、生命周期、Virtual DOM 等概念，可能对初学者有一定的学习门槛

React 的 Virtual DOM 是一种提升性能的优化技术，通过在内存中创建轻量级的虚拟 DOM 树，配合差异算法和批量更新策略，有效地提高了 UI 更新的效率，并简化了跨平台开发的复杂性。

### react中的hoc组件

在 React 中，HOC（Higher-Order Component，高阶组件） 是一种组件复用的高级技术。HOC 本质上是一个函数，它接受一个组件作为参数，并返回一个新的组件。高阶组件常用于逻辑复用、增强现有组件的功能，而不改变组件本身的实现。

HOC 的定义：
- 高阶组件是一个纯函数，它不修改传入的组件，只是返回一个增强后的新组件。

HOC 的工作原理：
- **参数是组件**：HOC 接收一个现有组件作为参数。
- **返回新组件**：它会返回一个新组件，该组件可以基于原组件进行额外的功能扩展。
- **不修改原组件**：HOC 不直接修改原组件，而是通过包装、组合的方式增强功能。

HOC 的常见用途：
- **复用逻辑**：多个组件共享相同的逻辑时，可以使用 HOC 将该逻辑提取出来。
- **处理权限控制**：HOC 可用于权限校验逻辑，例如检查用户是否有访问权限。
- **操作 props**：HOC 可以修改、增加或删除组件的 props。
- **操作生命周期**：通过 HOC 可以控制组件的生命周期，以实现缓存、请求拦截等功能。

### react组件间的通信方式

- **父子组件通信**：父组件通过props将数据传递给子组件。React 的数据流是单向的，从父组件流向子组件
- **子组件向父组件通信（回调函数 Props**：子组件无法直接修改父组件的数据，因此子组件可以通过调用父组件传递的回调函数来向父组件传递信息。父组件可以在回调函数中接收子组件传递的数据并更新状态
- **兄弟组件通信（通过父组件中转）**：由于 React 的单向数据流，兄弟组件之间不能直接通信。通常通过父组件中转实现兄弟组件间的通信。父组件将一个回调函数传递给一个子组件，当这个子组件触发回调函数时，父组件更新状态，然后将状态传递给另一个子组件
- **跨层级组件通信（Context API）**: 对于层级嵌套较深的组件，逐层通过 props 传递数据会导致代码臃肿。此时可以使用 Context API 来实现跨层级组件的通信。Context 提供了一个全局的状态或函数，允许任意组件订阅并访问这些数据
- **全局状态管理（Redux、MobX 等）**: 对于复杂的应用，尤其是有大量组件需要共享状态时，使用 Redux 或 MobX 等状态管理库可以帮助管理全局状态。Redux 的核心思想是将状态集中管理，组件通过订阅状态和分发（dispatch）动作来实现通信
- **Refs（父子组件直接引用通信）**: 有时需要直接访问子组件的某个实例或 DOM 节点，此时可以通过 Refs 实现父组件与子组件的通信。父组件可以通过 ref 获取子组件实例，并调用子组件的方法或访问子组件的 DOM 节点

### css中垂直居中的4种方式

- 使用 flexbox 实现垂直居中
- 使用 grid 实现垂直居中
- 使用 absolute positioning 和 transform 实现垂直居中
- 使用 line-height 实现垂直居中（仅限单行文本）