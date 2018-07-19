---
title: 《Node.js 设计模式》读书笔记 第四章
date: 2018-06-23 10:05:34
tags:
    - 读书笔记
---

# Asynchronous Control Flow Patterns with ES2015 and Beyond(使用 ES2015 以上异步控制流模式)

## Promise

`Promise` 是一种抽象的对象，我们通常允许函数返回一个名为 `Promise` 的对象，它表示异步操作的最终结果。通常情况下，我们说当异步操作尚未完成时，我们说 `Promise` 对象处于 `pending` 状态，当操作成功完成时，我们说 `Promise` 对象处于 `fulfilled` 状态，当操作错误终止时，我们说 `Promise` 对象处于 `rejected` 状态。一旦 `Promise` 处于 `fulfilled` 或 `rejected`，我们认为当前异步操作结束。

<!-- more -->

接收异步操作的结果（`settled`）使用 `then`:

```js
promise.then([onFulfilled], [onRejected])
```

使用 `then` 后就不必像 `CPS` 风格代码那样多重嵌套了，而是像这样：

```js
asyncOperation(arg).then(
  result => {
    // 错误处理
  },
  err => {
    // 正常结果处理
  }
)
```

`then` 方法同步返回另一个 `Promise`，`onFulfilled` 或 `onRejected` 返回的 `x` 不同时 `then` 方法返回的 `Promise` 也会不同：

- 如果 `x` 是一个值，则这个 `Promise` 对象会正确处理 `resolve(x)`
- 如果 `x` 是一个 `Promise` 对象或 `thenable`，则会正确处理 `x` 的处理后的结果 `resolve(x_fulfilled)`
- 如果 x 是一个异常，则会捕获异常 `reject(x)`

