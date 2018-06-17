---
title: Node 应用性能优化
date: 2018-06-13 15:22:18
tags:
    - Node
    - 优化
---

在实际 `Node.js` 应用开发中可能会遇到性能上的瓶颈，在项目比较复杂的时候光阅读代码是很难发现问题所在的，这时候就需要有效的方法来发现瓶颈所在。

这里介绍一种简单的方法来帮助开发者洞察瓶颈所在，提升 `Node.js` 应用性能。

<!-- more -->

这个方法的主要目标就是度量在 `Node.js` 应用中执行每个函数所花费的 `CPU` 时间。当然也可以通过对内存的度量来检测内存泄漏，但在本篇文章只介绍对性能上面的优化。

## 度量

谷歌浏览器可以通过 `DevTools` 来记录每个函数信息和执行时间并将这些东西打进日志文件，来帮助开发者来发现瓶颈。而 `Node.js` 也有类似的内置工具，就是 `--prof` 配置，这个可以统计所有函数执行消耗的 `CPU` 时间片长度。

先来看个简单的例子：

```js
// file test.js
const cluster = require('cluster')
const cpuNums = require('os').cpus().length //我的机器上是8

let clusters = []
if (cluster.isMaster) {
  for (let i = 0; i < cpuNums; i++) {
    clusters[i] = cluster.fork()
  }
}

function add(n) {
  const n0 = parseInt(n / cpuNums)
  let s = 0,
    t = 0,
    start = Date.now()
  if (cluster.isMaster) {
    for (let i = 0; i < cpuNums; i++) {
      const worker = clusters[i]
      worker.process.send(`${i * n0}|${(i + 1) * n0}`)
      worker.on('message', s0 => {
        s += Number(s0)
        t += 1
        if (t === cpuNums) {
          console.log('time:', Date.now() - start)
          console.log('sum:', s)
          process.exit(0)
        }
      })
    }
  } else {
    process.on('message', n => {
      let [start, end] = n.split('|'),
        s = 0
      console.log(start, end)
      for (let i = Number(start); i < Number(end); i++) {
        if (i % 4 === 0) s += i
        if (i % 4 === 1) s -= i
        if (i % 4 === 2) s *= i
        if (i % 4 === 3) s /= i
      }

      console.log('end')
      process.send(s)
    })
  }
}

add(100000000)
```

上面的代码的主要功能是将一区间的整数分成几个部分分别对每 4 个数做加减乘除运算，最后将每个部分的结果加合得到最后的结果。可以发现上面代码为了提高性能而使用多个 `CPU` 核去执行计算任务，想充分利用 `CPU` 的性能，然而事实并非如此，在我的机器上运行结果如下：

```zsh
time: 1369
sum: -23641385.327095307
```

再来看看只用一个 `CPU` 来执行的代码和结果：

```js
// file test2.js
// code
// CPU核共8个，所以这里也分成8分，保持一致性
let s = 0, result = 0
let start = Date.now()

console.log('0 12500000')
for (let i = 0;i < 12500000;i++) {
  if (i % 4 === 0) s += i
  if (i % 4 === 1) s -= i
  if (i % 4 === 2) s *= i
  if (i % 4 === 3) s /= i
}
console.log('end')

result += s
s = 0

console.log('12500000 25000000')
for (let i = 12500000;i < 25000000;i++) {
  if (i % 4 === 0) s += i
  if (i % 4 === 1) s -= i
  if (i % 4 === 2) s *= i
  if (i % 4 === 3) s /= i
}
console.log('end')

...

console.log('87500000 1000000000')
for (let i = 87500000;i < 100000000;i++) {
  if (i % 4 === 0) s += i
  if (i % 4 === 1) s -= i
  if (i % 4 === 2) s *= i
  if (i % 4 === 3) s /= i
}
console.log('end')

result += s

console.log('time:', Date.now() - start)
console.log('sum:', result)
```

结果：

```zsh
time: 541
sum: -23641385.32709531
```

