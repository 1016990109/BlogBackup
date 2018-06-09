---
title: 《Node.js 设计模式》读书笔记 第三章
date: 2018-06-07 10:39:52
tags:
    - 读书笔记
categories:
    - 学习
---

# Asynchorous Control Flow Patterns with Callbacks(使用回调的异步控制流模式)

异步的代码使得难以预测语句的执行顺序，所以在一些场景(比如遍历一些文件，执行一系列任务等等)下，这就要求开发者去使用一些方法或技术来防止编写出低效和难以阅读的代码。

## The difficulties of asynchonous programming(异步编程的困难)

> `KISS` 原则：Keep It Simple, Stupid，注重简约。

匿名函数的闭包和原地定义使得开发者不用跳去另外一个地方写代码，这样编程就变得非常顺利。(比先去定义一个函数再回来引入要简单多了)这很好地体现了 `KISS` 原则，因为它是简单的，保持了代码编写的流(或理解为顺序吧)，开发时间少。但是当嵌套的层次变得多了起来之后，可维护性、复用性、模块性就被破坏了

### Creating a simple web spider(创建一个简单的 web 爬虫)

```js
// file spider.js
const request = require('request')
const fs = require('fs')
const mkdirp = require('mkdirp')
const path = require('path')
const utilities = require('./utilities')

function spider(url, callback) {
  const filename = utilities.urlToFilename(url)
  fs.exists(filename, exists => {
    //[1]
    if (!exists) {
      console.log(`Downloading ${url}`)
      request(url, (err, response, body) => {
        //[2]
        if (err) {
          callback(err)
        } else {
          mkdirp(path.dirname(filename), err => {
            //[3]
            if (err) {
              callback(err)
            } else {
              fs.writeFile(filename, body, err => {
                //[4]
                if (err) {
                  callback(err)
                } else {
                  callback(null, filename, true)
                }
              })
            }
          })
        }
      })
    } else {
      callback(null, filename, false)
    }
  })
}
```

上述函数执行下面的任务：

1.  通过查看相关的文件已经是否被创建来检查 `URL` 是不是已经被下载过了:

```js
fs.exists(filename, exists => ...
```

2.  文件没有找到，则下载 `URL`:

```js
request(url, (err, response, body) => ...
```

3.  保证包含该文件的文件夹存在:

```js
mkdirp(path.dirname(filename), err => ...
```

4.  最后将 `HTTP` 响应内容写入文件系统的文件中:

```js
fs.writeFile(filename, body, err => ...
```

使用:

```js
spider(process.argv[2], (err, filename, downloaded) => {
  if (err) {
    console.log(err)
  } else if (downloaded) {
    console.log(`Completed the download of "${filename}"`)
  } else {
    console.log(`"${filename}" was already downloaded`)
  }
})
```

### The callback hell(回调地狱)

从上面的 `spider()` 函数中可以看到，尽管我们非常直接清晰地实现这个算法，但是代码还是有很多层的缩进导致难以阅读。

当然将上面的逻辑使用同步代码的方式实现会更加直接，而且出错的概率也会变得更小，但是阻塞就会使得效率更低，而且使用异步 `CPS` 风格也是另外一种尝试了。

这种大量的闭包和内联回调函数定义导致代码变得不可读和不可控的场景称为回调地狱:

```js
asyncFoo(err => {
  asyncBar(err => {
    asyncFooBar(err => {
      //...
    })
  })
})
```

上面这种代码一个显而易见的问题就是降低了可读性，当层次变得很多的时候会看不清一个函数是什么时候结束的或者另一个函数是什么时候开始的。

另一个问题就是变量名的重叠，有时候我们不得不使用相似的甚至相同的名字来描述一个内容，例如错误处理中使用 `err1` 、`err2`、`err3` 表示错误，甚至是直接使用相同的名字如 `err`，这些都不是好的实现，并且会导致混淆，提高缺陷发生的概率。

还有一点需要注意，虽然闭包在性能和内存上代价较小，但是这可能导致不易识别的内存泄漏，因为被一个活动的闭包持有的上下文引用是不会被垃圾回收的。

