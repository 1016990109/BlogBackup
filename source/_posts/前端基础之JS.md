---
title: 前端基础之JS
date: 2018-09-08 09:44:06
tags:
  - 前端
  - JS
---

## JavaScript 中的 this

`JS` 中的 `this` 是一个相对复杂的概念，不是简单几句能解释清楚的。粗略地讲，函数的调用方式决定了 `this` 的值。`this` 取值符合以下规则：

<!-- more -->

1. 在调用函数时使用 `new` 关键字，函数内的 `this` 是一个全新的对象。
2. 如果 `apply`、`call` 或 `bind` 方法用于调用、创建一个函数，函数内的 `this` 就是作为参数传入这些方法的对象。
3. 当函数作为对象里的方法被调用时，函数内的 `this` 是调用该函数的对象。比如当 `obj.method()` 被调用时，函数内的 `this` 将绑定到 `obj` 对象。
4. 如果调用函数不符合上述规则，那么 `this` 的值指向全局对象（`global object`）。浏览器环境下 `this` 的值指向 `window` 对象，但是在严格模式下(`'use strict'`)，`this` 的值为 `undefined`。
5. 如果符合上述多个规则，则较高的规则（1 号最高，4 号最低）将决定 `this` 的值。
6. 如果该函数是 `ES2015` 中的箭头函数，将忽略上面的所有规则，`this` 被设置为它被创建时的上下文。

## IIFE

`IIFE`(Immediately Invoked Function Expressions)代表立即执行函数。 `JavaScript` 解析器将 `function foo(){ }();` 解析成 `function foo(){ }` 和 `();`。其中，前者是函数声明；后者（一对括号）是试图调用一个函数，却没有指定名称，因此它会抛出 `Uncaught SyntaxError: Unexpected token` 的错误。修改方法：

`(function foo(){ })()` 和 `(function foo(){ }())`。

可能会用到 `void` 操作符：`void function foo(){ }();`，但是返回值是 `undefined`。

```js
// Don't add JS syntax to this code block to prevent Prettier from formatting it.
const foo = void (function bar() {
  return 'foo'
})()

console.log(foo) // undefined
```

## null、undefined 和未声明变量

当你没有提前使用 `var`、`let` 或 `const` 声明变量，就为一个变量赋值时，该变量是未声明变量（`undeclared variables`）。未声明变量会脱离当前作用域，成为全局作用域下定义的变量。在严格模式下，给未声明的变量赋值，会抛出 `ReferenceError` 错误。

```js
function foo() {
  x = 1 // 在严格模式下，抛出 ReferenceError 错误
}

foo()
console.log(x) // 1
```

> 注意 `null == undefined`，但是 `null !== undefined`。

## 宿主对象（host objects）和原生对象（native objects）

原生对象是由 `ECMAScript` 规范定义的 `JavaScript` 内置对象，比如 `String`、`Math`、`RegExp`、`Object`、`Function` 等等。

宿主对象是由运行时环境（浏览器或 `Node`）提供，比如 `window`、`XMLHTTPRequest` 等等。

## 判断数据类型

一般有以下一种方式：

- `typeof`，返回对象的基础数据类型(`boolean`,`number`,`string`,`object`,`undefined`,`function`, `es6` 的 `symbol`)是何种，小写。
- `instanceof`，一般用来判断引用类型，不是所有浏览器都支持这个语法。
- `Object.prototype.toString.call(object)`，**通用的方法**，返回 `[object + 类型]`，这里的类型首字母大写，如 `Object`。

## document.write

`document.write()` 接收一个字符串作为参数，将该字符串写入文档流中。一旦文档流已经关闭（`document.close()`），那么 `document.write` 就会重新利用 `document.open()` 打开新的文档流并写入，此时原来的文档流会被清空，已渲染好的页面就会被清除，浏览器将重新构建 `DOM` 并渲染新的页面。

> 实际生产中，要尽量避免使用 `document.write`。

## 功能检测（feature detection）、功能推断（feature inference）和 UA 字符串

### 功能检测

功能检测包括确定浏览器是否支持某段代码，以及是否运行不同的代码（取决于它是否执行），以便浏览器始终能够正常运行代码功能，而不会在某些浏览器中出现崩溃和错误。例如：

```js
if ('geolocation' in navigator) {
  // 可以使用 navigator.geolocation
} else {
  // 处理 navigator.geolocation 功能缺失
}
```

### 功能推断

功能推断与功能检测一样，会对功能可用性进行检查，但是在判断通过后，还会使用其他功能，因为它假设其他功能也可用，例如：