可以发现使用了多个 `CPU` 来执行反而更慢了，这是为啥？我们用上面说的配置来看一下问题出在哪。运行：

```zsh
node --prof test.js
```

执行完后出现一堆类似于 `isolate-*-v8-*.log` 的子进程 log(共 8 个，子进程 `worker`) 和一个主进程 log `isolate-*-v8.log`(主进程 `master`)。

## 分析

使用 `--prof-process` 配置运行 `NodeJS`，并提供上面生成的文件的路径。

```zsh
node --prof-process isolate-0x102802400-v8-81626.log
```

从上到下主要分为几个耗时分类，`Shared libraries`, `JavaScript`, `C++`, `Summary`, `C++ entry points`, `Bottom up (heavy) profile`:

### Shared libraries

`Node` 进程使用到的系统级动态链接库部分的时间消耗，会显示在这个分类下。

该分类的几列：

`ticks`：每个库所占用的 `ticks` 数量
`total`：每个库占用的 `ticks` 总量百分比
`nonlib`：这列在当前分类不适用，因为本来这里列的就都是类库时间消耗，`nonlib` 当然没有数据
`name`：动态链接库的文件位置

### JavaScript、C++、Summary

#### JavaScript

`JavaScript` 代码部分的时间消耗，包括了当前项目源代码部分的时间消耗和第三方 `node_modules` 的时间消耗。

#### C++

`Node` 进程在 `C++` 代码里的时间消耗，`Node` 本身是构建在 `V8` 引擎之上的，所以一些 `Node` 标准库里的 `API`，基本上都是 `C++` 时间消耗。当然这个分类也包含了一些作为第三方 `addon` 加载的插件的时间消耗。

#### Summary

将所有的分类的时间消耗总量都放在一起，形成一个直观的结果。

#### 列含义

- `ticks`：占用的 `ticks` 数量
- `total`：占用的 `ticks` 总量百分比
- `nonlib`：
  这列描述的是将 `Shared libraries` 所产生的时间消耗忽略之后，当前条目自身产生的时间消耗（`ticks`）所占的百分比
- `name`(每个 `name` 列实际函数名之前一般会有一个 `*` 或 `~`，星号表示该函数得到了优化，而波浪号则表示没有)：
  `JavaScript`：函数名，以及其在源代码中的位置
  `C++`：函数名，一般都是 `Node` 运行时和 `V8` 相关的函数
  `Summary`：分类名

### C++ entry points

这部分描述的是当逻辑从 `JS` 代码跨界到 `C++` 代码运行时，其中消耗的时间。

### Bottom up (heavy) profile

这部分是性能问题的暴露部分，一般看完 `Summery` 不想了解其细节就直接来看这部分是解决问题的最快方案。

和之前其他分类耗时部分不同的是，在这部分里按空行分隔的不同段落都是一个个单独的性能瓶颈点，每个段落的多行表示的是一个调用栈。

此外，这个部分的列内容也和之前的略有不同（主要是`parent`字段），`parent` 列的百分比意味着：表示上一行中的函数由当前行中的函数调用的百分比。

### 具体分析

先看 `Summary`：

```log
[Summary]:
  ticks  total  nonlib   name
  262   23.7%   23.7%  JavaScript
  778   70.3%   70.4%  C++
    38    3.4%    3.4%  GC
    1    0.1%          Shared libraries
    65    5.9%          Unaccounted
```

可以发现大部分性能浪费在执行 `C++` 代码上了，所以我们重点观察 `C++` 部分。

