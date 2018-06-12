---
title: Node 服务部署
date: 2018-06-09 09:36:22
tags:
    - Node
---

## forever 与 pm2

在之前部署 `Node.js` 服务都是使用 `forever` 的，现在基本上都改为 `pm2` 了，`pm2` 比 `forever` 功能更加强大，可以配置集群、集成日志、控制台监视等等，下面是两者的比较：

| Feature             | Forever | PM2 |
| ------------------- | ------- | --- |
| Keep Alive          | ✔       | ✔   |
| Coffeescript        | ✔       |     |
| Log aggregation     |         | ✔   |
| API                 |         | ✔   |
| Terminal monitoring |         | ✔   |
| Clustering          |         | ✔   |
| JSON configuration  |         | ✔   |

<!-- more -->

所以如果还在使用 `forever` 的读者，快快加入到 `pm2` 的阵营来吧。

## pm2 使用

详情请看[官方文档](https://pm2.io/doc/en/runtime/overview/)

### 配置

`pm2` 可以直接在命令行中通过参数配置，也可以用过 `ecosystem.config.js` 文件来配置，`pm2 init` 初始化一个配置文件如下：

```js
module.exports = {
  apps : [{
    name: "app",
    script: "./app.js",
    env: {
      NODE_ENV: "development",
    },
    env_production: {
      NODE_ENV: "production",
    }
  }]

  // deploy config(部署配置也在这里)
  ...
}
```

### 集成 log

可以单独配置普通输出和错误输出到不同的文件：

```js
module.exports = {
  apps: [
    {
      name: 'app',
      script: 'app.js',
      output: './out.log',
      error: './error.log',
      log: './combined.outerr.log'
    }
  ]
}
```

同时也可以集成 `pm2-logrotate` 工具来管理日志，可以配置日志分片大小、时间、名字等等。

### 自启动

`pm2` 可以配置开机启动，通过 `pm2 startup` 命令会自动提示你该怎么配置自启动。

### 负载均衡

`pm2` 支持集群模式，可以在不更改代码的前提下自动帮你配置集群，有效利用系统的 `CPU` 资源，充分利用计算机能力，消除 `Node.js` 单线程的瓶颈，可以同时启动多个进程来监听同一个端口(这在文后会提到具体怎么实现多进程监听)，提高性能。

```zsh
-i <number-instances>
```

或者配置 `ecosystem.config.js` 文件：

```js
module.exports = {
  apps: [
    {
      script: 'app.js',
      instances: 'max'
    }
  ]
}
```

### SSH 部署

`pm2` 还支持自动化将代码部署到远程服务器，只需要做一些简单的配置即可。

```js
module.exports = {
  apps: [{
    name: "app",
    script: "app.js"
  }],
  deploy: {
    // "production" is the environment name
    production: {
      // SSH key path, default to $HOME/.ssh
      key: "/path/to/some.pem",
      // SSH user
      user: "ubuntu",
      // SSH host
      host: ["192.168.0.13"],
      // SSH options with no command-line flag, see 'man ssh'
      // can be either a single string or an array of strings
      ssh_options: "StrictHostKeyChecking=no",
      // GIT remote/branch
      ref: "origin/master",
      // GIT remote
      repo: "git@github.com:Username/repository.git",
      // path in the server
      path: "/var/www/my-repository",
      // Pre-setup command or path to a script on your local machine
      pre-setup: "apt-get install git ; ls -la",
      // Post-setup commands or path to a script on the host machine
      // eg: placing configurations in the shared dir etc
      post-setup: "ls -la",
      // pre-deploy action
      pre-deploy-local: "echo 'This is a local executed command'"
      // post-deploy action
      post-deploy: "npm install",
    },
  }
}
```

### 注意事项

1.  如果使用集群模式需要主要应用得是个无状态应用，所以诸如 `sessions`、`websocket connection` 等不要使用，可以使用 `Redis` 等来共享应用的状态。
2.  关闭应用之前最好确认所有的请求已经被处理，数据库连接已经释放，释放其他资源。

## cluster 原理

`pm2` 集群其实是封装了 `cluster` 模块的一系列操作，自动包装代码启动集群，重加载的时候也是先启动新的 `worker` 再把之前的 `worker` 停掉等等。

这里是一个简单的 `cluster` 的示例：

```js
const cluster = require('cluster')
const os = require('os')

if (cluster.isMaster) {
  for (let i = 0, n = os.cpus().length; i < n; i += 1) {
    cluster.fork()
  }
} else {
  http
    .createServer(function(req, res) {
      res.writeHead(200)
      res.end('hello world\n')
    })
    .listen(8000)
}
```

`fork` 的本质还是使用了 `child_process.fork` 生成的子进程，但是注意不能直接使用 `child_process.fork()` 来生成，因为这样会缺少 `process.env.NODE_UNIQUE_ID`，会导致 `cluster.isMaster` 判断总是为 `true`。

运行时，所有新建立的连接都由主进程完成，然后主进程再把 `TCP` 连接分配给指定的 `worker` 进程。分配根据 `Round-robin` 算法分发，子进程 `worker` 具体处理请求。

更加详细的 `cluser` 用法与讲解可查看[阮一峰的博客](http://javascript.ruanyifeng.com/nodejs/cluster.html) 和 [官方文档](https://nodejs.org/dist/latest-v10.x/docs/api/cluster.html)。
