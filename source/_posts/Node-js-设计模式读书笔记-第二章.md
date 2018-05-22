---
title: 《Node.js 设计模式》读书笔记 第二章
date: 2018-05-18 16:51:57
tags:
    - 读书笔记
---

# Node.js Essential Patterns(Node.js 基本模式)

## The callback pattern(回调模式)

回调是`reactor`模式中`handler`的实例。

### The continuation-passing pattern(连续传递模式)

在 JavaScript 中回调就是传入作为参数传入另外一个函数中的函数，并且在操作完成后调用。在函数式编程中，这种传递结果的方式被称为`continuation-passing style(CPS)`。这是个一般概念，并不是针对异步操作。实际上，它只是通过将结果作为参数传递给另一个函数（回调函数）来传递结果，然后在主体逻辑中调用回调函数拿到操作结果，而不是直接将其返回给调用者。

<!-- more -->

#### Synchronous continuation-passing style(同步连续传递风格)

```js
function add(a, b, callback) {
  callback(a + b)
}
```

这个`add()`函数就是一个同步的 CPS 函数，意味着只有回调函数执行完成它才会返回值。

#### Asynchronous continuation-passing style(异步连续传递风格)

```js
function additionAsync(a, b, callback) {
  setTimeout(() => callback(a + b), 100)
}
```

`setTimeout()`触发了一个异步操作，不需要等待回调函数执行完就会返回到`additionAsync()`的控制权，然后再回到`additionAsync`的调用者。这个属性对于`Node.js`是至关重要的，当一个异步的请求发出后会立即回到事件循环中， 因而允许队列中  新的事件被处理。下图描述了事件循环是怎么运作的：

![事件循环](assets/img/event_loop2.png)

#### Non-continuation-passing style callbacks(非连续传递风格的回调)

某些情况下，比如一个回调函数作为参数传入，我们可能会以为这是一个异步操作或者是使用 CPS，但是有例外：

```js
const result = [1, 5, 7].map(element => element - 1)
console.log(result) // [0, 4, 6]
```

这就是一个同步的调用。

### Synchronous or asynchronous?(同步还是异步?)

代码的执行顺序会因同步或异步的执行方式产生根本性的改变。这对整个应用程序的流程，正确性和效率都产生了重大影响。以下是对这两种模式的范例和缺陷的分析：

#### An unpredictable function(一个不可预测的函数)

最危险的情况之一就是使一个 API 在某种特定情况下是同步执行的但是在另一种情况却是异步执行的：

```js
const fs = require('fs')
const cache = {}

function inconsistentRead(filename, callback) {
  if (cache[filename]) {
    // 同步执行回调
    callback(cache[filename])
  } else {
    // 异步
    fs.readFile(filename, 'utf8', (err, data) => {
      cache[filename] = data
      callback(data)
    })
  }
}
```

上面的函数使用 cache 缓存文件读取结果，这个函数是危险的，因为当缓存命中时表现为同步的，当缓存未命中表现为异步的。

#### Unleashing Zalgo(解放 Zalgo)

围绕着同步或异步行为的不确定性，几乎总是导致非常难追踪的 Bug，这被称为`Zalgo`。

