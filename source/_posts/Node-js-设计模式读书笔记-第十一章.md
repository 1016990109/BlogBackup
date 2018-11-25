---
title: 《Node.js 设计模式》读书笔记 第十一章
date: 2018-10-29 10:00:51
tags:
  - 读书笔记
---

# Messaging and Integration Patterns(消息和模式整合)

## Fundamentals of a messaging system(一个消息系统的基础)

一般来说消息系统有以下 4 个基础：

- 消息传递的方向，是单向的还是“请求/响应”的方向。
- 消息的目的，这也决定了消息的内容
- 消息的时间性，是同步还是异步
- 消息的分发，是直接发送还是通过代理

<!-- more -->

### Message types(消息的类型)

#### Command Message(命令消息)

这类消息的目的是在接收方执行一个动作或任务，因此消息通常包含了执行任务所需要的参数(操作的名称+运行的参数)。命令消息可以用来实现 `RPC`、分布式计算、请求数据等。`RESTful HTTP` 请求就是命令的例子，`HTTP` 的动作都有特定的含义：`GET` 表示获取资源，`POST` 表示创建新资源，`PUT` 表示更新，`DELETE` 表示删除。

#### Event Message(事件消息)

事件消息是用来通知其他组件有事情发横了，通常包含了事件的类型，甚至有时候包含了一些细节如上下文、创建者。

#### Document Message(文档消息)

主要用于组件和机器之间传递数据。和命令消息的区别就是不含如何处理命令的任何内容，和事件消息的区别就是没有特定的事件的关联。通常对命令消息的回复是文档消息。

### Asynchronous messaging and queues(异步消息队列)

异步消息类似于 `SMS`，发邮件时不要求对方已连接互联网，可能立即或者一段时间后收到回复，甚至是没有回复，这样我们就可以不用等待回复继续发送下一封邮件。总之，就是以更少的资源获得更好的并发性。

另一个优点是可以将消息存储并尽快或延迟发送，当接收方太忙或者我们想要保证传送时非常有用。这可以使用消息队列来实现：

![message queue](/assets/img/message_queue.png)

### Peer-to-peer or broker-based messaging(点对点通信或基于代理的消息)

消息可以点对点直接发送，也可以使用中心化的消息代理系统转发。

![message architecture](/assets/img/message_architecture.png)

在点对点体系中要求每个节点知道接收方得地址和端口，并且得使用相同的协议和消息格式，限制太多；而消息代理则不需要，每个节点完全独立，还可以有多种协议。

## Publish/subscribe pattern(发布/订阅 模式)

”发布/订阅“ 模式也有点对点架构和代理架构：

![Pub/Sub architecture](/assets/img/pub_sub_architecture.png)

### Building a minimalist real-time chat application(构建一个微型实时聊天应用)

为了展示 “发布/订阅” 模式如何帮助我们构建一个分布式架构，我们通过构建一个简单的基于 `WebSockets` 实时聊天应用来说明。

#### Implementing the server side

```js
const WebSocketServer = require('ws').Server

//static file server
const server = require('http').createServer(
  //[1]
  require('ecstatic')({ root: `${__dirname}/www` })
)

const wss = new WebSocketServer({ server: server }) //[2]
wss.on('connection', ws => {
  console.log('Client connected')
  ws.on('message', msg => {
    //[3]
    console.log(`Message: ${msg}`)
    broadcast(msg)
  })
})

function broadcast(msg) {
  //[4]
  wss.clients.forEach(client => {
    client.send(msg)
  })
}

server.listen(process.argv[2] || 8080)
```

#### Implementing the client side

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      var ws = new WebSocket('ws://' + window.document.location.host)
      ws.onmessage = function(message) {
        var msgDiv = document.createElement('div')
        msgDiv.innerHTML = message.data
        document.getElementById('messages').appendChild(msgDiv)
      }

      function sendMessage() {
        var message = document.getElementById('msgBox').value
        ws.send(message)
      }
    </script>
  </head>
  <body>
    Messages:
    <div id="messages"></div>
    <input type="text" placeholder="Send a message" id="msgBox" />
    <input type="button" onclick="sendMessage()" value="Send" />
  </body>
</html>
```

#### Running and scaling the chat application(运行并扩展聊天应用)

```bash
node app 8080
node app 8081
```

启动两个服务，当在一个客户端发送消息时，只是将消息广播给本地连接的客户端了，另一个连着另外一个服务器的客户端并收不到消息，这和我们的意愿是相悖的。

### Using Redis as a message broker(使用 Redis 作为消息代理)

了解 `Redis` 请查看 [http://redis.io/topics/quickstart](http://redis.io/topics/quickstart)。

![redis broker](/assets/img/redis_broker.png)

修改服务器代码，并引入 `redis` 模块，代码可查看 []：

```js
const WebSocketServer = require('ws').Server
const redis = require('redis')
const redisSub = redis.createClient()
const redisPub = redis.createClient()

