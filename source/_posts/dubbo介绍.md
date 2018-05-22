---
title: dubbo 探索
date: 2018-05-22 11:09:44
tags:
    - java
    - 框架
    - 开源
    - 微服务
    - RPC
---
# Dubbo 框架的介绍以及源码阅读

`Apache Dubbo|ˈdʌbəʊ|` 是阿里开源的一个RPC框架。

和大多数RPC系统一样，`dubbo`基于一个理念：定义一个服务，确定远程调用的方法，并且包含他们的参数和返回类型。在服务端，服务器实现接口并且运行一个`dubbo`的服务来处理来自客户端的请求；在客户端，客户端持有提供与服务端方法一模一样的桩。

<!-- more -->

![dubbo 架构](/assets/img/dubbo_architecture.png)

`Apache Dubbo(incubating)`提供三个关键的功能:

- 基于接口的远程调用
- 错误容忍和负载均衡
- 自动化服务注册和发现

详细的架构描述请查看[官方文档](http://dubbo.apache.org/books/dubbo-user-book/preface/architecture.html)。