```js
if (document.getElementsByTagName) {
  element = document.getElementById(id)
}
```

非常不推荐这种方式。功能检测更能保证万无一失。

### UA 字符串

这是一个浏览器报告的字符串，它允许网络协议对等方（`network protocol peers`）识别请求用户代理的应用类型、操作系统、应用供应商和应用版本。它可以通过 `navigator.userAgent` 访问。 然而，这个字符串很难解析并且很可能存在欺骗性。例如，`Chrome` 会同时作为 `Chrome` 和 `Safari` 进行报告。因此，要检测 `Safari`，除了检查 `Safari` 字符串，还要检查是否存在 `Chrome` 字符串。不要使用这种方式。

## 变量提升

变量提升（`hoisting`）是用于解释代码中变量声明行为的术语。**使用 `var`(`let` 是没用的) 关键字声明或初始化的变量**，会将声明语句“提升”到当前作用域的顶部。 但是，**只有声明**才会触发提升，赋值语句（如果有的话）将保持原样。我们用几个例子来解释一下。

```js
// 用 var 声明得到提升
console.log(foo) // undefined
var foo = 1
console.log(foo) // 1

// 用 let/const 声明不会提升
console.log(bar) // ReferenceError: bar is not defined
let bar = 2
console.log(bar) // 2
```

函数声明会使函数体提升，但函数表达式（以声明变量的形式书写）只有变量声明会被提升。

```js
// 函数声明
console.log(foo) // [Function: foo]
foo() // 'FOOOOO'
function foo() {
  console.log('FOOOOO')
}
console.log(foo) // [Function: foo]

// 函数表达式
console.log(bar) // undefined
bar() // Uncaught TypeError: bar is not a function
var bar = function() {
  console.log('BARRRR')
}
console.log(bar) // [Function: bar]
```

## attribute 和 property

`Attribute` 是在 `HTML` 中定义的，而 `property` 是在 `DOM` 上定义的。为了说明区别，假设我们在 `HTML` 中有一个文本框：`<input type="text" value="Hello">`。**`Attribute` 只有通过初始 `HTML` 中设置或者 `setAttribute` 方法设置才能改变**。

```js
const input = document.querySelector('input')
console.log(input.getAttribute('value')) // Hello
console.log(input.value) // Hello
```

但是在文本框中键入 “ World!”后:

```js
console.log(input.getAttribute('value')) // Hello
console.log(input.value) // Hello World!
```

## load 事件和 DOMContentLoaded 事件

当初始的 `HTML` 文档被完全加载和解析完成之后，`DOMContentLoaded` 事件被触发，而无需等待样式表、图像和子框架的完成加载。

`window` 的 `load` 事件仅在 `DOM` 和所有相关资源全部完成加载后才会触发。

## 严格模式

在文件、脚本、函数的开头加上 `'use strict'` 来启用严格模式。

严格模式特点：

1. 全局变量需要显示声明，直接 `x = 1` 会报错。
2. 禁止使用 `with` 语句，创建 `eval` 的作用域。
3. 禁止 `this` 指向全局对象。
4. 禁止在函数内部遍历调用栈，`fn.caller`、`fn.arguments` 都会报错，注意直接使用 `arguments` 是不会报错的。
5. 禁止删除变量，除非 `configurable` 属性为 `true`。
6. 对象不能有重名属性，函数不能有重名参数。
7. 显示报错，比如删除不可删除变量、对只读属性赋值都会显示报错。
8. 禁止八进制表示法。
9. 不允许对 `arguments` 赋值，不跟踪 `arguments` 的变化，禁止使用 `arguments.callee`。
10. 函数必须声明在顶层，也就是说不能在非函数的代码块中声明函数。
11. 新增保留字，和 `ES6` 接轨，如 `implements`,`let`,`static` 等等。

## 柯里化(curry)

柯里化（`currying`）是一种模式，其中具有多个参数的函数被分解为多个函数，当被串联调用时，将一次一个地累积所有需要的参数。

```js
function curry(fn) {
  //保留参数合并结果
  let args = []

  let curried = function() {
    if (arguments.length === 0) {
      return fn.apply(this, args)
    } else {
      args.push(...arguments)
      return curried
    }
  }

  return curried
}

let addCurry = curry(function() {
  let sum = 0
  for (let i in arguments) {
    sum += arguments[i]
  }
  return sum
})

console.log(addCurry(1)(2)(3)(4)())//10
```
