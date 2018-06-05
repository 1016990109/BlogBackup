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
const fs = require('fs')

function readJSONThrows(filename, callback) {
  fs.readFile(filename, 'utf8', (err, data) => {
    if (err) {
      return callback(err)
    }
    callback(null, JSON.parse(data))
  })
}
```

在上面的函数中，如果`JSON.parse(data)`异常的话是没有办法捕获的：

```js
try {
  readJSONThrows('nonJSON.txt', function(err, result) {
    // ...
  })
} catch (err) {
  console.log('This will not catch the JSON parsing exception')
}
```

上面`catch`语句将捕获不到错误，因为错误是在回调函数中产生的。然而，我们仍然有机会在应用程序终止之前执行一些清理或日志记录。事实上，当这种情况发生时，Node.js 会在退出进程之前发出一个名为`uncaughtException`的特殊事件：

```js
process.on('uncaughtException', err => {
  console.error(
    'This will catch at last the ' + 'JSON parsing exception: ' + err.message
  )
  // Terminates the application with 1 (error) as exit code:
  // without the following line, the application would continue
  process.exit(1)
})
```

需要注意的是，`uncaughtException`会使得应用处于一个不能保证一致的状态 ，而这可能导致不可预见的错误。比如还有未完成的 I/O 请求正在运行或关闭，这可能导致不一致。所以建议，尤其是在生产环境，在收到任何`uncaught exception`之后停止应用的运行。

### The module system and its patterns(模块系统和其中的模式)

模块可以隐藏不想暴露的函数、变量，是构成大型应用的基础。

#### The revealing module pattern(模块模式)

`JavaScript` 是没有命名空间的，在全局范围内运行的程序会污染全局命名空间，造成相关变量、数据、方法名的冲突。解决该问题的一个比较流行的做法是使用 `模块模式`:

```js
const module = (() => {
  const privateFoo = () => {
    // ...
  }
  const privateBar = []
  const exported = {
    publicFoo: () => {
      // ...
    },
    publicBar: () => {
      // ...
    }
  }
  return exported
})()
console.log(module)
```

该模式利用自执行函数创建私有空间，只导出需要暴露的部分，前面的代码中， `module` 变量只包含了暴露的 `API`，而内部的其他部分是外面访问不到的。这个模式背后的思想就是用来构建 `Node.js` 模块系统的基础。

#### Node.js modules explained(Node.js 模块解释)

`CommonJS` 是一个旨在规范 `JavaScript` 生态系统的组织，他们提出了 `CommonJS模块规范`。`Node.js` 在此规范之上构建了其模块系统，并添加了一些自定义的扩展。每个模块都在自己的私有空间下运行，所以在模块内定义的本地变量不会污染全局变量。

##### A homemade module loader(自定义模块加载器)

为了解释加载器是如何工作的，先简单勾勒一个类似的系统，下面的代码模仿了内部函数 `require()` 的一部分功能：

```js
function loadModule(filename, module, require) {
  const wrappedSrc = `(function(module, exports, require) {
         ${fs.readFileSync(filename, 'utf8')}
       })(module, module.exports, require);`
  eval(wrappedSrc)
}
```

模块的源码被包装入一个函数，并且是使用模块模式的。区别在于传递了一些参数到模块中，实际上就是 `module`, `exports`, `require`。`exports` 参数被初始化为 `module.exports`。

> 注意： 上面只是个示例，其实很少使用 `eval` 来执行源码，这可能导致注入攻击，使用 `eval` 要十分谨慎。

现在通过实现 `require()` 函数来看看这都些变量中的包含了什么内容：

```js
const require = moduleName => {
  console.log(`Require invoked for module: ${moduleName}`)
  const id = require.resolve(moduleName)
  if (require.cache[id]) {
    return require.cache[id].exports
  }
  //模块元数据
  const module = {
    exports: {},
    id: id
  }
  //更新缓存
  require.cache[id] = module
  //加载模块
  loadModule(id, module, require)
  //返回导出的变量
  return module.exports
}
require.cache = {}
require.resolve = moduleName => {
  /* 通过模块名作为参数resolve一个完整的模块 */
}
```

上面函数模拟了原生 `require()` 函数，并不能准确完美地反应真实的行为，但是却能帮助我们理解一个模块是怎么被定义和加载的：

1.  一个模块的名字作为输入被接收，我们需要做的第一件事就是找到这个模块的路径(我们称之为`id`)，这个依靠 `require.resolve()`来完成。
2.  如果模块过去被加载过，那它应该存在于缓存。这种情况下我们直接返回就行。
3.  如果模块尚未加载，我们将初始化首次加载模块环境。具体来说就是，创建一个模块(`module`)对象，其中包含一个 `exports` (被初始化为空的对象字面量`{}`)属性。该属性将被模块的代码用于导出模块的公共 `API`。
4.  模块被缓存。
5.  像前面所看到的一样，源代码从文件中被加载，接着被执行。我们给模块提供一个刚才创建的 `module` 对象和一个 `require()` 函数的引用。模块通过修改或替换 `module.exports` 来提供公共 `API`。
6.  最后，包含公共 `API` 的 `module.exports`  返回给调用者。

##### Defining a module(定义一个模块)

让我们看看怎么定义一个模块：

```js
//加载另一个依赖
const dependency = require('./anotherModule')
//一个私有函数
function log() {
  console.log(`Well done ${dependency.username}`)
}
//API 被导出给外部用
module.exports.run = () => {
  log()
}
```

除了 `module.exports` 的内容其他都是私有的，当模块被加载的时候这个变量的内容被返回且被缓存。

##### Defining globals(定义全局内容)

即使在模块中声明的所有变量和函数都在其本地范围内定义，仍然可以定义全局变量。事实上，模块系统公开了一个名为 `global` 的特殊变量。分配给此变量的所有内容将会被定义到全局环境下。

> 注意：污染全局变量是不好的，模块化的优势就不在了，所以只有当你真的需要用的时候再用吧！

##### module.exports vs exports

`exports` 只是 `module.exports` 的一个引用，所以在 `exports` 中添加新属性是有效的，能更新 `module.exports` 的内容，而对 `exports` 重新赋值则不会更新 `module.exports`，只是让 `exports` 指向了另一个对象；但是对 `module.exports` 重新赋值就是实实在在地更改了 `module` 了，是能起作用的。

```js
//有效
exports.foo = () => {
  console.log('hello')
}