//static file server
const server = require('http').createServer(
  require('ecstatic')({ root: `${__dirname}/www` })
)

const wss = new WebSocketServer({ server: server })
wss.on('connection', ws => {
  console.log('Client connected')
  ws.on('message', msg => {
    console.log(`Message: ${msg}`)
    redisPub.publish('chat_messages', msg)
  })
})

redisSub.subscribe('chat_messages')
redisSub.on('message', (channel, msg) => {
  wss.clients.forEach(client => {
    client.send(msg)
  })
})

server.listen(process.argv[2] || 8080)
```

然后再启动多个服务就发现可以正常通信了。

> 详细代码可以查看 [https://github.com/1016990109/front_end_practice/tree/master/src/node/design-pattern/basic-chat](https://github.com/1016990109/front_end_practice/tree/master/src/node/design-pattern/basic-chat)

### Peer-to-peer publish/subscribe with 􏱔􏰊􏰖􏱔􏰊􏰖􏱔􏰊􏰖ØMQ(使用 ØMQ 实现点对点发布订阅)

#### Introducing ØMQ

在 `ØMQ` 中，我们有两种专门为此设计的套接字:`PUB` 和 `SUB`。典型的模式是将 `PUB` 套接字绑定到一个端口，该端口将开始侦听来自其他 `SUB` 套接字的订阅。订阅可以有一个过滤器，指定将传递到 `SUB` 套接字的消息。该过滤器是一个简单的二进制缓冲区(所以它也可以是一个字符串)，它将与消息的开头(这也是一个二进制缓冲区)相匹配。当通过 `PUB` 套接字发送一条消息时，它将被广播到所有连接的 `SUB` 套接字，但仅在应用了它们的订阅过滤器之后。仅当使用连接的协议时，过滤器才会应用到发布方，例如 `TCP`。

#### Designing a peer-to-peer architecture for the chat server(为聊天服务设计一个点对点架构)

![ØMQ pub sub](/assets/img/ØMQ_pub_sub.png)

#### Using the ØMQ PUB/SUB sockets(使用 ØMQ 发布订阅套接字)

修改服务端代码：

```js
const WebSocketServer = require('ws').Server
const args = require('minimist')(process.argv.slice(2))
const zmq = require('zmq')

//static file server
const server = require('http').createServer(
  require('ecstatic')({ root: `${__dirname}/www` })
)

const pubSocket = zmq.socket('pub')
pubSocket.bind(`tcp://127.0.0.1:${args['pub']}`)

const subSocket = zmq.socket('sub')
const subPorts = [].concat(args['sub'])
subPorts.forEach(p => {
  console.log(`Subscribing to ${p}`)
  subSocket.connect(`tcp://127.0.0.1:${p}`)
})
subSocket.subscribe('chat')

subSocket.on('message', msg => {
  console.log(`From other server: ${msg}`)
  broadcast(msg.toString().split(' ')[1])
})

const wss = new WebSocketServer({ server: server })
wss.on('connection', ws => {
  console.log('Client connected')
  ws.on('message', msg => {
    console.log(`Message: ${msg}`)
    broadcast(msg)
    pubSocket.send(`chat ${msg}`)
  })
})

function broadcast(msg) {
  wss.clients.forEach(client => {
    client.send(msg)
  })
}

