---
title: 前端基础之JS事件循环
date: 2018-08-19 23:22:38
tags:
  - 前端
  - eventloop
  - 事件循环
---

## 背景

`JavaScript` 从诞生之日起就是一门单线程的非阻塞的脚本语言

单线程是必要的，也是 `JavaScript` 这门语言的基石，原因之一在其最初也是最主要的执行环境——浏览器中，我们需要进行各种各样的 `DOM` 操作。试想一下 如果 `JavaScript` 是多线程的，那么当两个线程同时对 `DOM` 进行一项操作，例如一个向其添加事件，而另一个删除了这个 `DOM`，此时该如何处理呢？因此，为了保证不会 发生类似于这个例子中的情景，`JavaScript` 选择只用一个主线程来执行代码，这样就保证了程序执行的一致性。

<!-- more -->

为了利用多核 `CPU` 的计算能力，`HTML5` 提出 `Web Worker` 标准，允许 `JavaScript` 脚本创建多个线程，但是子线程完全受主线程控制，且不得操作 `DOM`。所以，这个新标准并没有改变 `JavaScript` 单线程的本质。

## 浏览器 JS 引擎事件循环

### 任务队列

主线程完全可以不管 `IO` 设备，挂起处于等待中的任务，先运行排在后面的任务。等到 `IO` 设备返回了结果，再回过头，把挂起的任务继续执行下去。

于是，所有任务可以分成两种，一种是同步任务（`synchronous`），另一种是异步任务（`asynchronous`）。同步任务指的是，在主线程上排队执行的任务，只有前一个任务执行完毕，才能执行后一个任务；异步任务指的是，不进入主线程、而进入"任务队列"（`task queue`）的任务，只有"任务队列"通知主线程，某个异步任务可以执行了，该任务才会进入主线程执行。

运行机制如下：

（1）所有同步任务都在主线程上执行，形成一个执行栈（`execution context stack`）。
（2）主线程之外，还存在一个"任务队列"（`task queue`）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。
（3）一旦"执行栈"**所有**同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。
（4）主线程不断重复上面的第三步。

> "任务队列"是一个事件的队列（也可以理解成消息的队列），`IO` 设备完成一项任务，就在"任务队列"中添加一个事件，表示相关的异步任务可以进入"执行栈"了。主线程读取"任务队列"，就是读取里面有哪些事件。除了 `IO` 设备事件，用户点击事件等也是会进入"任务队列"的。"任务队列"是先进先出的，主线程的读取过程基本上是自动的，只要执行栈一清空，"任务队列"上第一位的事件就自动进入主线程。

### macro task(宏任务) 与 micro task(微任务)

异步任务之间也有执行的优先级，不同的异步任务被分为两类：微任务（`micro task`）和宏任务（`macro task`）。

以下事件属于宏任务：

- setInterval()
- setTimeout()

以下事件属于微任务：

- new Promise()
- new MutaionObserver()

在一个事件循环中，异步事件返回结果后会被放到一个任务队列中。然而，根据这个异步事件的类型，这个事件实际上会被对应的宏任务队列或者微任务队列中去。并且在当前执行栈为空的时候，主线程会查看微任务队列是否有事件存在。

**当当前执行栈执行完毕时会立刻先处理所有微任务队列中的事件，然后再去宏任务队列中取出一个事件。同一次事件循环中，微任务永远在宏任务之前执行。**

**同类型异步任务按进入的先后顺序依次触发。**

## Node 环境下的事件循环

`Node.js` 也是单线程的 `Event Loop`，但是它的运行机制不同于浏览器环境。

`Node` 中事件循环的实现是依靠的 `libuv` 引擎。我们知道 `Node` 选择 `Chrome v8` 引擎作为 `js` 解释器，`v8` 引擎将 `js` 代码分析后去调用对应的 `Node API`，而这些 `API` 最后则由 `libuv` 引擎驱动，执行对应的任务，并把不同的事件放在不同的队列中等待主线程执行。 因此实际上 `Node` 中的事件循环存在于 `libuv` 引擎中。

我们来看看 `libuv` 的事件循环模型：

