---
title: 《Node.js 设计模式》读书笔记 第一章
date: 2018-05-11 20:00:45
tags:
    - 读书笔记
---

# Welcome to the Node.js Platform

## Small modules(小模块)

`Node.js`使用`module`(模块)的概念组织代码的结构。  
`package`可提供复用的模块，有一个 module 作为入口。  
`Node.js`中，致力于设计小模块，为了代码的简洁，更为了更好地控制作用域。有两个主要原则：

<!-- more -->

> * "Small is beautiful."(小而精)
> * "Make each program do one thing well." (每个程序只有单一的职责)

`Node.js` 通过官方包管理工具`npm`解决包之间的依赖问题，每个`package`都有它自己的依赖，故而一个程序中多个`package`能够无冲突地安装。应用程序都是由一个个很小的、单职责的依赖构成的。

**小模块**应该有的特性：

> * Easier to understand and use(易理解、易用)
> * Simpler to test and maintain(易于测试和维护)
> * Perfect to share with the browser(完美支持浏览器)

**DRY(Dont't Repeat Yourself)原则**

## Small surface area(暴露需要的接口)

一般使用者只会用到很有限的功能，而很少去  扩展一个模块，所以`Node.js`的很多模块只会暴露一个函数或者一个构造器，然后把更细节的东西都放在函数或者构造器里，这样能帮助  使用者认清什么是主要的什么是次要的.

模块是不允许被扩展的，看起来扩展性低，但实际上有很多优势：减少了应用场景(这样考虑的情况就少了，容易实现)，简化了实现，便于维护，提高了可用性。

## Simplicity and progmatism(简单而实用)

> 简单就是复杂到极致。—— 达尔文

> 设计必须简单， 不管是实现还是接口。实现的简洁比接口的简洁更重要。设计中最重要的就是简洁。—— Richard P.Gabriel(一位杰出的计算机科学家)

设计简单的而不是完美或功能完备的软件是一个好的实践：

*  更容易实现
* 更少的资源，传输更快
* 更容易适应
* 容易维护和理解

设计简单这个原则同样也适用于`JavaScript`，简单函数、闭包、`object`替代了复杂的类继承。

## Introduction to Node.js 6 and ES2015 (介绍 Node.js 6 和 ES6)

### The let and const keywords(let 和 const 关键字)

在之前(ES2015 之前)，`js`只支持函数作用域和全局作用域，例如在 if 中声明一个变量却能在 if 块之外访问：

```js
if (false) {
  var x = 'hello'
}

console.log(x) //undefined
```

但是使用`let`关键字后，if 块外就访问不到其中声明的变量了，这在一定程度上能减少因为误操作其中的变量而导致的 bug：

```js
if (false) {
  let x = 'hello'
}

console.log(x) //ReferenceError: x is not defined
```

`const`关键字用于声明不可变变量：

```js
const x = 'This will never change'
x = '...'
//TypeError: Assignment to constant variable
```

需要注意的是，`const`  是意味着变量的绑定不变而不是内容不变，示例入下：

```js
cosnt x = {}
x.name = 'John'//work

x = null//don't work
```

通常来说引入模块使用 `const`防止模块发生变化：

```js
const path = require('path')

let path = './some/path' //fail
```

如果你想要创建一个不可更改的对象，`const`是不够的，你可以使用 ES5 的[Object.freeze()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze)或者 [deep-freeze](https://www.npmjs.com/package/deep-freeze)模块，或者我使用`react`框架时候经常用的[immutable](https://www.npmjs.com/package/immutable)模块也可以。

> 扩展——这里提一下**ES5**中`freeze`和`seal`的区别，`seal`只限制无法增加和删除对象属性 ，而`freeze`在`seal`的基础上还限制了不可更改对象的属性。

### The arrow function(箭头函数)

箭头函数是**ES6**的一大亮点，能很大程度上简化代码。一个参数可以不需要圆括号，函数体只有一行且结果为返回值可不需要花括号，具体示例如下：

```js
const numbers = [2, 6, 7, 8, 1]
const event = numbers.filter(x => x % 2 === 0)

const event2 = numbers.filter(x => {
  if (x % 2 === 0) {
    return true
  }
})
```

箭头函数中 this 的指向跟随父函数，示例：

```js
function DelayedGreeter(name) {
  this.name = name
}

DelayedGreeter.prototype.greet = function() {
  setTimeout(function cb() {
    console.log('Hello ' + this.name)
  })
}

new DelayedGreeter('World').greet() //Hello undefined

DelayedGreeter.prototype.greet = function() {
  setTimeout(() => console.log('Hello ' + this.name))
}

new DelayedGreeter('World').greet() //Hello World
```

### Class syntax(Class 语法)

`class`只是个语法糖，使用 class 实现对象继承并不是通过`class`继承的，还是通过内部的 prototypes，properties 实现继承，但是`class`使得程序可读性变强了。

让我们来看个例子：

```js
//复杂，晦涩难懂
function Person(name, surname, age) {
  this.name = name
  this.surname = surname
  this.age = age
}

Person.prototype.getFullName = function() {
  return this.name + ' ' + this.surname
}

Person.older = function(person1, person2) {
  return preson1.age >= person2.age ? person1 : person2
}

//易懂
class Person {
  constructor(name, surname, age) {
    this.name = name
    this.surname = surname
    this.age = age
  }

  getFullName() {
    return this.name + ' ' + this.surname
  }

  static older(person1, person2) {
    return preson1.age >= person2.age ? person1 : person2
  }
}

class PersonWithMiddlename extends Person {
  constructor(name, middlename, surname, age) {
    super(name, surname, age)
    this.middlename = middlename
  }

  getFullName() {
    return this.name + ' ' + this.middlename + ' ' + this.surname
  }
}
```

### Enhanced object literals(对象字面量语法增强)

* 缺省键值，属性名和变量名相同时可省略属性名

```js
const x = 22
const y = 17
const obj = { x, y }
```

* 计算属性，属性可以是由变量计算而来

```js
const namespace = '-webkit-'
const style = {
  [namespace + 'box-sizing']: 'border-box',
  [namespace + 'box-shadow']: '10px 10px 5px #888888'
}
```

* getter 和 setter

先看例子：

```js
const person = {
  name: 'George',
  surname: 'Boole',
  get fullname() {
    return this.name + '' + this.surname
  },
  set fullname(fullname) {
    let parts = fullname.split('')
    this.name = parts[0]
    this.surname = parts[1]
  }
}
console.log(person.fullname) // "George Boole"
console.log((person.fullname = 'Alan Turing')) // "Alan Turing"
console.log(person.name) // "Alan"
```

可以看到第二个console.log输出的是“Alan Turing”，这是因为调用set后默认返回get获得的值。

### Map and Set collections(Map和Set集合)

原来我们建立hash map的时候都是用`object`来完成的，而现在可以直接使用`Map`原型，提供了set、get、has、delete方法和size属性，比使用`object`更加直接、简单，遍历可使用`for...of`语法，这种遍历方式是和Map中属性的插入顺序是一样的(这在普通`object`中是无法保证的)。

```js
const tests = new Map();
   tests.set(() => 2+2, 4);
   tests.set(() => 2*2, 4);
   tests.set(() => 2/2, 1);
   for (const entry of tests) {
     console.log((entry[0]() === entry[1])   'PASS' : 'FAIL');
}
```

`Set`只允许存在不同的值，和数学上的集合是一个概念，里面内容可以是`number`也可以是`object`或`function`，除了set换成add外其他的都与map相同，遍历时每一个entry内容是value。

### WeakMap and WeakSet collections

顾名思义，`WeakMap`和`WeakSet`是`Map`和`Set`弱化后的原型，但是这其中并无优劣之分，只是适用于不同的场合。

`WeakMap`的**key**只能是非空对象，对**key**仅保持弱引用，最大的好处是可以避免内存泄漏，一旦**key**的引用为空或者undefined，垃圾回收器就可以回收这个对象，但是`WeakMap`不能迭代遍历。

`WeakSet`与`WeakMap`同。

### Template literals(模板字符串)

使用`代替双引号和单引号，在字符串中可以使用表达式${expression}，可以换行。

### ES6其他语法

* [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)(稍后会详细讲到)
* [函数默认参数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters)
* [剩余参数语法](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Rest_parameters)
* [拓展运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator)
* [解构赋值](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)
* [new.target](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new.target)
* [代理](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
* [反射](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect)
* [Symbol](https://developer.mozilla.org/en-US/docs/Glossary/Symbol)

## The reactor pattern

---

未完待续