> 注意：更多关于 Zalgo 的信息，参见 Oren Golan 的[Don't Release Zalgo!(不要释放 Zalgo!)](https://github.com/oren/oren.github.io/blob/master/posts/zalgo.md)和 Isaac Z. Schlueter 的[Designing APIs for Asynchrony(异步 API 设计)](http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony)。

看个例子：

```js
function createFileReader(filename) {
  const listeners = []
  inconsistentRead(filename, value => {
    listeners.forEach(listener => listener(value))
  })
  return {
    onDataReady: listener => listeners.push(listener)
  }
}
```

函数被调用的时候创建了一个作为通知器的对象，允许为一个  读文件操作设置多个监听。当文件读取完后或者数据可用后所有的监听都会被调用，现在使用上面的`inconsistentRead`函数来完成这个功能：

```js
const reader1 = createFileReader('data.txt')
reader1.onDataReady(data => {
  console.log('First call data: ' + data)
  // 之后再次通过fs读取同一个文件
  const reader2 = createFileReader('data.txt')
  reader2.onDataReady(data => {
    console.log('Second call data: ' + data)
  })
})
```

输出是这样的：

```js
First call data: some data
```

会发现，第二个操作的回调未被调用，这是因为第一次调用时没有缓存，`inconsistentRead`是异步调用的，这个时候第一个回调函数已经加入到监听列表中了，然而第二次调用时，缓存中已经存在了，所以`inconsistentRead`是同步调用的，这个时候监听列表中还没有第二个回调函数，故而之后也没有再调用了。

#### Using asynchronous API(使用同步 API)

从上面`unleashing Zalgo`中我们知道，清楚地定义`API`性质是非常重要的：同步还是异步？

可以使整个函数同步来解决上面的问题，`readFileSync`代替`readFile`：

```js
const fs = require('fs')
const cache = {}

function consistentReadSync(filename) {
  if (cache[filename]) {
    return cache[filename]
  } else {
    cache[filename] = fs.readFileSync(filename, 'utf8')
    return cache[filename]
  }
}
```

没有理由为一个同步函数使用`CPS`风格。

> 注意：最好为纯同步函数使用`direct style`。

使用同步 API 替代异步 API 需要注意：

* 同步 API 并不适用所有场景。
* 同步 API 会阻塞事件循环，降低了应用性能，破坏了`JavaScript`并发模型。

如果只是读取有限的文件，那么`consistentReadSync`并不会对事件循环的性能造成多大影响，但是当文件很多时就不同了。

很多时候`Node.js`是不鼓励使用同步函数的，但是有时候同步函数也是最简单、最有效的解决方案。所以要根据情况来选择使用同步的方法还是异步的。

当对程序处理并发请求影响不大的时候使用同步(例如读取程序配置文件)。

### Deffered execution(延时处理)

另外一种处理`consistentRead`的方法就是使整个函数都是异步的，使用`process.nextTick()`使得回调函数在下一个事件循环周期调用而不是立即调用：

```js
const fs = require('fs')
const cache = {}

function consistentReadAsync(filename, callback) {
  if (cache[filename]) {
    process.nextTick(() => callback(cache[filename]))
  } else {
    // 异步函数
    fs.readFile(filename, 'utf8', (err, data) => {
      cache[filename] = data
      callback(data)
    })
  }
}
```

另外一个延迟执行的 API 是`setImmediate()`，尽管他们看起来相似，但是在语义上完全不同，`process.nextTick()`在任何 I/O 事件触发前执行，而`setImmediate()`在队列中所有的 I/O 事件执行之后执行。

### Node.js callback conventions(Node.js 回调风格)

对于`Node.js`而言，`CPS`风格的 API 和回调函数遵循一组特殊的约定。这些约定不只是适用于`Node.js`核心 API，对于它们之后也是绝大多数用户级模块和应用程序也很有意义。因此，我们了解这些风格，并确保我们在需要设计异步 API 时遵守规定显得至关重要。

#### Callbacks come last(回调函数在最后)

在所有核心 Node.js 方法中，标准约定是当函数在输入中接受回调时，必须作为最后一个参数传递。我们以下面的 Node.js 核心 API 为例：

```js
fs.readFile(filename, [options], callback)
```

从前面的例子可以看出，即使是在可选参数存在的情况下，回调也始终置于最后的位置。其原因是在回调定义的情况下，函数调用更可读。

#### Error comes first(错误处理在最前)

在`Node.js`中，在`CPS`中产生的错误总是作为第一个参数，这样便于调试，没有错误则第一个参数为 null 或者 undefined，在正式处理结果之前先判断 error 这样有利于 debug，错误类型为`Error`型，普通的字符串或者数字是不应该作为错误被传递的。

```js
fs.readFile('foo.txt', 'utf8', (err, data) => {
  if (err) handleError(err)
  else processData(data)
})
```

#### Propagating errors(传递错误)

在同步函数或者直接风格中，使用`throw`是最直接的，使得错误从调用栈中弹出直到被捕获。

但是在异步调用中，比较好的错误处理方式是将错误传递到回调链中的下一个回调函数中：

```js
const fs = require('fs')

function readJSON(filename, callback) {
  fs.readFile(filename, 'utf8', (err, data) => {
    let parsed
    if (err)
      //传递错误并退出当前函数
      return callback(err)
    try {
      //解析文件内容
      parsed = JSON.parse(data)
    } catch (err) {
      //捕获解析中的错误
      return callback(err)
    }
    //没有错误，只传递数据
    callback(null, parsed)
  })
}
```

有错误直接`return`避免继续执行。

#### Uncaught exceptions

在异步回调过程中，错误是难以被捕获的，例如：

```js
const fs = require('fs');

function readJSONThrows(filename, callback) {
  fs.readFile(filename, 'utf8', (err, data) => {
    if (err) {
      return callback(err);
    }
    callback(null, JSON.parse(data));
  });
};
```

在上面的函数中，如果`JSON.parse(data)`异常的话是没有办法捕获的：

```js
try {
  readJSONThrows('nonJSON.txt', function(err, result) {
    // ... 
  });
} catch (err) {
  console.log('This will not catch the JSON parsing exception');
}
```

上面`catch`语句将捕获不到错误，因为错误是在回调函数中产生的。然而，我们仍然有机会在应用程序终止之前执行一些清理或日志记录。事实上，当这种情况发生时，Node.js会在退出进程之前发出一个名为`uncaughtException`的特殊事件：

```js
process.on('uncaughtException', (err) => {
  console.error('This will catch at last the ' +
    'JSON parsing exception: ' + err.message);
  // Terminates the application with 1 (error) as exit code:
  // without the following line, the application would continue
  process.exit(1);
});
```

需要注意的是，`uncaughtException`会使得应用处于一个不能保证一致的状态，而这可能导致不可预见的错误。比如还有未完成的I/O请求正在运行或关闭，这可能导致不一致。所以建议，尤其是在生产环境，在收到任何`uncaught exception`之后停止应用的运行。

### The module system and its patterns(模块系统和其中的模式)

模块可以隐藏不想暴露的函数、变量，是构成大型应用的基础。

#### The revealing module pattern

未完待续...