> 想知道闭包怎么在 `V8` 中工作的可以看[这里](https://mrale.ph/blog/2012/09/23/grokking-v8-closures-for-fun.html)

## Using plain JavaScript(使用纯 JavaScript)

### Callback decipline(回调准则)

写异步函数的第一准则就是不要滥用闭包。

下面是一些减少嵌套层次的原则：

* 尽快返回。根据上下文，使用 `return`、`continue` 或 `break`，以便立即退出当前代码块，而不是写完整的 `if...else` 的语句。
* 给函数命名，将中间结果作为参数传递。
* 模块化代码。尽可能地将代码分成更小、更可复用的函数。

### Applying the callback decipline(应用回调准则)

我们来应用这些准则来修复上面的 `spider` 应用。

第一步：移除 `else` 语句，发现错误后立即返回，可以发现很容易就较少了嵌套的层级了(少了 `else` 那一层级)。

```js
if (err) {
  return callback(err)
}
//code to execute when there are no errors
```

第二步：识别可复用的代码，独立出来为一个函数，这里为写入字符串到一个文件中：

```js
function saveFile(filename, contents, callback) {
  mkdirp(path.dirname(filename), err => {
    if (err) {
      return callback(err)
    }
    fs.writeFile(filename, contents, callback)
  })
}
```

同样的也可以独立出一个 `download(url, filename, callback)` 函数来下载 `URL` 内容。

最后一步：整合上面的两步，修改 `spider()` 函数：

```js
function spider(url, callback) {
  const filename = utilities.urlToFilename(url)
  fs.exists(filename, exists => {
    if (exists) {
      return callback(null, filename, false)
    }
    download(url, filename, err => {
      if (err) {
        return callback(err)
      }
      callback(null, filename, true)
    })
  })
}
```

可以发现只是简单地重新组织了代码，嵌套的层级就降低了很多，代码也变得更易读了。

> 其中的 `saveFile()` 和 `download` 还可以考虑导出给其他模块使用，增加了复用性。

### Sequential execution(顺序执行)

当需要顺序执行一组任务时(比如先对数据预处理，接着在按照步骤一步步处理数据等等)，尽管使用同步代码容易实现，但是在使用异步 `CPS` 风格来做时就可能导致回调地狱了。

#### Executing a known set of tasks in sequence(顺序执行一个已知的任务集合)

直接上代码：

```js
function task1(callback) {
  asyncOperation(() => {
    task2(callback);
  });
}

function task2(callback) {
  asyncOperation(result() => {
    task3(callback);
  });
}

function task3(callback) {
  asyncOperation(() => {
    callback(); //finally executes the callback
  });
}

task1(() => {
  //executed when task1, task2 and task3 are completed
  console.log('tasks 1, 2 and 3 executed');
});
```

上面演示了在一个任务中，当异步操作完成时如何调用下一个任务，这告诉我们处理异步代码不一定需要闭包。

#### Sequential iteration(顺序迭代)

上面示例中，我们是知道有多少任务要执行的，那么当任务数量和具体任务不清楚的时候该怎么办呢，我们就不能硬编码任务执行顺序了，得动态地生成。

##### Web spider version 2(Web 爬虫第 2 版)

为了显示顺序迭代的例子，让我们为 `Web` 爬虫应用程序引入一个新功能：我们现在想要递归地下载网页中的所有链接。

第一步是修改我们的 `spider()` 函数，以便通过调用一个名为 `spiderLinks()` 的函数触发页面所有链接的递归下载。

此外，我们现在尝试读取文件，而不是检查文件是否已经存在，并开始爬取其链接。这样，我们就可以在中断爬虫后恢复爬虫而不需要继续下载。最后还有一个变化是传递一个新的参数 `nesting`，用来限制递归深度。结果代码如下：

```js
function spider(url, nesting, callback) {
  const filename = utilities.urlToFilename(url);
  fs.readFile(filename, 'utf8', (err, body) => {
    if (err) {
      if (err.code! == 'ENOENT') {
        return callback(err);
      }
      return download(url, filename, (err, body) => {
        if (err) {
          return callback(err);
        }
        spiderLinks(url, body, nesting, callback);
      });
    }
    spiderLinks(url, body, nesting, callback);
  });
}
```

##### Sequential crawling of links(顺序爬取连接)

```js
function spiderLinks(currentUrl, body, nesting, callback) {
  if (nesting === 0) {
    return process.nextTick(callback)
  }

  let links = utilities.getPageLinks(currentUrl, body) //[1]
  function iterate(index) {
    //[2]
    if (index === links.length) {
      return callback()
    }

    spider(links[index], nesting - 1, function(err) {
      //[3]
      if (err) {
        return callback(err)
      }
      iterate(index + 1)
    })
  }
  iterate(0) //[4]
}
```

1.  我们使用 `utilities.getPageLinks()` 函数获取页面中包含的所有链接的列表。此函数仅返回指向相同主机名的链接。
2.  我们使用 `iterate()` 本地函数来遍历链接，该函数需要下一个链接的索引进行分析。在这个函数中，我们首先要检查索引是否等于链接数组的长度，如果等于则是迭代完成，在这种情况下我们立即调用 `callback()` 函数，因为这意味着我们处理了所有的项目。
3.  这时，已准备好处理链接。我们减少嵌套层级(`nesting - 1`)后调用 `spider()`，然后当操作完成后继续下一个迭代(`index + 1`)。
4.  调用 `iterate(0)` 来开始迭代。

现在这个 `spider` 已经可以递归的爬取网页的链接了。中断(`ctrl + c`)后再次启动也可以继续上次的任务。

##### The pattern(迭代模式)

```js
function iterate(index) {
  if (index === tasks.length) {
    return finish()
  }
  const task = tasks[index]
  task(function() {
    iterate(index + 1)
  })
}

function finish() {
  // 迭代完成的操作
}

iterate(0)
```

上面表示了异步任务需要按顺序执行时的一个通用模式，可以在集合的元素或通常的任务列表上按顺序异步迭代。

> 注意，当 `task` 是同步任务的时候，那就是一个同步递归操作了，这可能会造成栈溢出。

### Parallel execution(并行执行)

在 `Node.js` 中，我们只能并行执行异步操作，因为它们的并发性由非阻塞 `API` 在内部处理。在 `Node.js` 中，同步阻塞操作不能并行运行，除非它们被插入异步操作中或使用 `setTimeout` 之类的做延迟。

#### Web spider version 3(Web 爬虫第 3 版)

现在需要并行地下载网页的内容，只需要略做修改：

```js
function spiderLinks(currentUrl, body, nesting, callback) {
  if (nesting === 0) {
    return process.nextTick(callback)
  }
  const links = utilities.getPageLinks(currentUrl, body)
  if (links.length === 0) {
    return process.nextTick(callback)
  }
  let completed = 0,
    hasErrors = false
  function done(err) {
    if (err) {
      hasErrors = true
      return callback(err)
    }
    if (++completed === links.length && !hasErrors) {
      return callback()
    }
  }
  links.forEach(link => {
    spider(link, nesting - 1, done)
  })
}
```

`forEach` 同时爬取链接列表中的链接，而不用等待前一个爬取完了才开始爬取下一个，增加一个 `completed` 变量记录已经爬取完的链接数，当 `completed` 等于链接个数时就说明所有的链接都已爬取完了，就可以调用最后的回调函数了。或者说当中间出错了也会立即执行回调函数返回。

#### The pattern(并行模式)

```js
const tasks = [
  /* ... */
]
let completed = 0
tasks.forEach(task => {
  task(() => {
    if (++completed === tasks.length) {
      finish()
    }
  })
})

function finish() {
  // 所有任务执行完成后调用
}
```

同时启动一系列任务，通过记录任务回调函数完成的个数来判断所有任务是否都完成了。

#### Fixing race conditions with concurrent tasks(修复并发任务中竞争条件)

在传统的多线程中处理竞争条件通常是锁、互斥条件、信号量和监视器，这些是多线程语言并行化的最复杂的方面之一，对性能也有很大的影响。但是 `Node.js` 就不同了，它本身就是运行在一个单线程上，这就变得简单多了。但是这不意味着就没有竞争条件了，相反还很普遍。就拿上面的爬虫例子来说，如果有两个爬虫同时运行，都在操作同一个 `URL` 时，`fs.readFile` 都读取不到文件，那么两个爬虫就同时去下载这个链接，这就导致了会同时写入内容到同一个文件中。修复办法很简单，在两个 `spider` 外面定义一个共享的变量记录爬取的链接，如果爬取过则另外一个 `spider` 就不再爬取：

```js
const spidering = new Map();
   function spider(url, nesting, callback) {
     if(spidering.has(url)) {
       return process.nextTick(callback);
     }
     spidering.set(url, true);
//...
```

### Limited parallel execution(有限制的并行执行)

不对并行任务做控制的话很容易导致昂贵的开销，例如同时读取很多文件会导致系统资源不足，在 web 应用中还可能导致 `DoS` 攻击，所以限制同一时间任务的执行数是非常重要的。

#### Limiting the concurrency(限制并行数)

```js
const tasks = ...
let concurrency = 2, running = 0, completed = 0, index = 0;
function next() {
  while(running < concurrency && index < tasks.length) {
    task = tasks[index++];
    task(() => {
      if(completed === tasks.length) {
        return finish();
      }
      completed++, running--;
      next();
});
running++; }
} next();
function finish() {
  //all tasks finished
}
```

上面所展示的模式就可以限制同时最大任务数为 2，启动时同时开始最大任务数的任务，之后每当有一个任务结束时，就会从剩下的任务中挑出一个任务开始执行，保持在限制范围内的最多任务同时进行。

#### Globally limiting the concurrency(全局地限制并发数)

> `Node.js` 0.11 版本以前是默认限制同一个主机名下最大 `HTTP` 连接数为 5 的，这个可以满足我们的需要。但是在之后的版本就取消了这个默认限制了。

##### Queues to rescue(队列来拯救)

我们需要的是限制同时下载的任务数，可以使用队列来解决：

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
      task(() => {
        this.running--
        this.next()
      })
      this.running++
    }
  }
}
```

通过 `pushTask` 添加任务，然后启动 `next` 来开始执行任务，`next` 会自动识别是否任务数达到上限。

##### Web spider version(Web 爬虫第 4 版)

使用上面的队列来更改我们的爬虫程序：

```js
const TaskQueue = require('./taskQueue')
const downloadQueue = new TaskQueue(2)