server.listen(args['http'] || 8080)
```

1. 创建 `pub` 套接字，绑定到对应参数的端口。
2. 创建 `sub` 套接字，连接到应用其他实例的 `pub` 套接字。
3. 然后通过 `chat` 作为过滤器创建实际的发布订阅。
4. 当我们的 `WebSocket` 收到新小时时，广播给所有连接的客户端，也通过 `pub` 套接字发布它。注意内容是 `chat ${msg}` 前面加了 `chat`，所以发布到所有使用 `chat` 订阅者。
5. `sub` 套接字收到消息则做处理，取得需要的消息，并广播到所有连接当前 `WebSocket` 的客户端。

尝试运行：

```
node app --http 8080 --pub 5000 --sub 5001 --sub 5002
node app --http 8081 --pub 5001 --sub 5000 --sub 5002
node app --http 8082 --pub 5002 --sub 5000 --sub 5001
```

> 详细代码可以查看 [https://github.com/1016990109/front_end_practice/tree/master/src/node/design-pattern/chat-zmq](https://github.com/1016990109/front_end_practice/tree/master/src/node/design-pattern/chat-zmq)

### Durable subscribers(持久订阅者)

消息传递系统中的一个重要抽象是消息队列(`MQ`)。对于消息队列，消息的发送者和接收者不需要同时处于活动状态和连接状态以建立通信，因为排队系统负责存储消息直到目的地能够接收他们。这种行为与 `set and forget` 范式相反，订户只能在消息系统连接期间才能接收消息。

一个能够始终可靠地接收所有消息的订阅者，即使是在没有收听这些消息时发送的消息，也被称为**持久订阅者**。

![durable subscribers](/assets/img/durable_subscribers.png)

`MQTT` 协议为发送方和接收方之间交换的消息定义了服务质量(`QoS`)级别。这些级别对描述任何其他消息系统(不仅仅是 MQTT )的可靠性也非常有用。如下描述:

1. `QoS0` ,最多一次:也被称为“设置并忘记”,消息不会被保留,并且传送未被确认。这意味着在接收机崩溃或断开的情况下,信息可能会丢失。
2. `QoS1` ,至少一次:保证至少收到一次该消息,但如果在通知发件人之前接收器崩溃,则可能发生重复。这意味着消息必须在必须再次发送的情况下持续下去。
3. `QoS2` ,正好一次:这是最可靠的 `QoS` ; 它保证该消息只被接收一次。 这是以用于确认消息传递的更慢和更数据密集型机制为代价的。

`Redis` 的发布/订阅命令实现了一个设置和遗忘机制( `QoS0` )。但是,`Redis` 仍然可以用于使用其他命令的组合来实现持久订阅者(不直接依赖其发布/订阅实现)。

#### Instroducing AMQP(AMQP 介绍)

`AMQP` 是许多消息队列系统支持的开放标准协议。除了定义通用通信协议外,它还提供了描述路由,过滤,排队,可靠性和安全性的模型。在 `AMQP` 中,有三个基本组成部分:

- `Queue`(队列):负责存储客户端消费的消息的数据结构。我们的应用程序推送消息到队列,供给一个或多个消费者。如果多个使用者连接到同一个队列,则这些消息会在它们之间进行负载平衡。 队列可以是以下之一:
  - `Durable`(持久队列) :这意味着如果代理重新启动,队列会自动重新创建。一个持久的队列并不意味着它的内容也被保留下来;实际上,只有标记为持久性的消息才会保存到磁盘,并在重新启动的情况下进行恢复。
  - `Exclusive`(专有队列) :这意味着队列只能绑定到一个特定的用户连接。当连接关闭时,队列被销毁。
  - `Auto-delete`(自动删除队列) :这会导致队列在最后一个用户断开连接时被删除。
- `Exchange`(交换机) :这是发布消息的地方。交换机根据它实现的算法将消息路由到一个或多个队列:
  - `Direct exchange`(直接交换机) :通过匹配路由键(例如, `chat.msg` )整个消息来路由消息。
  - `Topic exchange`(主题交换机) :它使用与路由密钥相匹配的类似 `glob` 的模式分发消息(例如, `chat.#` 匹配以 `chat` 开始的所有路由密钥)。
  - `Fanout exchange`(扇出交换机) :它向所有连接的队列广播消息,忽略提供的任何路由密钥。
- `Binding`(绑定) :这是交换机和队列之间的链接。它还定义了路由键或用于过滤从交换机到达的消息的模式。

