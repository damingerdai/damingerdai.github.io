---
title: JavaScript中的闭包
date: 2024-10-20 14:25:45
tags: [javascript]
categories: [前端]
---

## 定义

闭包（closure）是一个函数以及其捆绑的周边环境状态（lexical environment，词法环境）的引用的组合。换而言之，闭包让开发者可以从内部函数访问外部函数的作用域。在 JavaScript 中，闭包就是能够读取其他函数内部变量的, 闭包会随着函数的创建而被同时创建。


## 作用

> 本部分例子来自chatgpt

1. 创建私有变量

```javascript
function createCounter() {
    let count = 0;
    return function() {
        count++;
        return count;
    };
}

const counter = createCounter();
console.log(counter()); // 输出 1
console.log(counter());
```

2. 保持函数执行上下文

```javascript
function outer() {
    let name = "John";
    return function() {
        console.log(name);
    };
}

const inner = outer();
inner(); // 输出 "John"

```

3. 函数柯里化

```javascript
function add(a) {
    return function(b) {
        return a + b;
    };
}

const addFive = add(5);
console.log(addFive(10)); // 输出 15

```

4. 延迟执行和异步编程

闭包常用于延迟执行和处理异步操作，因为它能够保存外部函数中的状态。

```javascript
function delayedMessage(message, delay) {
    setTimeout(function() {
        console.log(message);
    }, delay);
}
```

5. 模拟模块

```javascript
const module = (function() {
    let privateVar = "I'm private";
    function privateMethod() {
        console.log(privateVar);
    }
    return {
        publicMethod: function() {
            privateMethod();
        }
    };
})();
module.publicMethod(); // 输出 "I'm private"
```

总结来说，闭包在 JavaScript 中用于数据的封装、状态的保持、异步编程以及提高代码的灵活性与模块化。

## 思考题

如果你能理解下面两段代码的运行结果，应该就算理解闭包的运行机制了。

代码片段一：

```javascript
var name = "The Window";

var object = {
　　name : "My Object",

　　getNameFunc : function(){
　　　　return function(){
　　　　　　return this.name;
　　　　};

　　}

};

alert(object.getNameFunc()())

```

代码二：

```javascript
var name = "The Window";

var object = {
　　name : "My Object",

　　getNameFunc : function(){
        that = this;
　　　  return function(){
　　　　　  return that.name;
　　　  };

　　}

};

alert(object.getNameFunc()());
```


## 参考资料

1. [闭包](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Closures)
2. [学习Javascript闭包（Closure）](https://www.ruanyifeng.com/blog/2009/08/learning_javascript_closures.html)