---
title: 前端基础之JS原型链
date: 2018-09-07 15:41:08
tags:
  - 前端
  - 原型链
---

## 原型链简介

关于 `JS` 中的原型链，以及 `__proto__` 与 `prototype` 之间的关系，直接看一张图：

<!-- more -->

![prototype](/assets/img/prototype.jpg)

举个例子来说：

```js
function Person() {}

var person = new Person()

console.log(person.__proto__ === Person.prototype) //true

console.log(Person.prototype.constructor === Person) //true

console.log(Person.__proto__ === Function.prototype) //true

console.log(Function.prototype.__proto__ === Object.prototype) //true

console.log(Function.__proto__ === Function.prototype) //true

console.log(Object.prototype.__proto__ === null) //true

console.log(Object.__proto__ === Function.prototype) //true
```

总结：

1. 一个对象的 `__proto__` 指向它构造函数的 `prototype`(有特殊情况)。
2. `__proto__` 的末尾是 `Object.prototype.__proto__`，指向 `null`。
3. 原型的构造函数就是对象的构造函数。

> 特殊情况：对象由 `Object.create` 函数创建：

    ```js
    var person1 = {
      name: 'person1'
    }
    var person2 = Object.create(person1)
    console.log(person2.__proto__ === person1) //true
    ```

## 属性查找

当我们读取一个属性的时候，如果在实例属性上找到了，就读取它，不会管原型属性上是否还有相同的属性，这其实就是属性屏蔽。

但是如果在实例属性上没有找到的话，就会在实例的原型上去找，如果原型上还没有，就继续到原型的原型上去找，直到尽头(`Object.prototype`)。

如何检测一个属性存在于实例中，还是原型中？

使用方法 `hasOwnProperty`,属性只有存在于实例中才会返回 `true`:

```js
function Person() {}
Person.prototype.prototypeName = 'prototype name'
var person1 = new Person()
// 实例属性
person1.name = 'J'
person1.hasOwnProperty('name') // true
person1.hasOwnProperty('prototypeName')//false
```

而 `in` 操作符和 `Object.keys()` 都会返回所有属性，包括原型链上的属性。