> 可以在 `RabbitMQ` 网站上找到 `AMQP` 模型的详细介绍: [https://www.rabbitmq.com/tutorials/amqp-concepts.html](https://www.rabbitmq.com/tutorials/amqp-concepts.html)

下图展示了组件如何组合：

![AMQP](/assets/img/AMQP.png)

#### Durable subscribers with AMQP and RabbitMQ(使用 AMQP 和 RabbitMQ 实现持久订阅者)

现在让我们使用微服务方法扩展我们的小聊天应用程序。让我们添加一个历史记录服务,将我们的聊天消息保存在数据库中,这样当客户端连接时,我们可以查询服务并检索整个聊天记录。我们将使用 `RabbitMQ broker`
和 `AMQP` 将历史记录服务器与聊天服务器相集成。

下图显示了我们的架构:

![RabbitMQ broker](/assets/img/RabbitMQ_broker.png)

我们将使用熟悉的 `LevelUP` 作为历史记录服务的存储引擎,而我们将使用 `amqplib`,并通过 `AMQP` 协议连接到 `RabbitMQ`。现在让我们实施我们的历史记录服务器!我们将创建一个独立的应用程序(典型的微服务),它在模块 `historySvc.js` 中实现。该模块由两部分组成:向客户端展示聊天记录的 `HTTP` 服务器,以及负责捕获聊天消息并将其存储在本地数据库中的 `AMQP` 使用者。

```js
const level = require('level')
const timestamp = require('monotonic-timestamp')
const JSONStream = require('JSONStream')
const amqp = require('amqplib')
const db = level('./msgHistory')
require('http')
  .createServer((req, res) => {
    res.writeHead(200)
    db.createValueStream()
      .pipe(JSONStream.stringify())
      .pipe(res)
  })
  .listen(8090)
let channel, queue
amqp
  .connect('amqp://localhost') // [1]
  .then(conn => conn.createChannel())
  .then(ch => {
    channel = ch
    return channel.assertExchange('chat', 'fanout') // [2]
  })
  .then(() => channel.assertQueue('chat_history')) // [3]
  .then(q => {
    queue = q.queue
    return channel.bindQueue(queue, 'chat') // [4]
  })
  .then(() => {
    return channel.consume(queue, msg => {
      // [5]
      const content = msg.content.toString()
      console.log(`Saving message: ${content}`)
      db.put(timestamp(), content, err => {
        if (!err) channel.ack(msg)
      })
    })
  })
  .catch(err => console.log(err))
```

1. 我们首先与 `AMQP` 代理建立连接,在我们的例子中是 `RabbitMQ`。然后,我们创建一个 `channel`,该 `channel` 类似于保持我们通信状态的会话。
2. 接下来,我们建立了我们的会话,名为 `chat`。正如我们已经提到的那样,这是一种扇出交换机。`assertExchange()` 命令将确保代理中存在交换,否则它将创建它。
3. 我们还创建了我们的队列,名为 `chat_history`。默认情况下,队列是持久的;不是排他性的,也不会自动删除,所以我们不需要传递任何额外的选项来支持持久订阅者。
4. 接下来,我们将队列绑定到我们以前创建的交换机。在这里,我们不需要任何其他特殊选项,例如路由键或模式,因为交换机是扇出类型的交换机,所以它不执行任何过滤。
5. 最后,我们可以开始监听来自我们刚创建的队列的消息。我们将使用时间戳记作为密钥([https://npmjs.org/package/monotonic-timestamp](https://npmjs.org/package/monotonic-timestamp))在 `LevelDB` 数据库中收到的每条消息保存,以保持消息按日期排序。看到我们使用 `channel.ack(msg)` 来确认每条消息,并且只有在消息成功保存到数据库后,也很有趣。如果代理没有收到 `ACK` (确认),则该消息将保留在队列中以供再次处理。这是 `AMQP` 将服务可靠性提升到全新水平的另一个重要特征。如果我们不想发送明确的确认,我们可以将选项 `{noAck:true}` 传递给 `channel.consume()` `API` 。

然后更改我们的 `app.js`

```js
const WebSocketServer = require('ws').Server
const amqp = require('amqplib')
const JSONStream = require('JSONStream')
const request = require('request')
let httpPort = process.argv[2] || 8080

//static file server
const server = require('http').createServer(
  require('ecstatic')({ root: `${__dirname}/www` })
)

let channel, queue
amqp
  .connect('amqp://localhost')
  .then(conn => conn.createChannel())
  .then(ch => {
    channel = ch
    return channel.assertExchange('chat', 'fanout')
  })
  .then(() => {
    return channel.assertQueue(`chat_srv_${httpPort}`, { exclusive: true })
  })
  .then(q => {
    queue = q.queue
    return channel.bindQueue(queue, 'chat')
  })
  .then(() => {
    return channel.consume(
      queue,
      msg => {
        msg = msg.content.toString()
        console.log('From queue: ' + msg)
        broadcast(msg)
      },
      { noAck: true }
    )
  })
  .catch(err => console.log(err))

const wss = new WebSocketServer({ server: server })
wss.on('connection', ws => {
  console.log('Client connected')
  //query the history service
  request('http://localhost:8090')
    .on('error', err => console.log(err))
    .pipe(JSONStream.parse('*'))
    .on('data', msg => ws.send(msg))

  ws.on('message', msg => {
    console.log(`Message: ${msg}`)
    channel.publish('chat', '', new Buffer(msg))
  })
})

function broadcast(msg) {
  wss.clients.forEach(client => client.send(msg))
}

server.listen(httpPort)
```

正如我们所提到的,我们的聊天服务器不需要成为持久的订阅者。所以当我们创建我们的队列时,我们传递选项 `{exclusive:true}`,指示队列被限制到当前连接,因此一旦聊天服务器关闭,它就会被销毁。

启动应用：

```bash
node app 8080
node app 8081
node historySvc
```

现在看看我们的系统,特别是历史服务如何在停机的情况下运行,这一点很有意思。如果我们停止历史记录服务器并继续使用聊天应用程序的 `Web UI` 发送消息,我们将会看到,当历史记录服务器重新启动时,它将立即收到所有错过的消息。

## Pipelines and task distribution patterns(管道和任务分配模式)

有时候我们的需求并不是把消息传递给每一个消费者，例如 `Map/Reduce` 之类的架构，需要将 `Map` 的任务分配给不同的 `worker`，再将最终结果组合。那么继续使用“发布/订阅”模式就不行了，这里我们介绍管道和任务分配模式：

![Pipelines and task distribution patterns](/assets/img/pipelines_task_and_distribution_patterns.png)

### The ØMQ fanout/fanin pattern (ØMQ 扇出/扇出模式)

我们已经发现了 `ØMQ` 在构建点对点分布式体系结构方面的一些优势。在前一节中,我们使用 `PUB` 和 `SUB` 套接字向多个消费者传播单个消息;现在我们将看到如何使用称为 `PUSH` 和 `PULL` 的另一对套接字来构建并行管道。

#### PUSH/PULL socket

直观地说,我们可以说 `PUSH` 套接字用于发送消息,而 `PULL` 套接字是用于接收的。这似乎是一个微不足道的组合;然而,它们有一些很好的特性,使它们成为构建单向通信系统的完美选择:

- 两者都可以在 `connet` 模式或 `bind` 模式下工作。换句话说,我们可以构建一个 `PUSH` 套接字并将其绑定到本地端口,以监听来自 `PULL` 套接字的传入连接,反之亦然, `PULL` 套接字可以监听来自 `PUSH` 套接字的连接。消息总是以相同的方向传播,从 `PUSH` 到 `PULL`;它只是连接的发起者可能是不同的。绑定模式是耐用节点(例如任务生产者和接收器)的最佳解决方案,而连接模式对于瞬态节点(例如任务工作者)来说是完美的。这使得瞬时节点的数量可以任意变化,而不会影响其它正在使用的节点。
- 如果有多个 `PULL` 套接字连接到单个 `PUSH` 套接字,则消息均匀分布在所有的 `PULL` 套接字中;在实践中,它们是负载均衡的(点对点负载平衡!)。另一方面,从多个 `PUSH` 套接字接收消息的 `PULL` 套接字将使用公平排队系统处理消息,这意味着它们将从所有负载是均衡的。
- 通过没有任何连接 `PULL` 套接字的 `PUSH` 套接字发送的消息不会丢失;他们排队等待生产者,直到一个节点联机并开始提取消息。

#### Building a distributed hashsum cracker with ØMQ(使用 ØMQ 构建一个分布式的 hashsum cracker)

`hashsum cracker`,一个使用暴力破解技术来尝试将给定的 `hashsum`(`MD5`,`SHA1` 等) 与给定字母表中每个可能的字符变体进行匹配的系统。这个算法的负载量是很高的。

对于我们的应用程序,我们希望通过一个节点来实现典型的并行管道,以在多个 `worker` 之间创建和分配任务,以及一个节点来收集所有结果。我们刚刚描述的系统可以使用以下体系结构在 `ØMQ` 中实现:

![hashsum cracker](/assets/img/hashsum_cracker.png)

在我们的体系结构中,我们有一个 `ventilator`,用于生成给定字母表中所有可能的字符变体,并将它们分发给一组 `worker`,然后计算每个给定变体的哈希函数并尝试将其与输入的哈希函数进行匹配。如果找到匹配项,则结果将发送到结果收集器节点(`sink`)。

重点是 `ventilator` 和 `sink`,而 `worker` 节点是随时在变化中的。这意味着每个 `worker` 将其 `PULL` 套接字连接到 `ventilator`,并将其 `PUSH` 套接字连接到 `ventilator`;通过这种方式,我们可以在不改变 `ventilator` 和 `sink` 中的任何参数的情况下,启动和停止我们想要的 `worker` 数量。

##### Implementing the ventilator

```js
const zmq = require('zmq')
const variationsStream = require('variations-stream')
const alphabet = 'abcdefghijklmnopqrstuvwxyz'
const batchSize = 10000
const maxLength = process.argv[2]
const searchHash = process.argv[3]

const ventilator = zmq.socket('push') // [1]
ventilator.bindSync('tcp://*:5016')

let batch = []
variationsStream(alphabet, maxLength)
  .on('data', combination => {
    batch.push(combination)
    if (batch.length === batchSize) {
      // [2]
      const msg = { searchHash: searchHash, variations: batch }
      ventilator.send(JSON.stringify(msg))
      batch = []
    }
  })
  .on('end', () => {
    //send remaining combinations
    const msg = { searchHash: searchHash, variations: batch }
    ventilator.send(JSON.stringify(msg))
  })
```

如何给 `worker` 分配任务:

1. 我们首先创建一个 `PUSH` 套接字,并将其绑定到本地端口 5000 ;这是 `worker` 的 `PULL` 套接字将连接以接收任务的地方。
2. 我们将每个批次生成的变体进行分组,然后制作一条消息,其中包含匹配的散列和要检查的一批单词。这实质上是 `worker` 将接受的任务对象。当我们通过 `ventilator` 套接字调用 `send()` 时,消息将按循环分配传递给下一个可用的 `worker`。

##### Implementing worker

```js
const zmq = require('zmq')
const crypto = require('crypto')
const fromVentilator = zmq.socket('pull')
const toSink = zmq.socket('push')

fromVentilator.connect('tcp://localhost:5016')
toSink.connect('tcp://localhost:5017')

fromVentilator.on('message', buffer => {
  const msg = JSON.parse(buffer)
  const variations = msg.variations
  variations.forEach(word => {
    console.log(`Processing: ${word}`)
    const shasum = crypto.createHash('sha1')
    shasum.update(word)
    const digest = shasum.digest('hex')
    if (digest === msg.searchHash) {
      console.log(`Found! => ${word}`)
      toSink.send(`Found! ${digest} => ${word}`)
    }
  })
})
```

正如我们所说的,我们的 `worker` 在我们的体系结构中代表了一个临时节点,因此,它的套接字应连接到远程节点,而不是侦听传入连接。这正是我们在 `worker` 中所做的,我们创建了两个套接字:

- 连接到 `ventilator` 的 `PULL` 套接字
- 用于接收任务连接到接收器的 `PUSH` 套接字,用于传播结果

除此之外,我们的 `worker` 完成的工作非常简单:对于收到的每条消息,我们迭代它包含的一批单词,然后对每个单词计算 `SHA1` 校验和,并尝试将其与针对消息传递的 `searchHash` 进行匹配。当找到匹配时,结果被转发到接收器。

##### Implementing sink

```js
const zmq = require('zmq')
const sink = zmq.socket('pull')
sink.bindSync('tcp://*:5017')

sink.on('message', buffer => {
  console.log('Message from worker: ', buffer.toString())
})
```

我们现在准备运行我们的应用程序;让我们开始几个 `worker` 和 `sink`:

```bash
node worker
node worker
node sink
```

然后启动 `ventilator` ,指定要生成的单词的最大长度以及我们希望匹配的 `SHA1` 校验和。以下是参数的示例列表:

```bash
node ventilator 4 f8e966d1e207d02c44511a58dccff2f5429e9a3b
```

当运行上述命令时,`ventilator` 将开始生成所有可能的单词,其长度至多为四个字符,并将它们分配给我们开始的工作人员,以及我们提供的校验和。计算结果(如果有的话)将显示在接收器应用程序的终端中。

### Pipelines and competing consumers in AMQP(AMQP 中管道和竞争消费者)

#### Point-to-point communications and competing consumers(点对点通信和竞争消费者)

在点对点的配置中，管道模式是非常容易理解的，但是如果使用消息代理，那么就很难去理解各个节点之间的关系了，我们并不知道另一边是谁在监听消息。如果我们想要使用 `AMQP` 来实现管道和任务分配模式，我们就必须确保每一个消息只能被一个消费者接收，但是这在交换机与多个队列绑定时是无法保证的。所以解决方案就是，将消息直接发给某一个队列，这样就能保证只有一个队列能接收到消息了。这样我们的目标就已经实现一半了。

#### Implementing the hashsum cracker with AMQP

![the hashsum cracker with AMQP](/assets/img/AMQP_hashsum_cracker.png)

##### Implementing the producer

```js
const amqp = require('amqplib')
const variationsStream = require('variations-stream')
const alphabet = 'abcdefghijklmnopqrstuvwxyz'
const batchSize = 10000
const maxLength = process.argv[2]
const searchHash = process.argv[3]

let connection, channel
amqp
  .connect('amqp://localhost')
  .then(conn => {
    connection = conn
    return conn.createChannel()
  })
  .then(ch => {
    channel = ch
    produce()
  })
  .catch(err => console.log(err))

function produce() {
  let batch = []
  variationsStream(alphabet, maxLength)
    .on('data', combination => {
      batch.push(combination)
      if (batch.length === batchSize) {
        const msg = { searchHash: searchHash, variations: batch }
        //只发送给 jobs_queue 队列，保证只有一个队列接收消息
        channel.sendToQueue('jobs_queue', new Buffer(JSON.stringify(msg)))
        batch = []
      }
    })
    .on('end', () => {
      //send remaining combinations
      const msg = { searchHash: searchHash, variations: batch }
      channel.sendToQueue(
        'jobs_queue',
        new Buffer(JSON.stringify(msg)),
        //when the last message is sent, close the connection
        //to allow the application to exit
        function() {
          channel.close()
          connection.close()
        }
      )
    })
}
```

##### Implementing the worker

```js
const amqp = require('amqplib')
const crypto = require('crypto')

let channel, queue
amqp
  .connect('amqp://localhost')
  .then(conn => conn.createChannel())
  .then(ch => {
    channel = ch
    // 只从 jobs_queue 中消费
    return channel.assertQueue('jobs_queue')
  })
  .then(q => {
    queue = q.queue
    consume()
  })
  .catch(err => console.log(err.stack))

function consume() {
  channel.consume(queue, function(msg) {
    const data = JSON.parse(msg.content.toString())
    const variations = data.variations
    variations.forEach(word => {
      console.log(`Processing: ${word}`)
      const shasum = crypto.createHash('sha1')
      shasum.update(word)
      const digest = shasum.digest('hex')
      if (digest === data.searchHash) {
        console.log(`Found! => ${word}`)
        //结果发送给 results_queue，使用点对点通信
        channel.sendToQueue(
          'results_queue',
          new Buffer(`Found! ${digest} => ${word}`)
        )
      }
    })
    channel.ack(msg)
  })
}
```

##### Implementing the result collector

```js
const amqp = require('amqplib')

let channel, queue
amqp
  .connect('amqp://localhost')
  .then(conn => conn.createChannel())
  .then(ch => {
    channel = ch
    return channel.assertQueue('results_queue')
  })
  .then(q => {
    queue = q.queue
    channel.consume(queue, msg => {
      console.log('Message from worker: ', msg.content.toString())
    })
  })
  .catch(err => console.log(err.stack))
```

##### Running the application

运行应用：

```bash
node worker
node worker

node collector
node producer 4 f8e966d1e207d02c44511a58dccff2f5429e9a3b
```

## Request/reply patterns

单向异步消息可以在并发行和效率上带来很大帮助，但有时候我们也需要一个“请求/回复”模式来帮助我们解决剩下的问题。

### Correlation identifier(相关 ID)

该模式包括标记每个请求的标识符(`ID`),然后由接收方附加到响应中;通过这种方式,请求的发送者可以关联这两个消息并将响应返回给正确的处理程序。这优雅地解决了存在单向异步通道的问题,消息可以随时在任何方向传播:

![correlation identifier](/assets/img/correlation_identifier.png)

#### Implementing a request/reply abstraction using correlation identifier(使用关联 ID 实现 “请求/回复” 抽象)

##### Abstracting request(抽象请求)

```js
const uuid = require('node-uuid')

module.exports = channel => {
  const idToCallbackMap = {} // [1]

  channel.on('message', message => {
    // [2]
    const handler = idToCallbackMap[message.inReplyTo]
    if (handler) {
      handler(message.data)
    }
  })

  return function sendRequest(req, callback) {
    // [3]
    const correlationId = uuid.v4()
    idToCallbackMap[correlationId] = callback
    channel.send({
      type: 'request',
      data: req,
      id: correlationId
    })
  }
}
```

1. `request` 函数创建一个闭包。该模式的神奇之处在于 `idToCallbackMap` 变量,它存储了传出请求与其回复处理程序之间的关联。
2. 一旦工厂被调用,我们所做的第一件事就是开始监听收到的消息。如果消息的关联 `ID` (包含在 `inReplyTo` 属性中)与 `idToCallbackMap` 变量中包含的任何 `ID` 相匹配,我们知道我们刚收到一个回复,因此我们获得了对相关响应处理程序的引用,并且用 消息中包含的数据。
3. 最后,我们返回我们将用来发送新请求的函数。 其工作是使用 `node-uuid` 生成关联 `ID`,然后将请求数据包装起来,并指定关联 `ID correlationId` 和消息类型 `type`。

##### Abstracting reply(抽象回复)

```js
module.exports = channel => {
  return function registerHandler(handler) {
    channel.on('message', message => {
      if (message.type !== 'request') return
      handler(message.data, reply => {
        channel.send({
          type: 'response',
          data: reply,
          inReplyTo: message.id
        })
      })
    })
  }
}
```

##### Trying the full request/reply cycle(尝试完整的 “请求/回复” 环路)

```js
const replier = require('child_process').fork(`${__dirname}/replier.js`)
const request = require('./request')(replier)
request({ a: 1, b: 2, delay: 500 }, res => {
  console.log('1 + 2 = ', res.sum)
  // 这应该是我们收到的最后一个回复,所以我们关闭了channel
  replier.disconnect()
})
request({ a: 6, b: 1, delay: 100 }, res => {
  console.log('6 + 1 = ', res.sum)
})
```

至此我们的应用就完成了。

### Return address(返回地址)

关联 `ID` 是在单向信道之上创建 “请求/回复” 通信的基本模式;然而,当我们的消息架构拥有多个通道或队列,或者可能有多个请求者时,这还不够。在这些情况下,除了关联 `ID` 之外,我们还需要知道返回地址,这是允许回复者将回复发送回请求的原始发件人的一条信息。

#### Implementing the return address with AMQP

![the return address with AMQP](/assets/img/AMQP_return_address.png)

实现：

```js
// amqpRequest.js
const uuid = require('node-uuid')
const amqp = require('amqplib')

class AMQPRequest {
  constructor() {
    this.idToCallbackMap = {}
  }

  initialize() {
    return amqp
      .connect('amqp://localhost')
      .then(conn => conn.createChannel())
      .then(channel => {
        this.channel = channel
        // 没有指定名字，当连接关闭时销毁队列
        // 没有绑定交换机，因为我们只有一个队列
        return channel.assertQueue('', { exclusive: true })
      })
      .then(q => {
        // 监听回复，使用 channel.consume，replyQueue 指定回复的队列，q.queue 是随机生成的名字，因为 assertQueue 参数为空
        this.replyQueue = q.queue
        return this._listenForResponses()
      })
      .catch(function(err) {
        console.log(err)
      })
  }

  _listenForResponses() {
    // 收到回复后，调用回调函数
    return this.channel.consume(
      this.replyQueue,
      msg => {
        const correlationId = msg.properties.correlationId
        const handler = this.idToCallbackMap[correlationId]
        if (handler) {
          handler(JSON.parse(msg.content.toString()))
        }
      },
      { noAck: true }
    )
  }

  request(queue, message, callback) {
    const id = uuid.v4()
    // 设置对应关联 ID 的回调函数
    this.idToCallbackMap[id] = callback
    // 使用 sendToQueue 而不是 publish，因为我们希望是点对点直接传输
    this.channel.sendToQueue(queue, new Buffer(JSON.stringify(message)), {
      correlationId: id,
      replyTo: this.replyQueue
    })
  }
}

module.exports = () => new AMQPRequest()

// amqpReply.js
const amqp = require('amqplib')

class AMQPReply {
  constructor(qName) {
    this.qName = qName
  }

  initialize() {
    return amqp
      .connect('amqp://localhost')
      .then(conn => conn.createChannel())
      .then(channel => {
        this.channel = channel
        return this.channel.assertQueue(this.qName)
      })
      .then(q => (this.queue = q.queue))
      .catch(err => console.log(err.stack))
  }

  handleRequest(handler) {
    return this.channel.consume(this.queue, msg => {
      const content = JSON.parse(msg.content.toString())
      handler(content, reply => {
        this.channel.sendToQueue(
          msg.properties.replyTo, //消息中带了要求回复的队列名字和关联 ID
          new Buffer(JSON.stringify(reply)),
          { correlationId: msg.properties.correlationId }
        )
        this.channel.ack(msg)
      })
    })
  }
}

module.exports = (excName, qName, pattern) => {
  return new AMQPReply(excName, qName, pattern)
}

// requestor.js
const req = require('./amqpRequest')()

req.initialize().then(() => {
  // 随机发送 100 个 请求到 requests_queue 队列
  for (let i = 100; i > 0; i--) {
    sendRandomRequest()
  }
})

function sendRandomRequest() {
  const a = Math.round(Math.random() * 100)
  const b = Math.round(Math.random() * 100)
  req.request('requests_queue', { a: a, b: b }, res => {
    console.log(`${a} + ${b} = ${res.sum}`)
  })
}

// replier.js
const Reply = require('./amqpReply')
const reply = Reply('requests_queue') //监听 requests_queue 队列

reply.initialize().then(() => {
  reply.handleRequest((req, cb) => {
    console.log('Request received', req)
    cb({ sum: req.a + req.b })
  })
})
```