```log
[C++]:
  ticks  total  nonlib   name
  376   34.0%   34.0%  t bool v8::internal::StringToArrayIndex<v8::internal::StringCharacterStream>(v8::internal::StringCharacterStream*, unsigned int*)
    88    8.0%    8.0%  T v8::internal::String::ToNumber(v8::internal::Handle<v8::internal::String>)
    81    7.3%    7.3%  T v8::internal::Runtime_StringToNumber(int, v8::internal::Object**, v8::internal::Isolate*)
...
[C++ entry points]:
ticks    cpp   total   name
687   93.3%   62.1%  T v8::internal::Runtime_StringToNumber(int, v8::internal::Object**, v8::internal::Isolate*)
  28    3.8%    2.5%  T v8::internal::Builtin_HandleApiCall(int, v8::internal::Object**, v8::internal::Isolate*)
...
[Bottom up (heavy) profile]:
  Note: percentage shows a share of a particular caller in the total
  amount of its parent calls.
  Callers occupying less than 1.0% are not shown.

   ticks parent  name
    376   34.0%  t bool v8::internal::StringToArrayIndex<v8::internal::StringCharacterStream>(v8::internal::StringCharacterStream*, unsigned int*)
    376  100.0%    T v8::internal::Runtime_StringToNumber(int, v8::internal::Object**, v8::internal::Isolate*)
    376  100.0%      LazyCompile: *process.on.n /Users/hongchuanwang/Desktop/test2.js:29:31
    376  100.0%        LazyCompile: ~emitTwo events.js:124:17
    376  100.0%          LazyCompile: ~emit events.js:156:44
    376  100.0%            LazyCompile: ~emit internal/child_process.js:771:16

    155   14.0%  LazyCompile: *process.on.n /Users/hongchuanwang/Desktop/test2.js:29:31
    155  100.0%    LazyCompile: ~emitTwo events.js:124:17
    155  100.0%      LazyCompile: ~emit events.js:156:44
    155  100.0%        LazyCompile: ~emit internal/child_process.js:771:16
    155  100.0%          LazyCompile: ~_combinedTickCallback internal/process/next_tick.js:129:33
    155  100.0%            LazyCompile: ~_tickCallback internal/process/next_tick.js:151:25
```

34%的耗时都在 `StringToArrayIndex`，而调用该函数的都是 `Runtime_StringToNumber`，根据调用栈我们就发现代码中有一段 `for (let i = Number(start); i < Number(end); i++)` 发现问题所在，是循环的时候多次处理了 `end`，所以只需要将 `Number(end)` 放在循环外处理就行了：

```js
start = Number(start)
end = Number(end)
for (let i = start; i < end; i++) {
  if (i % 4 === 0) s += i
  if (i % 4 === 1) s -= i
  if (i % 4 === 2) s *= i
  if (i % 4 === 3) s /= i
}
```

改完再分析结果如下：

```log
[Summary]:
  ticks  total  nonlib   name
  123   53.0%   53.2%  JavaScript
  103   44.4%   44.6%  C++
    34   14.7%   14.7%  GC
    1    0.4%          Shared libraries
    5    2.2%          Unaccounted
...
[C++ entry points]:
   ticks    cpp   total   name
     40   65.6%   17.2%  T v8::internal::Builtin_HandleApiCall(int, v8::internal::Object**, v8::internal::Isolate*)
     13   21.3%    5.6%  T v8::internal::Runtime_CompileLazy(int, v8::internal::Object**, v8::internal::Isolate*)
...
[Bottom up (heavy) profile]:
  Note: percentage shows a share of a particular caller in the total
  amount of its parent calls.
  Callers occupying less than 1.0% are not shown.

   ticks parent  name
    119   51.3%  LazyCompile: *process.on.n /Users/hongchuanwang/Desktop/test2.js:29:31
    119  100.0%    LazyCompile: ~emitTwo events.js:124:17
    119  100.0%      LazyCompile: ~emit events.js:156:44
    119  100.0%        LazyCompile: ~emit internal/child_process.js:771:16
    119  100.0%          LazyCompile: ~_combinedTickCallback internal/process/next_tick.js:129:33
    119  100.0%            LazyCompile: ~_tickCallback internal/process/next_tick.js:151:25
...
```

可以发现耗时的 `StringToNumber` 已经没有了，`CPU` 的 `tick` 也只有119了。
