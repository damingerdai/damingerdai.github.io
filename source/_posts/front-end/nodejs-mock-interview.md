---
title: nodejs的模拟面试
date: 2024-10-29 20:11:28
tags: [nodejs, 面试]
categories: [前端]
---

## CommonJS是什么

CommonJS（简称CJS）是一种JavaScript模块化规范，最初为在服务端（如Node.js）实现模块化而设计。在CommonJS中，每个文件都被视为一个独立的模块，并通过`module.exports`导出和`require`引入其他模块，形成清晰的模块依赖结构。以下是CommonJS的几个关键点：

1. **模块导出和导入**：CommonJS使用`module.exports`导出内容，其他文件使用`require`导入。例如：

```javascript
// 导出模块
// math.js
module.exports = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
};

// 导入模块
const math = require("./math");
console.log(math.add(2, 3)); // 输出5
```

2. **同步加载**：CommonJS模块是同步加载的，这在服务端环境中是合理的，因为文件系统通常是本地的，读取速度快。但在浏览器端不适用，因为网络加载是异步的，CommonJS因此不适用于前端模块化。

3. **模块作用域**：CommonJS模块会为每个文件创建独立的作用域，因此不会污染全局命名空间。模块内定义的变量或函数不会泄露到其他模块。

4. **在Node.js中的应用**：Node.js遵循CommonJS规范，使开发者能够轻松创建模块化代码结构。CommonJS适合Node.js应用的模块化和依赖管理，但随着ECMAScript模块（ESM）的标准化，Node.js现已支持ESM标准。

## CommonJS中require是怎么实现的

在CommonJS中，`require`的实现是通过Node.js的`Module`系统来管理和加载模块的。`require`本质上是一个函数，用来加载模块、解析依赖、缓存已加载的模块，从而确保模块的高效加载。下面是`require`的实现原理分解：

### 1. **路径解析**

- `require`首先会解析模块的路径，以确定该路径对应的文件位置。
- 如果是核心模块（如`fs`、`path`等），Node.js会直接加载这些模块，因为它们内置在Node.js中。
- 如果是自定义模块或第三方库，Node.js会在`node_modules`目录中查找指定模块。
- Node.js的路径解析规则是先查找本地文件，然后查找`node_modules`，并遵循目录层级查找（从当前目录逐层往上）。

### 2. **模块缓存**

- Node.js会缓存已经加载的模块，保存在`require.cache`对象中。缓存的模块是一个`Module`对象，其`exports`属性包含模块的导出内容。
- 如果模块已经加载并存在于缓存中，`require`会直接从缓存返回导出的内容，避免重复加载。

### 3. **创建Module对象**

- 如果模块没有被缓存，Node.js会为该模块创建一个新的`Module`对象：
  ```javascript
  const module = new Module(filename);
  ```
- 这个`Module`对象包含`id`、`filename`、`loaded`和`exports`等属性，用来表示模块的唯一标识、文件路径、加载状态、导出对象等。

### 4. **加载模块并执行**

- `require`会读取模块文件内容，然后将代码包裹在一个自执行函数中，这个函数接收`exports`、`require`、`module`、`__filename`和`__dirname`五个参数，从而确保每个模块都有自己的作用域。
- 例如，假设模块代码是：
  ```javascript
  module.exports = {
    add: (a, b) => a + b,
  };
  ```
  加载时，Node.js会将其转换为如下结构：
  ```javascript
  (function (exports, require, module, __filename, __dirname) {
    module.exports = {
      add: (a, b) => a + b,
    };
  });
  ```
- `require`执行该函数，将执行结果赋值给`module.exports`。执行完后，`module.exports`就包含了模块的导出内容。

### 5. **返回module.exports**

- 执行完成后，`require`函数返回`module.exports`的内容。其他模块可以通过调用`require`访问该模块的导出对象。

### 6. **循环依赖处理**

