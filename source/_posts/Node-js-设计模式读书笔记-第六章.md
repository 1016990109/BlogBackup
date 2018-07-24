---
title: 《Node.js 设计模式》读书笔记 第六章
date: 2018-07-04 10:37:30
tags:
    - 读书笔记
    - 设计模式
---

# Design Patterns(设计模式)

## Factory(工厂)

### A generic interface for creating objects(创建对象的通用接口)

调用一个工厂，而不是直接使用 `new` 运算符或 `Object.create()` 从一个原型创建一个新的对象，在很多方面是非常方便和灵活的。

<!-- more -->

首先最重要的是，工厂允许我们将对象创建与实现分离开来；从本质上讲，一个工厂包装了一个新实例的创建，给了我们更多的灵活性和怎么创建的控制权。

在工厂内部，我们可以使用闭包，使用原型和 `new` 运算符，使用 `Object.create()` 创建新实例，甚至根据特定条件返回不同的实例。对于对象的使用者而言，是完全不知道这个实例是怎么进行创建的。

```js
function createImage(name) {
  if (name.match(/\.jpeg$/)) {
    return new JpegImage(name)
  } else if (name.match(/\.gif$/)) {
    return new GifImage(name)
  } else if (name.match(/\.png$/)) {
    return new PngImage(name)
  } else {
    throw new Exception('Unsupported format')
  }
}
const image = createImage('photo.jpeg')
```

### A mechanism to enforce encapsulation(强制封装的机制)

> 封装需要做的是隐藏对象信息，外部代码只能通过暴露的公开接口修改对象而不能直接作用于对象，这又要叫做信息隐藏，和继承、多态、抽象一起都是面向对象的基本原则。

在 `JavaScript` 中，没有权限修饰符（不能声明私有变量），所以强制封装的唯一方法是通过函数作用域和闭包。

```js
function createPerson(name) {
  const privateProperties = {}
  const person = {
    setName: name => {
      if (!name) throw new Error('A person must have a name')
      privateProperties.name = name
    },
    getName: () => {
      return privateProperties.name
    }
  }
  person.setName(name)
  return person
}
```

上面代码中创建了两个对象：`person` 是通过工厂返回的公开接口；`privateProperties` 是只能通过 `person` 公开接口更改的私有属性。