`Promise` 始终是异步的，就算直接同步地 `resolve` 也是一样，这能很好地避免 [`Zalgo`](https://github.com/oren/oren.github.io/blob/master/posts/zalgo.md)。

处理过程中抛出异常那么 `then` 返回的 `Promise` 会自动 `reject`，这个异常被作为 `reject` 的原因。

[`Promises/A+`](https://promisesaplus.com) 规范描述了 `then` 方法的行为，使得不同库的 `Promise` 能够兼容。

### Promise/A+ implementations(Promise/A+ 规范实现)

有很多实现了 `Promise/A+` 规范的库，但是目前基本上都用 `ES2015` 的 `Promise` 了，这个 `Promise` 是没有在标准上新加其他功能的。

`API` 可查看[官方文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)

### Promisifying a Node.js style function(使一个函数 Promise 化)

并不是所有的异步函数和库都支持 `promise`，有的时候得将一个基于回调的函数转换为返回 `Promise` 的函数，这个过程被称为 `Promise化`。

```js
module.exports.promisify = function(callbackBasedApi) {
  return function promisified() {
    const args = [].slice.call(arguments)
    return new Promise((resolve, reject) => {
      args.push((err, result) => {
        if (err) {
          return reject(err)
        }
        if (arguments.length <= 2) {
          resolve(result)
        } else {
          resolve([].slice.call(arguments, 1))
        }
      })
      callbackBasedApi.apply(null, args)
    })
  }
}
```

上面这个函数能把本来是基于 `callback` 的异步回调函数改为 `Promise` 风格的函数：

```js
const promisify = require('./promisify').promisify
let newCallbackBaseApi = promisify((input, callback) => {
  setTimeout(() => callback(null, input + 1), 100)
})

newCallbackBaseApi(999)
  .then(res => console.log(res))
  .catch(err => console.log(err))
```

### Sequential execution(顺序执行)

#### Sequential iteration(顺序迭代)

更改上一章爬虫程序：

```js
function spiderLinks(currentUrl, body, nesting) {
  let promise = Promise.resolve()
  if (nesting === 0) {
    return promise
  }
  const links = utilities.getPageLinks(currentUrl, body)
  links.forEach(link => {
    promise = promise.then(() => spider(link, nesting - 1))
  })
  return promise
}
```

新建一个 `resolve` 了 `undefined` 的 `Promise`，再将需要顺序执行的异步函数一个个按顺序填入 `then` 即完成了。等到最后一个 `then` 函数 `resolve` 了结果后整个顺序任务也就全部完成了。

#### Sequential iteration - the pattern(顺序迭代模式)

写一个通用的顺序处理任务模型：

```js
let tasks = [
  /* ... */
]
let promise = tasks.reduce((prev, task) => {
  return prev.then(() => {
    return task()
  })
}, Promise.resolve())

promise.then(() => {
  //All tasks completed
})
```

通过这种模式，我们可以将所有任务的结果收集到一个数组中，我们可以实现一个 `mapping` 算法，或者构建一个 `filter` 等等。

### Parallel execution(并行执行)

`Promise.all()` 并行执行多个异步任务。

### Limited parallel execution(限制并行执行任务数)

限制并行任务数可以简单地实现一个 `TaskQueue`:

```js
class TaskQueue {
  constructor(concurrency) {
    this.concurrency = concurrency
    this.running = 0
    this.queue = []
  }

  pushTask(task) {
    this.queue.push(task)
    this.next()
  }

  next() {
    while (this.running < this.concurrency && this.queue.length) {
      const task = this.queue.shift()
      task().then(() => {
        this.running--
        this.next()
      })
      this.running++
    }
  }
}
```

只需要将任务放到队列里就行了，然后开始 `next()`。

### Exposing callbacks and promises in public APIs(在公共 API 中暴露回调函数和 Promise)

`Promise` 固然有它的优点——易于理解和容易处理结果（`resolve`或`reject`），但是这要求开发者理解其中的原理，所以有些时候开发者更愿意使用回调函数模式。

像 `request` `redis` `mysql` 就使用回调函数的方式提供 `API`，`mongoose` `sequelize` 既支持回调函数的方式，又支持 `Promise` 的方式（不传回调函数时返回一个 `Promise`）。

最好是同时提供两种方式，这样方便开发者选择自己熟悉或者需要的方式，如：

```js
module.exports = function asyncDivision(dividend, divisor, cb) {
  return new Promise((resolve, reject) => {
    // [1]
    process.nextTick(() => {
      const result = dividend / divisor
      if (isNaN(result) || !Number.isFinite(result)) {
        const error = new Error('Invalid operands')
        if (cb) {
          cb(error) // [2]
        }
        return reject(error)
      }
      if (cb) {
        cb(null, result) // [3]
      }
      resolve(result)
    })
  })
}
```

```js
// 回调函数的方式
asyncDivision(10, 2, (error, result) => {
  if (error) {
    return console.error(error)
  }
  console.log(result)
})

// Promise化的调用方式
asyncDivision(22, 11)
  .then(result => console.log(result))
  .catch(error => console.error(error))
```

可以发现这种异步函数是默认返回一个 `Promise` 的，但是在异步处理完操作会判断回调函数是否已经传入，传入时会调用 `cb(null, result)` 或 `cb(error)`，然后始终都执行 `Promise` 需要的 `resolve` 或 `reject`。

## Generator(生成器)

### The basics of generators(生成器基础)

在 `function` 后面加上 `*` 就是声明生成器函数：

```js
function* makeGenerator() {
  // body
}
```

在 `makeGenerator()` 函数内部，使用关键字 `yield` 暂停执行并返回给调用者值：

```js
function* makeGenerator() {
  yield 'Hello World'
  console.log('Re-entered')
}
```

`makeGenerator()` 函数本质上是一个工厂，它在被调用时返回一个新的 `Generator` 对象:

```js
const gen = makeGenerator()
```

`next()` 函数用于启动/恢复 `Generator` 函数，并返回如下格式对象：

```js
{
  value: <yielded value>
  done: <true if the execution reached the end>
}
```

#### Generators as iterators(生成器函数作为迭代器)

```js
function* iteratorGenerator(arr) {
  for (let i = 0; i < arr.length; i++) {
    yield arr[i]
  }
}
const iterator = iteratorGenerator(['apple', 'orange', 'watermelon'])
let currentItem = iterator.next()
while (!currentItem.done) {
  console.log(currentItem.value)
  currentItem = iterator.next()
}
```

上面代码会输出如下：

```
apple
orange
watermelon
```

每次 `yield` 会暂停生成器函数并返回一个值，`next` 会恢复生成器函数的执行，恢复的时候的状态与暂停时候的状态一致。

#### Passing values back to a generator(传值给生成器函数)

想要传递值给 `Generator` 只需要添加 `next` 的参数就行了，传递的值会赋予给 `yield` 的返回值：

```js
function* twoWayGenerator() {
  const what = yield null
  console.log('Hello ' + what)
}
const twoWay = twoWayGenerator()
twoWay.next()
twoWay.next('world')
```

第一个 `next` 启动 `Generator`，接着 `yield` 暂定函数执行，再然后 `next('world')` 恢复函数执行并传递值 `world` 给 `Generator`，`yield` 收到该参数作为返回值返回（`what` 的值为 world）。

> 注意也可以返回一个异常，`next(new Error())`，这个错误就像是在生成器中抛出的一样，可以使用 `try...catch` 捕获。

### Asynchoronous control flow with generators(使用生成器做异步控制流)

直接看代码：

```js
function asyncFlow(generatorFunction) {
  function callback(err) {
    if (err) {
      return generator.throw(err)
    }
    const results = [].slice.call(arguments, 1)
    generator.next(results.length > 1 ? results : results[0])
  }
  const generator = generatorFunction(callback)
  generator.next()
}

const fs = require('fs')
const path = require('path')
asyncFlow(function*(callback) {
  const fileName = path.basename(__filename)
  const myself = yield fs.readFile(fileName, 'utf8', callback)
  yield fs.writeFile(`clone_of_${filename}`, myself, callback)
  console.log('Clone created')
})
```

`asyncFlow` 接收一个 `Generator` 函数，这个生成器函数里可以做一些异步的操作，异步操作完成时会调用 `asyncFlow` 中的 `callback` 将结果返回或者将错误抛出，不管如何都会被 `Generator` 函数中获取到（`yield` 返回值或者 `try...catch` 捕获异常，上面例子是获取返回值），`myself` 能拿到 `readFile` 的结果。

可以发现使用异步控制流后，可以像同步代码方式那样书写异步代码了，它的原理就是每个异步函数操作完后会恢复 `Generator` 函数的运行并返回处理的结果值给暂停的地方。

> 上述异步控制流还有两种变体，一种使用 `Promise`，另一种使用 `thunks`。`thunk` 指的是一个函数，接收原函数中除了回调函数以外的参数，返回一个只接收回调函数的函数，如 `fs.readFile()`:

```js
function readFileThunk(filename, options) {
  return function(callback) {
    fs.readFile(filename, options, callback)
  }
}
```

这两种变种允许我们创建没有回调函数作为参数的 `Generator`，就像下面(`thunk`)这样：

```js
function asyncFlowWithThunks(generatorFunction) {
  function callback(err) {
    if (err) {
      return generator.throw(err)
    }
    const results = [].slice.call(arguments, 1)
    const thunk = generator.next(results.length > 1 ? results : results[0])
      .value
    thunk && thunk(callback)
  }
  const generator = generatorFunction()
  const thunk = generator.next().value
  thunk && thunk(callback)
}

asyncFlowWithThunks(function*() {
  const fileName = path.basename(__filename)
  const myself = yield readFileThunk(__filename, 'utf8')
  yield writeFileThunk(`clone_of_${fileName}`, myself)
  console.log('Clone created')
})
```

从 `generator.next().value` 取到 `thunk` 的返回函数，再将回调函数传入，这样 `Generator` 函数就不用接收回调函数作为参数了。更详细有关 `thunk` 的介绍可以移步 `Github` 的 [thunks](https://github.com/thunks/thunks)。

`Promise` 也是类似的，使用上面有提到的 `promisify` 将异步函数转为 `Promise` 形式，然后类似处理（这里是我自己实现的一个版本）：

```js
function asyncFlowWithPromise(generatorFunction) {
  function processPromise() {
    promise
      .then(res => {
        if (res) {
          promise = generator.next(res.length > 1 ? res : res[0]).value
          promise && processPromise()
        }
      })
      .catch(err => generator.throw(err))
  }

  const generator = generatorFunction()
  let promise = generator.next().value
  promise && processPromise()
}

asyncFlowWithPromise(function*() {
  // readFilePromise() 生成一个Promise，使用promisify转换而来
  const fileName = path.basename(__filename)
  const myself = yield readFilePromise(__filename, 'utf8')
  yield writeFilePromise(`clone_of_${fileName}`, myself)
  console.log('Clone created')
})
```

#### Generator-based control flow using co(使用 co 的基于 Generator 的控制流)

`co` 已经包含了以上两种形式的异步控制流了，可以支持 5 种 `yieldable` 的类型：

- Promises
- Thunks (functions)
- array (parallel execution)
- objects (parallel execution)
- Generators and GeneratorFunctions

`co` 源码的理解可以查看[这里](https://i5ting.github.io/wechat-dev-with-nodejs/async/co.html)

#### Sequential execution(顺序执行)

使用 `co` 库可以很简单地实现任务的顺序执行，我们更改爬虫程序为下面这样：

```js
// thunkify 可以将接收回调函数的异步函数变为 thunk，使用 promisify 也是一样的，代码不需要变动
const thunkify = require('thunkify')
const co = require('co')
const request = thunkify(require('request'))
const fs = require('fs')
const mkdirp = thunkify(require('mkdirp'))
const readFile = thunkify(fs.readFile)
const writeFile = thunkify(fs.writeFile)
const nextTick = thunkify(process.nextTick)

function* download(url, filename) {
  console.log(`Downloading ${url}`);
  const response = yield request(url);
  const body = response[1];
  yield mkdirp(path.dirname(filename));
  yield writeFile(filename, body);
  console.log(`Downloaded and saved ${url}`);
  return body;
}

function* spider(url, nesting) {
  cost filename = utilities.urlToFilename(url);
  let body;
  try {
    body = yield readFile(filename, 'utf8');
  } catch (err) {
    if (err.code !== 'ENOENT') {
      throw err;
    }
    body = yield download(url, filename);
  }
  yield spiderLinks(url, body, nesting);
}

function* spiderLinks(currentUrl, body, nesting) {
  if (nesting === 0) {
    return nextTick();
  }
  const links = utilities.getPageLinks(currentUrl, body);
  for (let i = 0; i < links.length; i++) {
    yield spider(links[i], nesting - 1);
  }
}

//只有开始的入口需要调用 co，因为 co 是可以递归处理 yieldable 的数据，Generators 也是 yieldable 的
co(function*() {
  try {
    yield spider(process.argv[2], 1);
    console.log(`Download complete`);
  } catch (err) {
    console.log(err);
  }
});
```

#### Parallel execution(并行执行)

并行执行就很简单了，将需要并行的任务包装成一个数组就行了。

```js
function* spiderLinks(currentUrl, body, nesting) {
  if (nesting === 0) {
    return nextTick()
  }
  const links = utilities.getPageLinks(currentUrl, body)
  const tasks = links.map(link => spider(link, nesting - 1))
  yield tasks
}
```

也可以使用基于回调函数的方式实现并行执行：

```js
function spiderLinks(currentUrl, body, nesting) {
  if (nesting === 0) {
    return nextTick()
  }
  // 返回一个thunk
  return callback => {
    let completed = 0,
      hasErrors = false
    const links = utilities.getPageLinks(currentUrl, body)
    if (links.length === 0) {
      return process.nextTick(callback)
    }

    function done(err, result) {
      if (err && !hasErrors) {
        hasErrors = true
        return callback(err)
      }
      if (++completed === links.length && !hasErrors) {
        callback()
      }
    }
    for (let i = 0; i < links.length; i++) {
      co(spider(links[i], nesting - 1)).then(done)
    }
  }
}
```

`spiderLinks` 改为上面这种代码后就不再是 `Generator` 函数了，它返回的是一个 `thunk`，这样有利于支持其他的基于回调函数或者 `Promise` 的控制流算法。

#### Limited parallel execution(限制并行执行)

这里有几种凡是可以限制并行的任务数：

- `TaskQueue`(基于 `Promise` 或者 `callback`)
- 使用库，`async` 或者 `co-limiter`
- 自己实现算法（生产者-消费者）

这里主要看看第三种，直接看修改后的 `TaskQueue`:

```js
class TaskQueue {
  constructor(concurrency) {
    this.concurrency = concurrency
    this.running = 0
    this.taskQueue = []
    this.consumerQueue = []
    this.spawnWorkers(concurrency)
  }
  pushTask(task) {
    if (this.consumerQueue.length !== 0) {
      this.consumerQueue.shift()(null, task)
    } else {
      this.taskQueue.push(task)
    }
  }
  spawnWorkers(concurrency) {
    const self = this
    for (let i = 0; i < concurrency; i++) {
      co(function*() {
        while (true) {
          const task = yield self.nextTask()
          yield task
        }
      })
    }
  }
  nextTask() {
    return callback => {
      if (this.taskQueue.length !== 0) {
        return callback(null, this.taskQueue.shift())
      }
      this.consumerQueue.push(callback)
    }
  }
}
```

理解上述代码最好是自己手动去跑一边，一步步 `debug`，看到是如何进行的。

1.  第一步初始化 `TaskQueue`，传入最大并行任务数（假设为 `n`），这个时候会创建 `n` 个消费者，也就是通过 `nextTask` 创建的 `callback`(在 `co` 中的 `thunkToPromise` 中创建的) 函数，`nextTask` 返回一个 `thunk` 函数，在 `while` 循环中 `yield self.nextTask()` 来执行 `thunk` 的函数体，这个时候任务队列 `taskQueue` 还是空的，所以这个消费者会被暂时挂起（被 `push` 到消费者队列 `consumerQueue`中），之后该 `TaskQueue` 等待新任务的到来。
2.  生产者生产任务，也就是通过 `pushTask` 生产任务，这个时候判断是否有空闲的消费者，从消费者队列中取出 `callback`(`yield self.nextTask()` 时 `push` 进去的)，执行这个 `callback` 传入异步任务作为参数。
3.  收到异步任务后，立即 `resolve` 掉，接着恢复 `Generator` 函数的执行，继续执行时 `task` 能拿到 `next(ret)` 的参数也就是这个异步任务，接着继续执行这个 `task`，这个任务结束后又会回到 `while` 循环取下一个任务执行，以此类推。
4.  没有多余的消费者时会暂时将任务存到任务队列，等待消费者被释放。

### Async await using Babel(利用 Babel 使用 async 和 await)

我们发现，要理解上面那种 `Generator` 式的代码实在太难了，还好 `ES7` 规范发布了新的关键字 `async` 和 `await`，先来看看这两个关键字是怎么提高代码可读性的：

```js
const request = require('request')

function getPageHtml(url) {
  return new Promise(function(resolve, reject) {
    request(url, function(error, response, body) {
      resolve(body)
    })
  })
}
async function main() {
  const html = await getPageHtml('http://google.com')
  console.log(html)
}

main()
console.log('Loading...')
```

`getPageHtml` 返回一个 `Promise`，`async` 关键字修饰函数表明这个函数是处理异步操作的，并且可以使用 `await` 关键字了，`await` 关键字告诉 `JavaScript` 解释器在执行下面的语句之前要等待 `getPageHtml` 返回的 `Promise` 的结果。程序中只有 `main` 那段代码是异步，其他的还是同步的，所以是先看到 `Loading` 字样再看到网页的内容的。

#### Installing and running Babel(安装并运行 Babel)

`Babel` 是一个 `JavaScript` 编译器(或翻译器)，能够使用语法转换器将高版本的 `JavaScript` 代码转换成其他 `JavaScript` 代码。语法转换器允许例如我们书写并使用 `ES2015`，`ES2016`，`JSX` 和其它的新语法，来翻译成往后兼容的代码，在 `JavaScript` 运行环境如浏览器或 `Node.js` 中都可以使用 `Babel`。

详细的安装与运行参考[官方文档](https://babeljs.io/docs/en/index.html)。

`ES7` 的语法可以使用 [babel-plugin-transform-async-to-generator](https://babeljs.io/docs/en/babel-plugin-transform-async-to-generator)。

### Comparison(比较)

这里是几种处理 `JavaScript` 异步的方式的比较：

| 方案        | 优点                                                                                                           | 缺点                                                                                                                       |
| ----------- | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| 原生 js     | 不需要额外的库 </br> 性能最高 </br> 兼容性好 <br> 允许简单或更复杂算法的创建                                   | 可能需要更多的代码和相对复杂的算法                                                                                         |
| Async 库    | 简化常见的控制流模式 </br> 基于 callback 的方式 </br> 较好的性能                                               | 需要额外的库 </br> 不适用于更高级的流控制                                                                                  |
| Promises    | 简化常见的控制流模式 </br> 鲁棒的 error 处理 </br> ES6 规范一部分 </br>确保 onFulfilled 和 onRejected 延迟调用 | 需要将基于 callback 的函数 promisify </br> 带来了较小的性能上的损失                                                        |
| Generators  | 使得非阻塞 API 用起来和阻塞 API 一样 </br> 简化错误处理 </br> ES6 的特征                                       | 需要辅助的流控制库 </br> 需要 callback 或 promise 来实现非顺序流 </br>需要 thunkify 或 promisify 不是基于 generator 的 API |
| Async await | 使得非阻塞 API 用起来和阻塞 API 一样 </br> 简单直观的语法                                                      | 需要 Babel （为了兼容浏览器，Promise 和 Generators 也要用 Babel）                                                          |

### 总结

其实我个人建议是使用 `async await` 的方式的，这种方式使得代码看起来十分清爽，比原生 `js` 要好太多了，而其他的几种太过繁琐，这几种为了兼容浏览器也不得不使用 `Babel`、`polyfill`，所以 `asycn await` 的缺点也就不那么明显了。