- 如果模块A和模块B存在循环依赖，Node.js会在加载时创建一个未完成的`module.exports`对象放入缓存，使得相互依赖的模块可以访问未完成的内容，从而避免死循环。

### 简化版代码示例

以下代码是Node.js中`require`实现的简化示例：

```javascript
// 简化的 CommonJS require 实现
const Module = function (filename) {
  this.filename = filename;
  this.exports = {}; // 导出对象
  this.loaded = false; // 加载状态
};

Module._cache = {}; // 模块缓存

Module._load = function (filename) {
  // 检查缓存
  if (Module._cache[filename]) {
    return Module._cache[filename].exports;
  }

  // 创建新的模块并缓存
  const module = new Module(filename);
  Module._cache[filename] = module;

  // 读取文件内容并包装为函数
  const wrappedSrc = `(function(exports, require, module, __filename, __dirname) { ${readFileSync(filename)} \n})`;
  const compiledWrapper = eval(wrappedSrc);

  // 执行模块函数
  compiledWrapper(module.exports, require, module, filename, dirname(filename));

  // 设置加载完成状态
  module.loaded = true;

  return module.exports;
};

function require(filename) {
  return Module._load(filename);
}
```

### 总结

`require`的核心在于以下几步：

1. **解析路径**以找到模块文件。
2. **检查缓存**以避免重复加载。
3. **创建并缓存模块对象**，防止循环依赖。
4. **包装和执行代码**以提供模块独立作用域。
5. **返回module.exports**的内容供调用方使用。

这就是CommonJS中`require`的基本实现逻辑，通过缓存、作用域隔离等机制，确保模块的高效、独立运行。

## CommonJS和ESM的区别：

- **导入/导出语法**：CommonJS使用`require`和`module.exports`，而ESM使用`import`和`export`。
- **加载方式**：CommonJS是同步加载，ESM支持异步加载。
- **执行时机**：CommonJS模块在加载时立即执行，ESM模块则在编译时静态分析依赖关系。

总体来说，CommonJS是Node.js中常见的模块化规范，对服务端开发尤其重要，而在前端开发中更推荐使用ES模块化（ESM）标准。

## ESM是什么

ESM（ECMAScript Module）是JavaScript在ECMAScript 6（ES6）标准中引入的模块系统，也称为ES模块。它为JavaScript提供了一种官方、标准化的模块化方式，在浏览器和Node.js环境中都支持。ESM解决了JavaScript早期模块化规范（如CommonJS、AMD）存在的诸多问题，并引入了一些重要的新特性。

### ESM的特点和语法

1. **静态导入和导出**

   - ESM支持静态分析，这意味着在编译阶段，JavaScript引擎就能确定模块依赖关系，而不是像CommonJS那样动态加载。这种静态结构有助于性能优化和代码检查。
   - **导出**使用`export`关键字，支持命名导出和默认导出：

     ```javascript
     // named export
     export const add = (a, b) => a + b;
     export function subtract(a, b) {
       return a - b;
     }

     // default export
     export default function multiply(a, b) {
       return a * b;
     }
     ```

   - **导入**使用`import`关键字，可以选择导入模块的部分内容，或者直接导入默认导出：
     ```javascript
     import { add, subtract } from "./math.js";
     import multiply from "./math.js";
     ```

2. **支持异步加载**

   - 在浏览器环境中，ES模块可以异步加载，且在模块中可以使用`<script type="module">`标签加载脚本，且模块默认是异步加载的。这使得ESM在浏览器端比CommonJS更高效。
   - 例如，`import()`动态导入函数允许在运行时加载模块，是一种非常灵活的用法：
     ```javascript
     import("./math.js").then((module) => {
       console.log(module.add(2, 3));
     });
     ```

3. **模块作用域**

   - ESM模块代码默认在模块作用域内执行，不会污染全局作用域。
   - ESM模块中的顶层`this`值为`undefined`，避免了变量污染。