//无效
exports = {
  foo: () => {
    console.log('hello')
  }
}

//有效
module.exports = {
  foo: () => {
    console.log('hello')
  }
}
```

##### The require function is synchronous(require 函数是同步的)

原生的 `require()` 函数也是同步的，所以对 `module.exports` 的赋值操作也是要同步的。下面这种代码就是错误的：

```js
setTimeout(() => {
  module.exports = function() {
    // ...
  }
}, 100)
```

这就限制了我们绝大多数情况下都是使用同步的代码定义模块，这就是 `Node.js` 核心库为一些异步函数提供可选的同步的 `API` 的原因。

如果需要在模块初始化过程中使用异步方法，那么可以返回一个未初始化的模块，让使用者之后去初始化这个模块，这就导致了 `require` 不能保证模块被立即使用。

出于好奇，你可能想知道为什么 `Node.js` 早期是有异步的 `require()` 函数后来又被移除了，这是因为在初始化的过程中处理异步的 I/O 带来的复杂性比优势大太多了。

##### The resolve algorithm(resolve 算法)

为了解决[依赖地狱](https://zh.wikipedia.org/wiki/相依性地狱)问题，`Node.js` 根据模块的被加载的位置来加载不同版本的模块，这些理念也被运用到 `npm` 和 `require` 的 `resolve` 算法中。

`resolve()` 接收 `moduleName` 作为参数，并返回模块的完整路径。

`resolve` 算法的三个主要分支：

1.  **File modules**(文件模块)：模块名是 `/` 开头认为是绝对路径，以 `./` 开头则认为是相对当前使用 `require` 的模块的路径。
2.  **Core modules**(核心模块)：模块名不以 `/` 或 `./` 开头则优先从核心库开始查找。
3.  **Package modules**(包模块)：核心库没有查找到时，再从当前目录的 `node_modules` 中查找相应的模块，没有则继续往上层的 `node_modules` 中找直到系统的根目录。

对于文件和包模块，单个文件和目录也可以匹配到 `moduleName`。特别地，算法将尝试匹配以下内容：

* `<moduleName>.js`
* `<moduleName>/index.js`
* 在`<moduleName>/package.json` 的 `main` 值下声明的文件或目录

更详尽的 `resolve` 算法请看[这里](https://nodejs.org/api/modules.html#modules_all_together)。

`node_modules` 目录实际上是 `npm` 安装每个包并存放相关依赖关系的地方：

```
myApp
├── foo.js
└── node_modules
    ├── depA
    │   └── index.js
    └── depB
        │
        ├── bar.js
        ├── node_modules
        ├── depA
        │    └── index.js
        └── depC
             ├── foobar.js
             └── node_modules
                 └── depA
                     └── index.js
