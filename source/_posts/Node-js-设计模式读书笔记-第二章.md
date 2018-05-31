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
  };
  const privateBar = [];
  const exported = {
    publicFoo: () => {
      // ...
    },
    publicBar: () => {
      // ...
    }
  };
  return exported;
})();
console.log(module);
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
       })(module, module.exports, require);`;
  eval(wrappedSrc);
}
```

模块的源码被包装入一个函数，并且是使用模块模式的。区别在于传递了一些参数到模块中，实际上就是 `module`, `exports`, `require`。`exports` 参数被初始化为 `module.exports`。

>注意：上面只是个示例，其实很少使用 `eval` 来执行源码，这可能导致注入攻击，使用 `eval` 要十分谨慎。

现在通过实现 `require()` 函数来看看这都些变量中的包含了什么内容：

```js
const require = (moduleName) => {
  console.log(`Require invoked for module: ${moduleName}`);
  const id = require.resolve(moduleName);
  if (require.cache[id]) {
    return require.cache[id].exports;
  }
  //模块元数据
  const module = {
    exports: {},
    id: id
  };
  //更新缓存
  require.cache[id] = module;
  //加载模块
  loadModule(id, module, require);
  //返回导出的变量
  return module.exports;
};
require.cache = {};
require.resolve = (moduleName) => {
  /* 通过模块名作为参数resolve一个完整的模块 */
};
```

上面函数模拟了原生 `require()` 函数，并不能准确完美地反应真实的行为，但是却能帮助我们理解一个模块是怎么被定义和加载的：

1. 一个模块的名字作为输入被接收，我们需要做的第一件事就是找到这个模块的路径(我们称之为`id`)，这个依靠 `require.resolve()`来完成。
2. 如果模块过去被加载过，那它应该存在于缓存。这种情况下我们直接返回就行。
3. 如果模块尚未加载，我们将初始化首次加载模块环境。具体来说就是，创建一个模块(`module`)对象，其中包含一个 `exports` (被初始化为空的对象字面量`{}`)属性。该属性将被模块的代码用于导出模块的公共 `API`。
4. 模块被缓存。
5. 像前面所看到的一样，源代码从文件中被加载，接着被执行。我们给模块提供一个刚才创建的 `module` 对象和一个 `require()` 函数的引用。模块通过修改或替换 `module.exports` 来提供公共 `API`。
6. 最后，包含公共 `API` 的 `module.exports` 返回给调用者。

##### Defining a module(定义一个模块)

让我们看看怎么定义一个模块：

```js
//加载另一个依赖
const dependency = require('./anotherModule');
//一个私有函数
function log() {
  console.log(`Well done ${dependency.username}`);
}
//API 被导出给外部用
module.exports.run = () => {
  log(); 
};
```

除了 `module.exports` 的内容其他都是私有的，当模块被加载的时候这个变量的内容被返回且被缓存。

##### Defining globals(定义全局内容)

即使在模块中声明的所有变量和函数都在其本地范围内定义，仍然可以定义全局变量。事实上，模块系统公开了一个名为 `global` 的特殊变量。分配给此变量的所有内容将会被定义到全局环境下。

>注意：污染全局变量是不好的，模块化的优势就不在了，所以只有当你真的需要用的时候再用吧！

##### module.exports vs exports

`exports` 只是 `module.exports` 的一个引用，所以在 `exports` 中添加新属性是有效的，能更新 `module.exports` 的内容，而对 `exports` 重新赋值则不会更新 `module.exports`，只是让 `exports` 指向了另一个对象；但是对 `module.exports` 重新赋值就是实实在在地更改了 `module` 了，是能起作用的。

```js
//有效
exports.foo = () => {console.log("hello")}

//无效
exports = {
  foo: () => {
    console.log("hello")
  }
}

//有效
module.exports = {
  foo: () => {
    console.log("hello")
  }
}
```

##### The require function is synchronous(require函数是同步的)

原生的 `require()` 函数也是同步的，所以对 `module.exports` 的赋值操作也是要同步的。下面这种代码就是错误的：

```js
setTimeout(() => {
  module.exports = function() {
    // ...
  };
}, 100);
```

这就限制了我们绝大多数情况下都是使用同步的代码定义模块，这就是 `Node.js` 核心库为一些异步函数提供可选的同步的 `API` 的原因。

如果需要在模块初始化过程中使用异步方法，那么可以返回一个未初始化的模块，让使用者之后去初始化这个模块，这就导致了 `require` 不能保证模块被立即使用。

出于好奇，你可能想知道为什么 `Node.js` 早期是有异步的 `require()` 函数后来又被移除了，这是因为在初始化的过程中处理异步的I/O带来的复杂性比优势大太多了。

##### The resolve algorithm(resolve算法)

为了解决[依赖地狱](https://zh.wikipedia.org/wiki/相依性地狱)问题，`Node.js` 根据模块的被加载的位置来加载不同版本的模块，这些理念也被运用到 `npm` 和 `require` 的 `resolve` 算法中。

`resolve()` 接收 `moduleName` 作为参数，并返回模块的完整路径。

`resolve` 算法的三个主要分支：

1. **File modules**(文件模块)：模块名是 `/` 开头认为是绝对路径，以 `./` 开头则认为是相对当前使用 `require` 的模块的路径。
2. **Core modules**(核心模块)：模块名不以 `/` 或 `./` 开头则优先从核心库开始查找。
3. **Package modules**(包模块)：核心库没有查找到时，再从当前目录的 `node_modules` 中查找相应的模块，没有则继续往上层的 `node_modules` 中找直到系统的根目录。

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

可以发现 `depA`, `depB`, `depC` 都有它们自己的依赖，所以同样使用 `require('depA')`，在不同的地方加载就会加载不同的模块，如：

* 在 `/myApp/foo.js` 中调用的 `require('depA')` 会加载 `/myApp/node_modules/depA/index.js`
* 在 `/myApp/node_modules/depC/foobar.js` 中调用的 `require('depA')` 会加载 `/myApp/node_modules/depC/node_modules/depA/index.js`

`resolve` 算法是 `Node.js` 依赖关系管理的核心部分，它的存在使得即便应用程序拥有成百上千包的情况下也不会出现冲突和版本不兼容的问题。

当我们调用 `require()` 时，解析算法对我们是透明的。然而，仍然可以在任何模块中通过调用 `require.resolve()` 使用该算法。

##### The module cache(模块缓存)
未完待续...