![node event loop](/assets/img/node_event_loop.png)

> `timers`，一个 `timer` 指定一个下限时间而不是准确时间，在达到这个下限时间后执行回调。在指定的时间过后，`timers` 会尽早的执行回调，但是系统调度或者其他回调的执行可能会延迟它们。下限的时间有一个范围：[1, 2147483647]，如果设定的时间不在这个范围，将被设置为 1。

> `setImmediate()` 具有最高优先级，只要 `poll` 队列为空，代码被 `setImmediate()`，无论是否有 `timers` 达到下限时间，`setImmediate()`的代码都先执行。

我们可以大致分析出 `Node` 中的事件循环的顺序：

外部输入数据-->轮询阶段(`poll`)-->检查阶段(`check`)-->关闭事件回调阶段(`close callback`)-->定时器检测阶段(`timer`)-->`I/O` 事件回调阶段(`I/O callbacks`)-->闲置阶段(`idle`, `prepare`)-->轮询阶段...

除了 `setTimeout` 和 `setInterval` 这两个方法，`Node.js` 还提供了另外两个与"任务队列"有关的方法：`process.nextTick` 和 `setImmediate`。

执行顺序为 `process.nextTick`(单独的一个队列) --> 微任务(`Promise`，`MutaionObserver`) --> 宏任务(`setTimeout`，`setInterval`)/`setImmediate`

> 注意错误使用 `process.nextTick` 可能会进入一个死循环，而导致 `js` 主线程阻塞，而 `setTimeout(function, 0)` 不会。`process.nextTick` 在当前"执行栈"执行。

### 关于 setTimeout 和 setImmediate

`setTimeout` 和 `setImmediate` 在 `Node` 环境下执行是靠“随缘法则”的，执行先后顺序不确定。

首先进入的是 `timers` 阶段，如果我们的机器性能一般，那么进入 `timers` 阶段，一毫秒已经过去了（`setTimeout(fn, 0)`等价于 `setTimeout(fn, 1)`），那么 `setTimeout` 的回调会首先执行。

如果没有到一毫秒，那么在 `timers` 阶段的时候，下限时间没到，`setTimeout` 回调不执行，事件循环来到了 `poll` 阶段，这个时候队列为空，此时有代码被 `setImmediate()`，于是先执行了 `setImmediate()` 的回调函数，之后在下一个事件循环再执行 `setTimemout` 的回调函数。

而我们在执行代码的时候，进入 `timers` 的时间延迟其实是随机的，并不是确定的，所以会出现两个函数执行顺序随机的情况。

但是有种情况例外：

```js
var fs = require('fs')

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout')
  }, 0)
  setImmediate(() => {
    console.log('immediate')
  })
})
```

上面代码，`setImmediate` 永远优先 `setTimeout` 执行。

`fs.readFile` 的回调是在 `poll` 阶段执行的，当其回调执行完毕之后，`poll` 队列为空，而 `setTimeout` 入了 `timers` 的队列，此时有代码被 `setImmediate()`，于是事件循环先进入 `check` 阶段执行回调，之后在下一个事件循环再在 `timers` 阶段中执行有效回调。

**总结：**

- 如果两者都在主模块中调用，那么执行先后取决于进程性能，也就是随机。
- 如果两者都不在主模块调用（被一个异步操作包裹），那么 `setImmediate` 的回调永远先执行。

## 实践

为了更好地理解事件循环，可以尝试运行下面的代码，看看结果如何：

```js
setImmediate(function() {
  console.log(7)
})
setTimeout(function() {
  console.log(1)
}, 0)
process.nextTick(function() {
  console.log(6)
  process.nextTick(function() {
    console.log(8)
  })
})
new Promise(function executor(resolve) {
  console.log(2)
  for (var i = 0; i < 10000; i++) {
    i == 9999 && resolve()
  }
  console.log(3)
}).then(function() {
  console.log(4)
})
console.log(5)

//执行队列(同步) 2 3 5   6 8（6,8为nextTick队列中的)
//任务队列(异步) 4 (1,7顺序不确定)
```