```

可以发现 `depA`, `depB`, `depC` 都有它们自己的依赖， 所以同样使用 `require('depA')`，在不同的地方加载就会加载不同的模块，如：

* 在 `/myApp/foo.js` 中调用的 `require('depA')` 会加载 `/myApp/node_modules/depA/index.js`
* 在 `/myApp/node_modules/depC/foobar.js` 中调用的 `require('depA')` 会加载 `/myApp/node_modules/depC/node_modules/depA/index.js`

`resolve` 算法是 `Node.js` 依赖关系管理的核心部分，它的存在使得即便应用程序拥有成百上千包的情况下也不会出现冲突和版本不兼容的问题。

当我们调用 `require()` 时，解析算法对我们是透明的。然而，仍然可以在任何模块中通过调用 `require.resolve()` 使用该算法。

##### The module cache(模块缓存)

模块只有在第一次被 `require` 的时候才会去加载，之后都是直接从缓存中获取的，除了提升性能外还有 2 个好处：

* 模块依赖重复利用
* 从给定的包中获取相同的模块总是返回一个实例，避免了冲突

需要的时候，模块的缓存可以通过 `require.cache` 访问，想要使缓存的模块失效可以删除 `require.cache` 中对应的 key 就行了，一般来说只在测试中做，在正常环境下是非常危险的。

##### Circular dependencies(循环依赖)

假设有两个模块：

* 模块 `a.js`

```js
exports.loaded = false
const b = require('./b')
module.exports = {
  bWasLoaded: b.loaded,
  loaded: true
}
```

* 模块 `b.js`

```js
exports.loaded = false
const a = require('./a')
module.exports = {
  aWasLoaded: a.loaded,
  loaded: true
}
```

试着加载模块：

```js
const a = require('./a')
const b = require('./b')
console.log(a)
console.log(b)
```

结果：

```
{bWasLoaded: true,loaded: true}
{aWasLoaded: false,loaded: true}
```

先引入 `a` 模块，这时候 `a` 引入 `b` 模块，而 `b` 又去引入 `a` 模块，`b` 引入 `a` 的时候会直接返回缓存中的 `a`，也就是只有 `{load: false}`，然后 `b` 将 `{aWasLoaded: false, loaded: true}` 返回给 `a`，所以 `a` 中先拿到的 `b` 是完整的，而 `b` 拿到的 `a` 是不完整的。详情可查看[这里](https://nodejs.org/api/modules.html#modules_cycles)。

在项目中千万要注意不要出现循环依赖的情况，不然可能会出现严重的问题。

#### Module definition patterns(模块定义模式)

##### Named exports(命名导出)

暴露公共 `API` 最常用的方法就是命名导出，将想要公开的值分配给 `exports`（或者 `module.exports`):

```js
//file logger.js
exports.info = message => {
  console.log('info: ' + message)
}
exports.verbose = message => {
  console.log('verbose: ' + message)
}
```

导出的函数就可以当做已加载模块的属性使用了:

```js
//file main.js
const logger = require('./logger')
logger.info('This is an informational message')
logger.verbose('This is a verbose message')
```

`CommonJS` 规范只允许使用 `exports` 来导出公共的成员，所以命名导出也是唯一的与 `CommonJS` 规范兼容的模式。而 `module.exports` 是为了支持更广泛定义模式的一个扩展。

##### Exporting a function(导出一个函数)

它只暴露了一个函数，为模块提供了一个明确的入口点，使其更易于理解和使用，也很好地体现了单一职责原则，也被称为 `substack` 模式：

```js
// file logger.js
module.exports = message => {
  console.log(`info: ${message}`)
}
```

该模式的一种可能扩展是使用导出的函数作为其他公共 `API` 的命名空间，例如：

```js
module.exports.verbose = message => {
  console.log(`verbose: ${message}`)
}
```

```js
// file main.js
const logger = require('./logger')
logger('This is an informational message')
logger.verbose('This is a verbose message')
```

`Node.js` 鼓励 `Single Responsibility Principle(SRP)`(单一职责原则)：每个模块负责单一功能，该职责也完全由该模块封装。

> `substatck` 模式：导出一个函数来暴露主要功能(如上面的`logger(***)`)，使用导出的函数作为命名空间来导出次要的功能(如上面的`logger.verbose(***)`)，注意主要功能定义要在前，不然次要功能会被覆盖，因为 `module.exports` 被重新赋值了！

##### Exporting a constructor(导出一个构造器)

导出构造器是导出一个函数的特例，区别在于使用者可以用构造器创建实例也可以扩展原型并创建新类:

```js
//file logger.js
function Logger(name) {
  this.name = name
}
Logger.prototype.log = function(message) {
  console.log(`[${this.name}] ${message}`)
}
Logger.prototype.info = function(message) {
  this.log(`info: ${message}`)
}
Logger.prototype.verbose = function(message) {
  this.log(`verbose: ${message}`)
}
module.exports = Logger
```

下面是如何使用的代码:

```js
//file main.js
const Logger = require('./logger')
const dbLogger = new Logger('DB')
dbLogger.info('This is an informational message')
const accessLogger = new Logger('ACCESS')
accessLogger.verbose('This is a verbose message')
```

上面的 `logger` 也可以使用 `ES2015` 的 `class` 改写：

```js
class Logger {
  constructor(name) {
    this.name = name
  }
  log(message) {
    console.log(`[${this.name}] ${message}`)
  }
  info(message) {
    this.log(`info: ${message}`)
  }
  verbose(message) {
    this.log(`verbose: ${message}`)
  }
}
module.exports = Logger
```

这种模式的变种包括对不使用 `new` 调用的防御(不使用 `new` 只会当做普通函数调用，而不会返回实例，而且可能会污染调用者(`this.name = name`这就污染了`Logger()`所属的对象了，这里是`global`))，这可以让我们把模块当做工厂使用:

```js
function Logger(name) {
  if (!(this instanceof Logger)) {
    return new Logger(name)
  }
  this.name = name
}
```

其实这很简单：我们检查 `this` 是否存在，并且是 `Logger` 的一个实例。如果这些条件中的任何一个都为 `false`，则意味着 `Logger()` 函数在不使用 `new` 的情况下被调用，然后继续正确创建新实例并将其返回给调用者。这种技术允许我们将模块也用作工厂(**这里原书存在问题所以做了些改动**)：

```js
// 原书
// file logger.js
const Logger = require('./logger')
const dbLogger = Logger('DB')
accessLogger.verbose('This is a verbose message')
// 改后
// file main.js
const Logger = require('./logger')
const dbLogger = Logger('DB')
dbLogger.verbose('This is a verbose message')
```

另一种更加清晰的方法来实现这个防御的是使用 `ES2015` 的 `new.target` 的语法(`Node.js` 第 6 版开始)，`new.target` 是用来检测一个函数或者构造器是否是使用 `new` 调用的，如果是则为 `true`:

```js
function Logger(name) {
  if (!new.target) {
    return new Logger(name) //原书为 new LoggerConstructor(name)
  }
  this.name = name
}
```

> 注意：如果使用 `ES2015` 的 `class` 则不需要做 `new` 的防御，因为不使用 `new` 关键字来调用构造函数会抛出异常：`TypeError: Class constructor Logger cannot be invoked without 'new'`

##### Exporting an instance(导出一个实例)

我们可以利用 `require()` 的缓存机制来轻松地定义具有从构造函数或工厂创建的状态的有状态实例，可以在不同模块之间共享:

```js
//file logger.js
function Logger(name) {
  this.count = 0
  this.name = name
}
Logger.prototype.log = function(message) {
  this.count++
  console.log('[' + this.name + '] ' + message)
}
module.exports = new Logger('DEFAULT')
```

使用：

```js
// file main.js
const logger = require('./logger')
logger.log('This is an informational message')
```

因为模块会被缓存，所以通过 `require` 引入的模块都是同一个实例，共享状态，这就像单例模式一样。但是实际上有可能并不只有一个实例，我们知道在安装依赖的时候，一个模块可能会被安装多次(版本不同)，这就会有多个同时运行于一个 `Node.js` 的应用程序中实例出现。

该模式的一个扩展是不仅导出实例，同时也导出用于创建实例的构造器，这让使用者可以创建新的实例或者必要时扩展对象，这有点类似导出命名空间([Exporting a function](#Exporting-a-function-导出一个函数)):

```js
// other code
module.exports.Logger = Logger
```

```js
// file main.js
const customLogger = new logger.Logger('CUSTOM')
customLogger.log('This is an informational message')
```

##### Modifying other modules or the global scope(修改其他模块或全局作用域)

一个模块可以不导出任何东西，一个模块也可以修改全局域或者其他已经缓存的模块。

> 注意：修改全局域或者其他模块是不好的，但是这在某些情况(如测试)下是有用的。

这是向另一个模块添加函数的例子：

```js
// file patcher.js
// ./logger is another module
require('./logger').customMessage = () =>
  console.log('This is a new functionality')