> 工厂只是我们创建私有成员变量的技术之一，事实上，也有很多其它的方法定义私有成员变量。在构造函数中定义私有变量；使用约定，用下划线 `_` 或美元符号 `$`（实际上并不能阻止从外部访问内部成员）的属性名称前缀；使用`ES2015 WeakMaps`。更加详细的可以看看 `Mozilla` 的 [Private Properties](https://developer.mozilla.org/en-US/docs/Archive/Add-ons/Add-on_SDK/Guides/Contributor_s_Guide/Private_Properties) 的文章。

### Build a simple code profiler(构建一个简单的 profiler)

```js
class Profiler {
  constructor(label) {
    this.label = label
    this.lastTime = null
  }
  start() {
    this.lastTime = process.hrtime()
  }
  end() {
    const diff = process.hrtime(this.lastTime)
    console.log(
      `Timer "${this.label}" took ${diff[0]} seconds and ${diff[1]}
           nanoseconds.`
    )
  }
}
```

我们使用这样一个 `profiler` 来记录每个程序的执行时间，在生产环境中就会产生大量的输出，我们想做的可能是将这些信息重定向到另一个源，或者在生产环境下完全禁用，如果直接通过 `new` 来创建 `Profiler` 对象的话需要做一些额外的逻辑以便在不同的逻辑间来切换。而使用工厂模式就可以很好地解决这个问题，根据传入参数的不同创建不同逻辑的 `Profiler` 对象：

```js
module.exports = function(label) {
  if (process.env.NODE_ENV === 'development') {
    return new Profiler(label) // [1]
  } else if (process.env.NODE_ENV === 'production') {
    return {
      // [2]
      start: function() {},
      end: function() {}
    }
  } else {
    throw new Error('Must set NODE_ENV')
  }
}
```

### Composable factory functions(可组合的工厂函数)

可组合的工厂函数，它代表了一种特定类型的工厂函数，可以“组合”在一起构建新的更强大的工厂函数。

可组合工厂函数可以使用实现了 [Stamp 规范](https://github.com/stampit-org/stamp-specification) 的 `npm` 库 [stampit](https://www.npmjs.com/package/stampit)。详细的例子可直接查看官方文档 [stampit](https://stampit.js.org/essentials/overview)。

> `Stamp` 是一个可组合的工厂函数(`composable factory function`)，根据描述符返回对象实例。它有一个 `compose` 的方法，该方法使用当前的 `Stamp` 作为一个基础，将其他的 `Composable` 组合进来，返回一个新的 `Stamp`。可以通过 `staticProperties` 属性来重写 `compose` 方法，方法在[这里](https://github.com/stampit-org/stamp-specification#overriding-compose-method)。

### In the wild(实际应用)

很多的 `Node.js` 的库都有使用工厂模式，利用工厂返回实例，这里就不一一举例说明了，比较有意思的可以看看 `Node.js` 的核心模块 `HTTP` (`http.createServer()`创建实例)，使用 `Stamp` 规范的包，如 [react-stampit](https://www.npmjs.com/package/react-stampit) 可以轻松地组合组件的功能。

## Revealing constructor(揭露构造器)

其实简单说就是将要暴露的接口返回（`return`）出去。

我们分析一下 `Promise` 的构造函数：

```js
const promise = new Promise(function(resolve, reject) {
  // ...
})
```

`Promise` 接收一个函数作为构造器的参数，这个函数被称为执行函数，在 `Promise` 内部被调用，它提供了一种暴露可以被外界调用的 `resolve` 和 `reject` 方法的机制去修改 `Promise` 内部的状态。这样的好处是，只有构造器才有对 `resolve` 和 `reject` 的访问权限，一旦 `Promise` 对象被创建，`resolve` 和 `reject` 就能安全地传递，在其他地方是调用不了的。

### A read-only event emitter(一个只读的事件触发器)

使用这个模式我们创建一个只读的 `event emitter`:

```js
const EventEmitter = require('events')
module.exports = class Roee extends EventEmitter {
  constructor(executor) {
    super()
    const emit = this.emit.bind(this)
    this.emit = undefined
    executor(emit)
  }
}
```

可以发现，现在之后通过 `executor` 传入的参数才能获取到 `emit` 的访问权限了，关键就在于 `this.emit = undefined` 这一语句使得从外面不能访问 `emit` 方法了，只能通过传入构造器的函数中的参数访问，使用方式像这样：

```js
const Roee = require('./roee')
const ticker = new Roee(emit => {
  let tickCount = 0
  setInterval(() => emit('tick', tickCount++), 1000)
})
module.exports = ticker
```

> 注意：上面这种方式并不是完美的，有的方法可以绕过：`require('events').prototype.emit.call(ticker, 'someEvent', {});`。

### In the wild

除了 `Promise` 外其实很难再找到这样的库了，现在 `Streams` 议案中有一个新的规范，可以尝试使用揭示构造函数模式替代现今的模板模式，以便能够描述各种 `Streams` 对象的行为：可以看 [https://streams.spec.whatwg.org](https://streams.spec.whatwg.org)。

## Proxy(代理模式)

`Proxy` 是用来控制访问一个被称为主题（`subject`）的对象的，这种模式也可以叫做 `代替模式`，`Proxy` 拦截对 `subject` 的操作，并对行为进行增强和补充。

![Proxy](/assets/img/proxy.png)

上图说明 `Proxy` 和 `Subject` 是有相同的接口的，对客户端都是透明的，`Proxy` 将操作转发给 `Subject` 只不过通过额外的预处理增强其行为。

代理的应用场景：

- 数据验证：在 `Proxy` 向 `Subject` 转发数据前验证其数据输入的合法性。
- 安全性：代理验证客户端是否有权限，仅仅当有权限时才会向 `Subject` 发送相关请求。
- 缓存：代理对象保存内部缓存，仅仅当缓存未命中时才向 `Subject` 发送相关请求。
- 懒加载：如果 `Subject` 的创建需要消耗大量资源，代理可以推迟创建 `Subject` 的时机。
- 日志：代理拦截方法和对应的参数调用，并在他们执行前后实现日志打印。
- 远程对象：代理可以接收远程对象，并使得其呈现为本地对象。

### Techniques for implementing proxies(实现代理的方法)

#### Object Composition(对象组合)

创建具有与主体对象相同接口的新对象，并且对该主体的引用以实例变量或闭包变量的形式存储在代理内部。

```js
function createProxy(subject) {
  const proto = Object.getPrototypeOf(subject)

  function Proxy(subject) {
    this.subject = subject
  }
  Proxy.prototype = Object.create(proto)
  //proxied method
  Proxy.prototype.hello = function() {
    return this.subject.hello() + ' world!'
  }
  //delegated method
  Proxy.prototype.goodbye = function() {
    return this.subject.goodbye.apply(this.subject, arguments)
  }
  return new Proxy(subject)
}
module.exports = createProxy
```

可以发现这个 `Proxy` 对象提供了和 `Subject` 一样的接口，加强了 `hello()` 方法，直接委托了 `goodbye()` 方法。

前面的代码也显示了一个特定情况：主体对象有一个原型，我们希望维护正确的原型链，以便执行 `proxy instanceof Subject` 将返回 `true`，使用继承实现了这一点。

更多时候由于 `js` 的动态类型，我们可以简单化，使用对象字面量：

```js
function createProxy(subject) {
  return {
    //proxied method
    hello: () => subject.hello() + ' world!',
    //delegated method
    goodbye: () => subject.goodbye.apply(subject, arguments)
  }
}
```

#### Object augmentation(对象增强)

对象增强(或称为猴子补丁)是最为实用的实现代理的方式了，直接更改代理对象的方法：

```js
function createProxy(subject) {
  const helloOrig = subject.hello
  subject.hello = () => helloOrig.call(this) + ' world!'
  return subject
}
```

### A comparision of the different techniques(不同方法的比较)

对象组合方式是比较安全的，因为不修改原对象，缺点是需要委托所有的方法，尽管可能只需要代理某一个或某几个方法，甚至有时候还需要委托属性。

对象增强就与对象组合方式相反，通常来讲，如果更改对象影响不是很大的话首选对象增强方式。

### Creating a Writable stream

实现一个代理的 `Writable Stream`，增加写入日志的功能：

```js
const fs = require('fs')

function createLoggingWritable(writableOrig) {
  const proto = Object.getPrototypeOf(writableOrig)

  function LoggingWritable(writableOrig) {
    this.writableOrig = writableOrig
  }

  LoggingWritable.prototype = Object.create(proto)

  LoggingWritable.prototype.write = function(chunk, encoding, callback) {
    if (!callback && typeof encoding === 'function') {
      callback = encoding
      encoding = undefined
    }
    console.log('Writing ', chunk)
    return this.writableOrig.write(chunk, encoding, function() {
      console.log('Finished writing ', chunk)
      callback && callback()
    })
  }

  LoggingWritable.prototype.on = function() {
    return this.writableOrig.on.apply(this.writableOrig, arguments)
  }

  LoggingWritable.prototype.end = function() {
    return this.writableOrig.end.apply(this.writableOrig, arguments)
  }

  return new LoggingWritable(writableOrig)
}
```

可以看到上面的实现方式是用对象组合的方式的，这里为了简单只委托重要的几个方法，我们覆盖了 `write()` 方法，每次调用 `write()` 时都会将消息记录到标准输出，并且每次异步操作完成时都会记录消息。

### Proxy in the ecosystem - function hooks and AOP(生态中的代理——函数钩子和 AOP)

`npm` 中有几个库帮助开发人员使用函数钩子可以看下：[hooks](https://npmjs.org/package/hooks)、[hooker](https://npmjs.org/package/hooker)、[meld](https://npmjs.org/package/meld)。

### ES2015 Proxy

`ES2015` 规范引入了一个名为 `Proxy` 的全局对象，它可以从开始在 `Node.js v6.0` 中使用。

`Proxy API` 包含了一个接受 `target` 和 `handler` 的构造函数:

```js
const proxy = new Proxy(target, handler)
```

`handler` 对象包含一系列具有预定义名称的可选方法，这些方法称为陷阱方法（例如，`apply`，`get`，`set` 和 `has`），这些方法在代理实例上执行相应的操作时会自动调用。

```js
const scientist = {
  name: 'nikola',
  surname: 'tesla'
}
const uppercaseScientist = new Proxy(scientist, {
  get: (target, property) => target[property].toUpperCase()
})
console.log(uppercaseScientist.name, uppercaseScientist.surname)
// prints NIKOLA TESLA
```

这个例子使用 `Proxy API` 来拦截所有对 `scientist` 属性的访问，并将属性的原始值转换为大写字符串。

再看个例子：

```js
const evenNumbers = new Proxy([], {
  get: (target, index) => index * 2,
  has: (target, number) => number % 2 === 0
})
console.log(2 in evenNumbers) // true
console.log(5 in evenNumbers) // false
console.log(evenNumbers[7]) // 14
```

这个例子创建了一个虚拟数组，因为是不真正存储数据的，只是定义了 `has` 和 `get` 就完成了虚拟数组（包含了所有的偶数）。

更多详细的有关 `Proxy` 的用法可以查看[官方文档](https://developer.mozilla.org/it/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 或者 `Google` 的 [Introducing ES2015 Proxies](https://developers.google.com/web/updates/2016/02/es2015-proxies)。

### In the wild

`Mongoose` 是 `MongoDB` 的一个流行的对象文档映射（`ODM`）库。 在内部，它使用 `hooks` 为 `init`，`validate`，`save` 和 `remove` 函数提供预处理和后处理的钩子函数。有关官方文档，请参阅 [Mongoose](http://mongoosejs.com/docs/middleware.html) 的官方文档。

## Decorator(装饰器)

装饰器模式：动态增强已有的一个对象实例，而不是对整个类增强；与 `Proxy` 模式类似，但是不增强或者修改现有的接口，而是新增接口。

![Decorator](/assets/decorator.png)

从图中可以推断出，`Decorator` 模式可以和 `Proxy` 模式组合，装饰器模式负责新增接口，代理模式负责增强或修改现有的接口(拦截对主体的访问，并做增强)。

### Techniques for implementing Decorators(实现装饰器的方法)

#### Composition(组合)

一般使用一个新的对象包含被装饰的组件，该对象继承组件，并新增需要的方法，同时委托已有的方法给源组件。

```js
function decorate(component) {
  const proto = Object.getPrototypeOf(component)

  function Decorator(component) {
    this.component = component
  }
  Decorator.prototype = Object.create(proto)
  // new method
  Decorator.prototype.greetings = function() {
    return 'Hi!'
  }
  // delegated method
  Decorator.prototype.hello = function() {
    return this.component.hello.apply(this.component, arguments)
  }
  return new Decorator(component)
}
```

#### Object augmentation(对象增强)

对象装饰器也可以直接在源对象上添加新的方法：

```js
function decorate(component) {
  // new method
  component.greetings = () => {
    return component
  }
}
```

### Decorating a LevelUP database(装饰一个 LevelUP 数据库)

#### Introducing LevelUP and LevelDB(介绍 LevelUP 和 LevelDB)

[LevelUP](https://www.npmjs.com/package/levelup) 是 `Google` 的 `LevelDB` 上的一个 `Node.js` 包装器，它是最初为了在 `Chrome` 浏览器中实现 `IndexedDB` 而创建的 `key/value` 存储库。详细的 `LevelUP` 生态可查看 [https://github.com/rvagg/node-levelup/wiki/Modules](https://github.com/rvagg/node-levelup/wiki/Modules)。

#### Implementing a LevelUP plugin

我们想要构建的是一个 `LevelUP` 的插件，当将具有特定模式的对象保存到数据库时让我们接到通知。例如如果我们订阅 `{a:1}` 模式，那么当类似 `{a:1,b:3}` 或 `{a:1,c:'x'}` 的数据存储到数据库时，我们会收到一个通知。

```js
module.exports = function levelSubscribe(db) {
  db.subscribe = (pattern, listener) => {
    //[1]
    db.on('put', (key, val) => {
      //[2]
      const match = Object.keys(pattern).every(
        k => pattern[k] === val[k] //[3]
      )

      if (match) {
        listener(key, val) //[4]
      }
    })
  }
  return db
}
```

使用对象增强的方法直接将新方法附加到 `db` 实例上。

## Adapter(适配器模式)

适配器模式其实是包装接口来供不同的调用，例如升级接口后用适配器包装新接口给老代码调用。

![Adapter](/assets/img/adapter.png)

### Using LevelUP througn the filesystem API(通过 fs 的 API 来使用 LevelUP)

```js
const path = require('path')
module.exports = function createFsAdapter(db) {
  const fs = {}
  fs.readFile = function(filename, options, callback) {
    if (typeof options === 'function') {
      callback = options
      options = {}
    } else if (typeof options === 'string') {
      options = {
        encoding: options
      }
    }
    db.get(
      path.resolve(filename),
      {
        //[1]
        valueEncoding: options.encoding
      },
      function(err, value) {
        if (err) {
          if (err.type === 'NotFoundError') {
            //[2]
            err = new Error("ENOENT, open '" + filename + "'")
            err.code = 'ENOENT'
            err.errno = 34
            err.path = filename
          }
          return callback && callback(err)
        }
        callback && callback(null, value) //[3]
      }
    )
  }
}

fs.writeFile = (filename, contents, options, callback) => {
  if (typeof options === 'function') {
    callback = options
    options = {}
  } else if (typeof options === 'string') {
    options = { encoding: options }
  }

  db.put(
    path.resolve(filename),
    contents,
    {
      valueEncoding: options.encoding
    },
    callback
  )
}

return fs
```

### In the wild

`LevelUP` 能在浏览器中能以不同的存储后端运行，从 `LevelDB` 到 `IndexedDB`。这是通过那些适应了内部 `LevelUP API` 接口的适配器(`Adapter`)来实现的。具体有哪些实现方式查看 [https://github.com/rvagg/node-levelup/wiki/Modules#storage-back-ends](https://github.com/rvagg/node-levelup/wiki/Modules#storage-back-ends)。

## Strategy(策略模式)

![Strategy](/assets/img/strategy.png)

从图中看出该模式其实是根据配置(或用户输入之类)来做不同逻辑的事，都实现了相同的接口。比大量的 `if...else` 或 `swtich` 更易懂。

```js
const fs = require('fs')
const objectPath = require('object-path')

class Config {
  constructor(strategy) {
    this.data = {}
    this.strategy = strategy
  }

  get(path) {
    return objectPath.get(this.data, path)
  }

  set(path, value) {
    return objectPath.set(this.data, path, value)
  }

  read(file) {
    console.log(`Deserializing from ${file}`)
    this.data = this.strategy.deserialize(fs.readFileSync(file, 'utf-8'))
  }

  save(file) {
    console.log(`Serializing to ${file}`)
    fs.writeFileSync(file, this.strategy.serialize(this.data))
  }
}

module.exports = Config
```

上面代码可以传入不同的配置来做不同的序列化与反序列化：

```js
// strategy 1
module.exports.json = {
  deserialize: data => JSON.parse(data),
  serialize: data => JSON.stringify(data, null, '  ')
}
```

```js
// strategy 2
const ini = require('ini') // https://npmjs.org/package/ini
module.exports.ini = {
  deserialize: data => ini.parse(data),
  serialize: data => ini.stringify(data)
}
```

## State(状态模式)

状态模式是策略模式的变种，策略模式一旦确定策略在整个过程中策略(也就是处理逻辑不变)，而状态模式可以动态地改变状态来间接地影响策略：

![State](/assets/img/state.png)

想象一下，我们有一个酒店预订系统和一个 `Reservation` 对象来模拟房间预订。

这是一个经典的情况，我们必须根据其状态来调整对象的行为。考虑以下一系列事件：

- 当订单初始创建时，用户可以使用 `confirm()` 方法确认订单；当然，他们不能使用 `cancel()` 方法取消预约，因为订单还没有被确认。但是，如果他们在购买之前改变主意，他们可以使用 `delete()` 方法删除它。
- 一旦确认订单，再次使用 `confirm()` 方法没有任何意义；不过，现在应该可以取消预约，但不能再删除，因为要保留对应记录。
- 在预约日期前一天，不应取消订单。因为这太迟了。

### Implementing a basic fail-safe socket(实现一个基本的 fail-safe socket)

尝试一个例子，建立一个 `socket`，当与服务器断开连接时保存客户端的请求，并将这些请求按顺序排队，等到下次重新连接时按照顺序一一请求。

```js
// file offlineState.js
const jot = require('json-over-tcp') // [1]

module.exports = class OfflineState {
  constructor(failsafeSocket) {
    this.failsafeSocket = failsafeSocket
  }

  send(data) {
    // [2]
    this.failsafeSocket.queue.push(data)
  }

  activate() {
    // [3]
    const retry = () => {
      setTimeout(() => this.activate(), 500)
    }

    this.failsafeSocket.socket = jot.connect(
      this.failsafeSocket.options,
      () => {
        this.failsafeSocket.socket.removeListener('error', retry)
        this.failsafeSocket.changeState('online')
      }
    )
    this.failsafeSocket.socket.once('error', retry)
  }
}
```

```js
// file onlineState.js
module.exports = class OnlineState {
  constructor(failsafeSocket) {
    this.failsafeSocket = failsafeSocket
  }

  send(data) {
    // [1]
    this.failsafeSocket.socket.write(data)
  }

  activate() {
    // [2]
    this.failsafeSocket.queue.forEach(data => {
      this.failsafeSocket.socket.write(data)
    })
    this.failsafeSocket.queue = []

    this.failsafeSocket.socket.once('error', () => {
      this.failsafeSocket.changeState('offline')
    })
  }
}
```

```js
// file failsafeSocket.js
const OfflineState = require('./offlineState')
const OnlineState = require('./onlineState')

class FailsafeSocket {
  constructor(options) {
    // [1]
    this.options = options
    this.queue = []
    this.currentState = null
    this.socket = null
    this.states = {
      offline: new OfflineState(this),
      online: new OnlineState(this)
    }
    this.changeState('offline')
  }

  changeState(state) {
    // [2]
    console.log('Activating state: ' + state)
    this.currentState = this.states[state]
    this.currentState.activate()
  }

  send(data) {
    // [3]
    this.currentState.send(data)
  }
}

module.exports = options => {
  return new FailsafeSocket(options)
}
```

`FailsafeSocket` 从一个状态切换到另一个状态，只是切换了实例，具体的发送方法根据状态来选择，离线则使用 `OfflineState` 来发送，连接则使用 `OnlineState` 来发送。

## Template(模板模式)

和策略模式差不多，只是需要预先定义变体，使用继承改变原有的方法，注意在 `js` 中模板类是总是抛出异常的类或未定义的方法(因为一定要有实现类才能使用)。

![Template](/assets/img/template.png)

### In the wild

其实在第五章中流的实现就是用了这种模式，自定义的流需要实现 `_read` 和 `_write` 这类的方法。

## Middleware(中间件模式)

### Middleware in Express(Express 中的中间件)

在 `Express` 中，中间件表示一组服务，通常是函数，它们被组织在一个 `pipeline` 中，负责处理传入的 `HTTP` 请求和进行响应。

一个 `Express` 的中间件有下面这种形式：

```js
function(req, res, next) { ... }
```

在这里，`req` 是传入的 `HTTP` 请求，`res` 是响应，`next` 是当前中间件完成其任务时调用的回调，用来触发 `pipeline` 中的下一个中间件。可能的 `Express` 中间件任务有:

- 解析请求的 `body`
- 压缩/解压 `req` 和 `res` 对象
- 生成访问日志
- 管理 `sessions`
- 管理加密的 `cookie`
- 提供跨站请求伪造（`CSRF`）保护

这些都是与应用程序的主要业务逻辑没有严格关联的任务，也不是 Web 服务器最核心的部分；它们是应用程序公共功能的中间件，使得实际的请求处理程序只关注其主要业务逻辑。

### Middleware as a pattern(中间件作为一种模式)

其实类似于 `Pipe-Filter` 模式，通过看一张图更能明白：

![Middleware](/assets/img/middleware.png)

最重要的就是这个 `Middleware Manager`，负责组织和执行中间件功能。

1.  新的中间件通过 `use()`(一般约定，当然也可以用别的名称) 来注册，一般是管道末尾。
2.  注册的中间件在异步顺序执行流中被调用，后一个的输入是前一个中间件的输出。
3.  中间件只负责处理正常流程，错误通常会触发另一个专门的中间件序列。

### Creating a middleware for ØMQ(为 ØMQ 创建一个中间件框架)

`ØMQ`（也称为 `ZMQ` 或 `ZeroMQ`）提供了一个简单的接口，用于通过各种协议在网络中交换原子消息；它的性能绝佳，其基本的抽象集是专门构建的，以促进自定义消息体系结构的实现。因此，经常选择 `ØMQ` 来构建复杂的分布式系统。

我们将构建一个中间件基础结构，以抽象通过 `ØMQ` 套接字传递的数据的预处理和后处理，以便我们可以透明地处理 `JSON` 对象，同时无缝地压缩通过线路传递的消息。

#### The Middleware Manager(中间件管理器)

```js
module.exports = class ZmqMiddlewareManager {
  constructor(socket) {
    this.socket = socket
    this.inboundMiddleware = [] // [1]
    this.outboundMiddleware = []
    socket.on('message', message => {
      // [2]
      this.executeMiddleware(this.inboundMiddleware, {
        data: message
      })
    })
  }

  send(data) {
    const message = {
      data: data
    }

    this.executeMiddleware(this.outboundMiddleware, message, () => {
      this.socket.send(message.data)
    })
  }

  use(middleware) {
    if (middleware.inbound) {
      this.inboundMiddleware.push(middleware.inbound)
    }
    if (middleware.outbound) {
      this.outboundMiddleware.unshift(middleware.outbound)
    }
  }

  executeMiddleware(middleware, arg, finish) {
    function iterator(index) {
      if (index === middleware.length) {
        return finish && finish()
      }
      middleware[index].call(this, arg, err => {
        if (err) {
          // 这里本应该有对应的错误中间件去处理，为了简洁直接输出
          return console.log('There was an error: ' + err.message)
        }
        // 一个中间件处理完参数后，传递给下一个中间件
        iterator.call(this, ++index)
      })
    }

    iterator.call(this, 0)
  }
}
```

管理器接收 `ØMQ` 套接字作为参数，定义一个近站中间件列表和一个出站中间件列表，当有消息来时依次调用进站中间件(按照 `use` 的顺序来)，需要发送消息时就依次调用出站中间件，被处理后的参数也是一一传播。

#### A middleware to support JSON messages(一个支持 JSON 消息的中间件)

```js
// file jsonMiddleware.js
module.exports.json = () => {
  return {
    inbound: function(message, next) {
      message.data = JSON.parse(message.data.toString())
      next()
    },
    outbound: function(message, next) {
      message.data = new Buffer(JSON.stringify(message.data))
      next()
    }
  }
}
```

想要使用的时候只需要 `use(jsonMiddleware)` 就行了，很方便。

#### Using the ØMQ middleware framework(使用 ØMQ 中间件框架)

##### The server

```js
const zmq = require('zmq')
const ZmqMiddlewareManager = require('./zmqMiddlewareManager')
const jsonMiddleware = require('./jsonMiddleware')
const reply = zmq.socket('rep')
reply.bind('tcp://127.0.0.1:5000')

const zmqm = new ZmqMiddlewareManager(reply)
zmqm.use(jsonMiddleware.json())
```

##### The client

```js
const zmq = require('zmq')
const ZmqMiddlewareManager = require('./zmqMiddlewareManager')
const jsonMiddleware = require('./jsonMiddleware')
const request = zmq.socket('req')
request.connect('tcp://127.0.0.1:5000')

const zmqm = new ZmqMiddlewareManager(request)
zmqm.use(jsonMiddleware.json())

//处理服务器响应的中间件
zmqm.use({
  inbound: function(message, next) {
    console.log('Echoed back: ', message.data)
    next()
  }
})

setInterval(() => {
  zmqm.send({
    action: 'ping',
    echo: Date.now()
  })
}, 1000)
```

### Middleware using generators in Koa(在 Koa 中使用生成器中间件)

`Koa` 不像 `Express` 一样使用回调函数来完成中间件模式，而是使用生成器(`generator`)，使用中间件包装核心应用程序，这种形式更像是洋葱一样：

![Koa Middleware](/assets/img/koa_middleware.png)

我们来看一个官方的例子(`ES7`)：

```js
const Koa = require('koa')
const app = new Koa()

// x-response-time

app.use(async (ctx, next) => {
  const start = Date.now()
  await next()
  const ms = Date.now() - start
  ctx.set('X-Response-Time', `${ms}ms`)
})

// logger

app.use(async (ctx, next) => {
  const start = Date.now()
  await next()
  const ms = Date.now() - start
  console.log(`${ctx.method} ${ctx.url} - ${ms}`)
})

// response

app.use(async ctx => {
  ctx.body = 'Hello World'
})

app.listen(3000)
```

可以发现 `response` 部分才是核心应用程序部分，只不过被其他的中间件包裹起来了，通过 `await` 分割。

> 注意，现在 `Koa` 已经开始使用 `ES7` 的语法 `async`/`await` 了，详情查看[官方文档](https://koajs.com)。

## Command(命令模式)

可以认为一个命令(`Command`)是一个封装了重要的信息以便之后去执行一个特定的动作的对象。我们不直接在主体对象上调用一个方法或一个函数，而是创建一个对象来执行这样一次调用；而实现这个意图将是另一个组件的责任，该组件将意图转化为一系列操作。

![Command](/assets/img/command.png)

命令模式典型的架构：

- `Command`:这是一个封装了足够的信息去调用方法或函数的对象，就像是定义了一个接口。
- `Client`:创建命令对象并提供给调用者(`Invoker`)。
- `Invoker`:负责执行目标(`Target`)上的命令，负责调用 `Command`。
- `Target`(或 `Receiver`):调用的主体，它可以是一个对象上的单独的方法或函数。

命令模式有点：

- 命令可以稍后执行。
- 命令可以被序列化并在网络上传输。这使得我们可以远程分配任务，通过浏览器传输命令给服务器，创建 `RPC` 系统等等。
- 很容易记录操作历史。
- 命令是数据同步和冲突解决某些算法的重要部分。
- 定时执行的命令可以取消；命令也可以撤销(`undone`)。
- 命令可以组合起来，用来创建原子事务或实现同时执行一些操作的机制。
- 一组命令可以有不同的变化，例如可以删除、插入、分割等等。

### A flexible pattern(一个灵活的模式)

正如上面所说，命令模式可以有很多种实现方式，我们来看看其中几个。

#### A task pattern(任务模式)

最简单的方式就是创建一个闭包：

```js
function createTask(target, args) {
  return () => {
    target.apply(null, args)
  }
}
```

这种技术允许我们使用单独的组件来控制和调度任务的执行，这在本质上等同于命令模式的调用者(`Invoker`，其实是同时创建了命令(`Command`))。

#### A more complex command(一个更复杂的命令)

我们希望撤销和序列化。命令的目标(`Target`) 是一个负责发送状态更新的对象:

```js
const statusUpdateService = {
  statusUpdates: {},
  sendUpdate: function(status) {
    console.log('Status sent: ' + status)
    let id = Math.floor(Math.random() * 1000000)
    statusUpdateService.statusUpdates[id] = status
    return id
  },
  destroyUpdate: id => {
    console.log('Status removed: ' + id)
    delete statusUpdateService.statusUpdates[id]
  }
}
```

接着创建一个命令来新状态的发布：

```js
function createSendStatusCmd(service, status) {
  let postId = null
  const command = () => {
    postId = service.sendUpdate(status)
  }
  command.undo = () => {
    if (postId) {
      service.destroyUpdate(postId)
      postId = null
    }
  }
  command.serialize = () => {
    return {
      type: 'status',
      action: 'post',
      status: status
    }
  }
  return command
}
```

`command` 本身是一个函数，使用目标的方法发送状态更新，附在上面的 `undo` 函数直接调用目标的 `destroyUpdate` 函数来完成命令撤销，`serialize` 函数构建了一个 `JSON` 对象记录执行命令所需要的重要信息。

然后再来创建执行者 `Invoker`:

```js
class Invoker {
  constructor() {
    this.history = []
  }
  run(cmd) {
    this.history.push(cmd)
    cmd()
    console.log('Command executed', cmd.serialize())
  }
}
```

执行者还可以做一些额外的操作，如记录命令的执行，远程调用，延迟执行命令，例如：

```js
class Invoker {
  constructor() {
    this.history = []
  }
  run(cmd) {
    this.history.push(cmd)
    cmd()
    console.log('Command executed', cmd.serialize())
  }
  delay(cmd, delay) {
    setTimeout(() => {
      this.run(cmd)
    }, delay)
  }
  runRemotely(cmd) {
    request.post(
      'http://localhost:3000/cmd',
      {
        json: cmd.serialize()
      },
      err => {
        console.log('Command executed remotely', cmd.serialize())
      }
    )
  }
}
```

最后编写客户端(`Client`):

```js
const invoker = new Invoker()
const command = createSendStatusCmd(statusUpdateService, 'HI!')
invoker.run(command)
invoker.runRemotely(command)
```

> 命令模式最好在需要一些复杂的代码来调用目标上的函数或方法时使用，不然只是简单地调用一个方法就显得非常多余了。