4. **浏览器和Node.js的支持**

   - 浏览器原生支持ESM，不需要任何工具或库即可直接加载。
   - Node.js 12及以上版本也支持ES模块（文件后缀为`.mjs`，或在`package.json`中指定`"type": "module"`）。这使得ESM成为了跨平台的标准模块化方案。

5. **Tree Shaking**
   - 由于ESM是静态结构的模块，构建工具（如Webpack、Rollup）可以在编译阶段优化代码，删除未使用的导出（Tree Shaking），减少最终打包文件的大小。

### ESM与CommonJS的区别

| 特性             | ESM                  | CommonJS                     |
| ---------------- | -------------------- | ---------------------------- |
| 导入/导出语法    | `import` / `export`  | `require` / `module.exports` |
| 依赖解析时间     | 静态解析             | 动态解析                     |
| 加载方式         | 异步加载（浏览器端） | 同步加载                     |
| 缓存机制         | 缓存，但不可变       | 缓存，但内容可变             |
| 顶层`this`       | `undefined`          | `global`（在Node.js中）      |
| Tree Shaking支持 | 支持（便于编译优化） | 不支持                       |

### 示例

假设我们有一个`math.js`文件，包含以下代码：

```javascript
// math.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export default function multiply(a, b) {
  return a * b;
}
```

使用ESM导入该模块：

```javascript
// app.js
import { add, subtract } from "./math.js";
import multiply from "./math.js";

console.log(add(2, 3)); // 输出5
console.log(subtract(5, 3)); // 输出2
console.log(multiply(4, 5)); // 输出20
```

### 总结

ESM模块系统在JavaScript生态系统中逐渐成为主流，因其具有静态分析、异步加载、Tree Shaking支持等优势，成为了现代JavaScript开发中的重要模块化工具。

## module.exports和exports的区别

在Node.js中，`module.exports`和`exports`都是用于模块导出的对象，但它们之间有一些细微的区别。理解这些区别可以帮助你避免一些常见的错误。

### 1. **默认引用关系**

- `exports`和`module.exports`在模块开始时是指向同一个对象的，也就是说，`exports`是`module.exports`的引用。
- 例如，默认情况下它们等价于：
  ```javascript
  const exports = (module.exports = {});
  ```

### 2. **导出整个对象**

- `module.exports`是真正导出的对象，`require`的返回值最终是`module.exports`的值。
- 如果想要导出一个新的对象或函数，应直接赋值给`module.exports`，而不是`exports`，否则不会生效。
- 例如：
  ```javascript
  module.exports = {
    foo: "bar",
  };
  // 或者导出函数
  module.exports = function () {
    console.log("Hello");
  };
  ```
- 错误示例，如果直接修改`exports`，`module.exports`不会受影响：
  ```javascript
  exports = {
    foo: "bar",
  };
  // require时返回的是一个空对象，而不是{ foo: 'bar' }
  ```

### 3. **添加属性或方法**

- 如果只是想给模块添加一些属性或方法，可以直接在`exports`上添加属性，因为`exports`是`module.exports`的引用。
- 例如，以下两种写法都可以：

  ```javascript
  // 方法一：使用 exports
  exports.foo = "bar";

  // 方法二：使用 module.exports
  module.exports.foo = "bar";
  ```

### 4. **覆盖 vs. 扩展**

- 当需要**覆盖整个导出对象**时，必须使用`module.exports`。
- 而当只是想在现有的导出对象上**添加属性或方法**时，可以使用`exports`或`module.exports`，效果相同。

### 例子比较

假设有一个模块`myModule.js`：

```javascript
// 错误写法
exports = { foo: "bar" }; // 此时 `exports` 不再指向 `module.exports`，不会生效

// 正确写法
module.exports = { foo: "bar" };

// 添加属性的正确写法
exports.foo = "bar"; // 或者 module.exports.foo = 'bar';
```

### 总结

- `module.exports`是真正的导出对象。
- `exports`只是`module.exports`的引用，主要用于辅助导出属性或方法。
- 覆盖整个导出对象时使用`module.exports`；仅添加属性或方法时两者皆可。