```

所以只要在引入了 `patcher` 模块之后才有 `customMessage` 函数，这是非常危险的，特别是当多个模块与相同的实体进行交互时。

#### The observer pattern(观察者模式)

观察者模式是对 `Node.js` 响应模型的理想解决方案，也是对回调的完美补充。我们给出以下定义：

> 模式(观察者)定义一个对象(`subject`，主题)，它可以在其状态发生变化时通知一组观察者(或监听器)。

和回调的不同在于可以通知多个观察者，传统的 `CPS` 模式只能传递结果给一个监听器(就是 `callback`)。

##### The EventEmiiter class

在传统的面向对象编程中，观察者模式需要接口，具体类和层次结构。在 `Node.js` 中，都变得简单得多。观察者模式已经内置在核心模块中，可以通过 `EventEmitter` 类来实现。 `EventEmitter` 类允许我们注册一个或多个函数作为监听器，当特定的事件类型被触发时，它的回调将被调用，以通知其监听器:

![EventEmitter](/assets/img/EventEmitter.png)

`EventEmitter` 是从事件核心模块导出的原型，下面是如何获取一个实例的方法：

```js
const EventEmitter = require('events').EventEmitter
const eeInstance = new EventEmitter()
```

`EventEmitter` 的基本方法如下：

* `on(event, listener)`:注册一个新的监听器(一个函数)到一个事件(string)中。
* `once(event, listener)`:也是为一个事件注册监听器，但是在事件第一次被触发后监听器被移除。
* `emit(event, [arg1], [...])`:生成一个新事件，并附带参数给监听器。
* `removeListener(event, listener)`:删除指定事件的一个监听器。

上述每个方法都返回 `EventEmitter` 以便链式调用。监听器是类似于 `function([arg1], [...])` 的函数，可以接受通过 `emit` 传入的参数，函数中 `this` 是指向生产事件的 `EventEmitter` 实例。

> 和回调函数不同的是，监听器第一个参数就是来自于 `emit` 的任何类型的数据，而不是 `error`。

##### Creating and using EventEmitter(创建和使用 EventEmitter)

以下代码显示了在文件列表中找到匹配特定正则的文件内容时，使用 `EventEmitter` 实现实时通知订阅者的功能：

```js
const EventEmitter = require('events').EventEmitter
const fs = require('fs')

