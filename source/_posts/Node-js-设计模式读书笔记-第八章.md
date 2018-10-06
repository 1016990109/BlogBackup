---
title: 《Node.js 设计模式》读书笔记 第八章
date: 2018-09-25 11:54:47
tags:
  - 读书笔记
---

# Universal JavaScript for Web Applications(Web 应用的通用 JavaScript)

## Sharing code with browser(和浏览器共享代码)

### Universal Module Definition(UMD)

一般来说需要同时满足浏览器和服务端要求，最常用的就是使用 `UMD` 规范了，`umd` 是 `AMD` 和 `CommonJS` 的糅合。先判断是否支持 `AMD`（通过判断 `define` 是否存在），存在则使用 `AMD` 方式加载模块。再判断是否支持 `Node.js` 的模块（`exports`）是否存在，存在则使用 `Node.js` 模块模式。如果两个都不存在，那么可能就使用全局变量来定义了(一般根据传入的 `root`，可能是执行的 `this`)。

<!-- more -->

给个例子：

```js
;(function(root, factory) {
  if (typeof define === 'function' && define.amd) {
    define(['mustache'], factory)
  } else if (typeof module === 'object' && typeof module.exports === 'object') {
    var mustache = require('mustache')
    module.exports = factory(mustache)
  } else {
    root.UmdModule = factory(root.Mustache)
  }
})(this, function(mustache) {
  var template = '<h1>Hello <i>{{name}}</i></h1>'
  mustache.parse(template)
  return {
    sayHello: function(toWhom) {
      return mustache.render(template, { name: toWhom })
    }
  }
})
```

### ES2015 Modules

`ES2015` 模块充分利用了 `AMD` 和 `CommonJS` 的优点：

- 像 `CommonJS` 一样，`ES2015` 模块提供了简洁的语法，单独使用 `export` 导出模块而不用 `module.export = ...`，并且支持循环依赖。
- 像 `AMD` 一样，`ES2015` 模块直接就支持异步模块加载和可配置模块加载。

## Introducing Webpack(Webpack 介绍)

`Webpack` 帮助我们将用 `Node.js` 模块规范些的代码编译成能在浏览器上运行的代码，解决了 `require` 在浏览器中的问题(浏览器是没有 `require` 函数的)。

上面的 `UMD` 代码变得更简洁：

```js
var mustache = require('mustache')
var template = '<h1>Hello <i>{{name}}</i></h1>'
mustache.parse(template)
module.exports.sayHello = function(toWhom) {
  return mustache.render(template, { name: toWhom })
}
```

### 优点

1. 自动提供 `Node.js` 核心模块的浏览器兼容，如 `http`、`assert`、`events` 等等。`fs` 模块是不支持的。
2. 如果不自动支持，可以手动引入兼容包，也就是 `polyfill` 这类的。
3. 能从不同模块生成 `bundles`。
4. 对源文件可做额外的操作，利用 `loaders` 和 `plugins` 来完成。
5. 可以非常容易地通过任务执行管理器(例如 `Gulp` 或 `Grunt`)来执行 `Webpack`。
6. 可以管理除 `JS` 文件外的文件预处理，如样式表文件、图片、字体、模板等等。
7. 配置 `Webpack` 来分离依赖，组织代码为多个 `chunk`，做到动态加载，等需要的时候再加载。

### Using ES2015 with Webpack(在 Webpack 中使用 ES2015)

前面有说到利用 `loaders` 和 `plugins` 来做额外的做操作，这里就利用 `babel-loader` 来将 `ES6` 的语法转为 `ES5`，只需要在 `webpack.config.js` 中配置：

```js
const path = require('path')
module.exports = {
  entry: path.join(__dirname, 'src', 'main.js'),
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'bundle.js'
  },
  module: {
    loaders: [
      {
        test: path.join(__dirname, 'src'),
        loader: 'babel-loader',
        query: {
          presets: ['es2015']
        }
      }
    ]
  }
}
```

## Fundamentals of cross-platform development(跨平台开发基础)

为不同平台开发最主要的一个问题就是如何共享一个组件的公共部分(除去平台特定的部分)。

### Runtime code branching(运行时代码分支)

当需要对平台定制化代码时，很容易就想到要根据当前的环境来加载不同的模块：

```js
if (typeof window !== 'undefined' && window.document) {
  require('clientModule')
} else {
  require('serverModule')
}
```

但是这有个问题就是，所有的模块都会被打包进去，会导致最终代码的大小太大。

### Build-time code branching(编译时代码分支)

使用 `Webpack` 可以在编译时就将代码进行分割，只讲需要用的代码打包进最终包。

为了完成编译时代码分支，我们使用了两个内置的插件 `DefinePlugin` 和 `UglifyJsPlugin` 连接的管道。

```js
//main.js
if (typeof __BROWSER__ !== 'undefined') {
  console.log('Hey browser!')
} else {
  console.log('Hey Node.js!')
}
```

```js
//webpack.config.js
const path = require('path')
const webpack = require('webpack')

const definePlugin = new webpack.DefinePlugin({
  "__BROWSER__": true
})

const uglifyJsPlugin = new webpack.UglifyJsPlugin({
  beautify: true,
  dead_code: true
})

module.exports = {
  entry: path.join(__dirname, 'src', 'main.js'),
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'bundle.js'
  },
  plugins: [definePlugin, uglifyJsPlugin]
}
```

`DefinePlugin` 用来定义一个在源代码中能访问的一个全局变量，`UglifyJsPlugin` 用来减少最终生成的代码体积(因为去除了**无用代码**，这是在运行时代码分支里存在的问题，当然同时会去除掉一些空格之类的空白字符以及替换变量名等等)。

> 需要注意的是，`DefinePlugin` 其实是在编译的时候找到源码中所有的 `__BROWSER__` 变量，然后将其替换为 `true`，和普通地声明一个全局变量不同。

### Module swapping(模块交换)

有点时候我们会已知某些模块是不需要的，需要用其他模块来代替，这个时候再构建过程中去替换。

`Webpack` 使用 `NormalModuleReplacementPlugin` 来完成这件事，匹配对应的正则表达式，然后用定义的模块替换掉匹配的模块。

这样有时候就很容易地将 `Node.js` 环境下运行的代码转换成浏览器运行的代码。