function spiderLinks(currentUrl, body, nesting, callback) {
  if (nesting === 0) {
    return process.nextTick(callback)
  }
  const links = utilities.getPageLinks(currentUrl, body)
  if (links.length === 0) {
    return process.nextTick(callback)
  }
  let completed = 0,
    hasErrors = false
  links.forEach(link => {
    downloadQueue.pushTask(done => {
      spider(link, nesting - 1, err => {
        if (err) {
          hasErrors = true
          return callback(err)
        }
        if (++completed === links.length && !hasErrors) {
          callback()
        }
        done()
      })
    })
  })
}
```

## The async library(async 库)

### Sequential execution(顺序执行)

`async` 库可以在实现复杂的异步控制流程时很大程度上帮助我们，但是选择正确的方法来处理具体问题是一个问题。顺序执行就有大约有 20 种方法，`eachSeries()`, `mapSeries()`, `filterSeries()`等等。

#### Sequential execution of a known set of tasks(已知任务的顺序执行)

```js
async.series(tasks, [callback])
```

`series` 顺序执行一组任务，在所有任务调用回调函数 `callback`。而每一个 `task` 只是个接受回调函数的函数 `function task(callback) {}`，当某一个任务回调时发送了错误，那么 `async` 会停止后面的任务，直接到最后的回调函数。

#### Sequential iteration(顺序迭代)

```js
async.eachSeries(iterable, fn(item, callback), [callback])
```

遍历一个可遍历的对象，顺序执行每一个元素对应的函数，所有元素对应的函数执行完后调用最后的回调函数。

### Parallel execution(并行执行)

`each()`，`map()`，`filter()`，`reject()`，`detect()`，`some()`，`every()`，`concat()`，`parallel()`，`applyEach()` 和 `times()` 都是并行执行的 `async` 的方法。

### Limited parallel execution(限制并行执行)

类似于 `async.queue(worker, concurrency)` 来限制同时执行的任务数。