function findPattern(files, regex) {
  const emitter = new EventEmitter()
  files.forEach(function(file) {
    fs.readFile(file, 'utf8', (err, content) => {
      if (err) return emitter.emit('error', err)
      emitter.emit('fileread', file)
      let match
      if ((match = content.match(regex)))
        match.forEach(elem => emitter.emit('found', file, elem))
    })
  })
  return emitter
}

findPattern(['fileA.txt', 'fileB.json'], /hello \w+/g)
  .on('fileread', file => console.log(file + ' was read'))
  .on('found', (file, match) =>
    console.log('Matched "' + match + '" in file ' + file)
  )
  .on('error', err => console.log('Error emitted: ' + err.message))
```

从上面我们也看到了是如何链式调用的了。

> 注意，这里因为 `fs.readFile` 是异步的，会在下一次事件循环中才执行，所以实际上还是先注册了事件监听(`on`)再生产了事件(`emit`)。

##### Propagating errors(传递错误)

对于错误事件，最佳做法是注册错误的侦听器(如在`findPattern`中的`error`事件一样)，因为 Node.js 会以特殊的方式处理它，并且如果没有找到相关联的侦听器，将自动抛出异常并退出程序。

##### Making any object observable(使任意对象可观察)

直接上代码：

```js
const EventEmitter = require('events').EventEmitter
const fs = require('fs')
class FindPattern extends EventEmitter {
  constructor(regex) {
    super()
    this.regex = regex
    this.files = []
  }
  addFile(file) {
    this.files.push(file)
    return this
  }
  find() {
    this.files.forEach(file => {
      fs.readFile(file, 'utf8', (err, content) => {
        if (err) {
          return this.emit('error', err)
        }
        this.emit('fileread', file)
        let match = null
        if ((match = content.match(this.regex))) {
          match.forEach(elem => this.emit('found', file, elem))
        }
      })
    })
    return this
  }
}
const findPatternObject = new FindPattern(/hello \w+/)
findPatternObject
  .addFile('fileA.txt')
  .addFile('fileB.json')
  .find()
  .on('found', (file, match) =>
    console.log(`Matched "${match}"
       in file ${file}`)
  )
  .on('error', err => console.log(`Error emitted ${err.message}`))
