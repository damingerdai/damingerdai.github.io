# TypeScript中可选链

## 什么是可选链

TypeScript 3.7中一个最引人关注的特性便是可选链(Optional Chaining)。

所谓可选链，就是当我们试图使用访问对象的字段或者方法时，如果对象为`null`或者`undefined`，TypeScript将会自动停止运行的代码，以防止空指针异常。

## 可选链的使用

首先定义一个接口 A, 有一个字段b，b可能是字符串，也可能是null:

```TypeScript
interface A {
    b: string | null
}
```

定义一个变量a:

```TypeScript
const a = {
    b: 'c'
}
```

使用`?.`来访问字段:

```TypeScript
console.log(a?.b)
```

如果a为`null`或者`undefined`, 将输出`undefined`

## TypeScript做了什么

可选链并不是ts的专利，在js就已经存在了，除了ie,其他现代浏览器最新版本都是支持了，但是一些老版本就不支持，因此ts没有使用js的语法，而是通过三元表达式`? :`转译。因此上面的代码将会转译成:

```JavaScript
console.log(a === null || a === void 0 ? void 0 : a.b);
```

但是ts没有判断a是否是`undefined`, 而是因为在js中， `undefined`不仅是值，也可能是一个全局变量。直到es5之前，`undefined`是可以被修改的。我们可以通过下面的代码来修改:
```JavaScript
(function() {
  var undefined = 1

  console.log(undefined) // 1
})()
```

## 参考

1. [深入理解 TypeScript(可选链（Optional Chaining）)](https://jkchao.github.io/typescript-book-chinese/new/typescript-3.7.html#%E5%8F%AF%E9%80%89%E9%93%BE%EF%BC%88optional-chaining%EF%BC%89)
2. [JS: The difference between "undefined", "null" and "void 0"](https://fodor.org/blog/js-undefined-null-void/)
3. [可选链操作符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Optional_chaining)