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

可以看到第二个 console.log 输出的是“Alan Turing”，这是因为调用 set 后默认返回 get 获得的值。

### Map and Set collections(Map 和 Set 集合)

原来我们建立 hash map 的时候都是用`object`来完成的，而现在可以直接使用`Map`原型，提供了 set、get、has、delete 方法和 size 属性，比使用`object`更加直接、简单，遍历可使用`for...of`语法，这种遍历方式是和 Map 中属性的插入顺序是一样的(这在普通`object`中是无法保证的)。

```js
const tests = new Map();
   tests.set(() => 2+2, 4);
   tests.set(() => 2*2, 4);
   tests.set(() => 2/2, 1);
   for (const entry of tests) {
     console.log((entry[0]() === entry[1])   'PASS' : 'FAIL');
}
```

`Set`只允许存在不同的值，和数学上的集合是一个概念，里面内容可以是`number`也可以是`object`或`function`，除了 set 换成 add 外其他的都与 map 相同，遍历时每一个 entry 内容是 value。

### WeakMap and WeakSet collections

顾名思义，`WeakMap`和`WeakSet`是`Map`和`Set`弱化后的原型，但是这其中并无优劣之分，只是适用于不同的场合。

`WeakMap`的**key**只能是非空对象，对**key**仅保持弱引用，最大的好处是可以避免内存泄漏，一旦**key**的引用为空或者 undefined，垃圾回收器就可以回收这个对象，但是`WeakMap`不能迭代遍历。

`WeakSet`与`WeakMap`同。

### Template literals(模板字符串)

使用`代替双引号和单引号，在字符串中可以使用表达式${expression}，可以换行。

### ES6 其他语法

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

`reactor`模式是 `Node.js` 异步的核心。

### I/O is slow(I/O 操作是慢的)

I/O 操作可以说是计算机操作中最慢的一环，I/O 的速度可能和网络速度、磁盘速率有关，也可能和其他因素有关，比如用户点击事件等等。

### Blocking I/O(阻塞 I/O)

传统的阻塞 I/O 模型中，I/O 请求会阻塞之后代码块的运行，例如：

```js
// 直到请求完成，数据可用，线程都是阻塞的
data = socket.read()
// 请求完成，数据可用
print(data)
```

而为了达到并发的  目的，传统的 web 服务器是  选择新开一个线程或进程，这样因为线程(或进程)之间的相互独立性，一个线程(或进程)阻塞并不会影响另一个。

但是创建一个线程是昂贵的，一个线程需要内存，而且切换线程需要保留线程的上下文等等，所以这种方式并不是最佳实践。

### Non-blocking I/O(非阻塞 I/O)

与阻塞 I/O 相反，遇到 I/O 请求不会阻塞后续代码的执行，如果访问的资源不可用则会返回一个预定义的常量值。

非阻塞 I/O 最基本的模式是轮询直到有数据已经返回了，也叫做 `忙等待`模式：

```js
resources = [socketA, socketB, pipeA]
while (!resources.isEmpty()) {
  for (i = 0; i < resources.length; i++) {
    resource = resources[i]
    // 进行读操作
    let data = resource.read()
    if (data === NO_DATA_AVAILABLE) {
      // 此时还没有数据
      continue
    }
    if (data === RESOURCE_CLOSED) {
      // 资源被释放，从队列中移除该链接
      resources.remove(i)
    } else {
      consumeData(data)
    }
  }
}
```

这个例子已经能有单线程处理多个请求了， 但是不够高效，资源不可用时循环占了太多了 CPU 时间，轮询算法浪费 CPU 时间。

### Event demultiplexing(事件多路复用)

对于获取非阻塞的资源而言，忙等待模型不是一个理想的技术，大多数现代的操作系统都提供了一种机制来处理并发和非阻塞资源，这个机制被称为`同步多路复用`。

这个组件从一系列被监听的资源中收集 I/O 事件并放入队列中，而且会一直处于阻塞状态直到有新的事件可以被处理：

```js
socketA, pipeB;
wachedList.add(socketA, FOR_READ);
wachedList.add(pipeB, FOR_READ);
while(events = demultiplexer.watch(wachedList)) {
  // 事件循环
  foreach(event in events) {
    // 永远不会阻塞，并且总会有返回值
    data = event.resource.read();
    if (data === RESOURCE_CLOSED) {
      // 资源已经被释放，从观察者队列移除
      demultiplexer.unwatch(event.resource);
    } else {
      // 获得数据进行处理
      consumeData(data);
    }
  }
}
```

代码的三个重要步骤：

1.  资源被添加到一个数据结构中，为每个资源关联一个特定的操作，在这个例子中是 read。
2.  事件通知器由一组被观察的资源组成，事件通知器是同步和阻塞的直到有资源可以被`read`，事件触发后会从调用中返回，之后这些事件可以被处理。
3.  多路复用器返回的每个事件被处理，此时，和事件相关的资源都可用且不会在操作中阻塞。当所有的事件都被处理完后，继续进入循环等待下一个可以被处理的事件。这个被称作为`事件循环(event loop)`。

![多路复用](/assets/img/node_demultiplexer.png)

上图帮助我们理解如何在一个单线程中使用多路复用器和非阻塞 I/O 来处理并发。我们能够看到，只使用一个线程并不会影响我们处理多个 I/O 任务的性能。同时，我们看到任务是在单个线程中随着时间的推移而展开的，而不是分散在多个线程中。我们看到，在单线程中传播的任务相对于多线程中传播的任务反而节约了线程的总体空闲时间，并且更利于程序员编写代码。

### Introducing to reactor pattern(reactor 模式的介绍)

主要思想就是每一个 I/O 操作都有一个`handler`或者成为回调函数(`callback`)，当事件发生并且被`事件循环`处理后，这个回调函数就会被调用：

![event loop](/assets/img/event_loop.png)

一个应用使用`reactor`模式后：

1.  应用提交一个请求给事件多路复用器 ，生成 I/O 操作，同时提供事件触发时的`handler`， 发送请求给事件多路复用器是一个非阻塞的操作，发送后立即返回到应用。
2.  当一组 I/O 操作完成，事件多路复用器会将新来的事件添加到事件队列中。
3.  此时，事件循环会迭代事件队列中的每个事件。
4.  对于每个事件，对应的`handler`被处理。
5.  `handler`，是应用程序代码的一部分，`handler`执行结束后执行权会交回事件循环。但是，在`handler`执行时可能请求新的异步操作，从而新的操作被添加到事件多路复用器。
6.  当事件队列的全部事件被处理完后，事件多路复用器再次阻塞直到有一个新的事件触发。

现在来定义 Node.js 的核心模式：  
`模式(reactor)`这样处理 I/O，阻塞直到有新的事件从被观察的资源中触发，然后将事件派发给相应的`handler`。

### Node.js 非阻塞 I/O 引擎——libuv

每个操作系统都有不同的接口来实现事件多路复用器，Linux 是 epoll，Mac OSX 是 kqueue，Windows 的 IOCP API，即使是在相同的操作系统中对于不同资源的 I/O 操作也不同，所以 Node.js 使用`libuv`来统一处理 I/O 操作，来达到兼容不同操作系统的目的。

### Node.js 架构

![Node.js 架构](/assets/img/node_architecture.png)