```

这里直接使用 `ES6` 的继承语法(不推荐使用[util.inherits](https://nodejs.org/docs/latest/api/util.html#util_util_inherits_constructor_superconstructor)方式了，这里书中还没有更新过来)来实现对 `EventEmitter` 的继承， 这在 `Node.js` 生态系统中是一个很常见的模式，例如，核心 `HTTP` 模块的 `Server` 对象定义了 `listen()`，`close()`，`setTimeout()` 等方法，并且在内部它也继承自 `EventEmitter` 函数，从而允许它在收到新的请求时生产 `request` 事件、在建立新的连接时生产 `connection` 事件、或者在服务器关闭时生产 `closed` 事件。

##### Synchronous and asynchronous events(同步和异步事件)

同一个 `EventEmitter` 中不要同时使用同步和异步触发事件(这样来阻止`Zalgo`)，同步事件是要在所有监听器都注册完了才能触发的，而异步事件要保证在下一个事件循环周期前不被触发，异步事件触发前都能添加监听器(上面的`findPattern`就是这种)。

像下面这种就是不会正常运行的：

```js
const EventEmitter = require('events').EventEmitter
class SyncEmit extends EventEmitter {
  constructor() {
    super()
    this.emit('ready')
  }
}
const syncEmit = new SyncEmit()
syncEmit.on('ready', () => console.log('Object is ready to be  used'))
```

##### EventEmitter versus callbacks(EventEmitter vs 回调函数)

定义异步 `API` 时是使用 `callbacks` 还是 `EventEmitter` 的一般规则：当结果必须通过异步方式返回时使用 `callbacks`，当有什么东西需要被传达时使用 `EventEmitter`。

来看一个例子：

```js
function helloEvents() {
  const eventEmitter = new EventEmitter()
  setTimeout(() => eventEmitter.emit('hello', 'hello world'), 100)
  return eventEmitter
}

function helloCallback(callback) {
  setTimeout(() => callback('hello world'), 100)
}
```

两个函数在功能上是等价的，第一个函数通过事件传递延迟函数的结束，第二个函数使用回调函数通知调用函数。实际上，真正的区别在于可读性、代码量、语法上，这里给出一些提示帮助决定是使用 `callbacks` 还是 `EventEmitter`：

* `callbacks` 在支持事件的不同类型上有限制，虽然可以把事件类型作为参数传递给回调函数，或者接受多个回调来区分多种类型事件，但是这样是不够优雅的，这种情况 `EventEmitter` 可以提供更简单的接口和更简洁的代码。
* 同一事件多次被触发或者从未触发，这种情况使用 `EventEmitter` 更好。
* 使用 `callbacks` 的 `API` 仅通知一个特定的回调函数，而使用 `EventEmitter` 可以让多个监听器接收同一个通知。

##### Combining callbacks and EventEmitter(结合回调和 EventEmitter)

某些场景下我们需要结合 `callbacks` 和 `EventEmitter` 来使用，这种模式在某种场景下是非常有用的：通过导出一个传统的异步函数作为主功能来实现最小接口原则，但同时通过返回 `EventEmitter` 提供更丰富的功能和控制。使用该模式的一个例子是 [node-glob](https://www.npmjs.com/package/glob) 模块，模块的主入口是它导出的一个函数：

```js
glob(pattern, [options], callback)
```

对于匹配到指定文件名匹配模式的文件列表，相关回调函数会被调用。同时，该函数返回 `EventEmitter`，它展现了当前进程的状态。例如，当成功匹配文件名时触发 `match` 事件，当文件列表全部匹配完毕时触发 `end` 事件，或者该进程被手动中止时触发 `abort` 事件:

```js
const glob = require('glob')
glob('data/*.txt', (error, files) =>
  console.log(`All files found: ${JSON.stringify(files)}`)
).on('match', match => console.log(`Match found: ${match}`))
```
