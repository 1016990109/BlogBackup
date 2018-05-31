---
title: Dubbo 探索
date: 2018-05-22 11:09:44
update: 2018-05-30 20:00:00
tags:
    - java
    - RPC
---

# Dubbo 框架的介绍以及源码阅读

## Dubbo 简介

`Apache Dubbo|ˈdʌbəʊ|` 是阿里开源的一个 RPC 框架。

和大多数 `RPC` 系统一样， `dubbo` 基于一个理念：定义一个服务，确定远程调用的方法，并且包含他们的参数和返回类型。在服务端，服务器实现接口并且运行一个 `dubbo` 的服务来处理来自客户端的请求；在客户端，客户端持有提供与服务端方法一模一样的桩。

<!-- more -->

![dubbo 架构](/assets/img/dubbo_architecture.png)

`Apache Dubbo(incubating)`提供三个关键的功能:

* 基于接口的远程调用
* 错误容忍和负载均衡
* 自动化服务注册和发现

详细的架构描述请查看[官方文档](http://dubbo.apache.org/books/dubbo-user-book/preface/architecture.html)。

## Dubbo 实现

### Dubbo 接入 spring

`spring` 可以在 `xml` 中做一些配置:

```xml
<context:component-scan base-package="com.demo.dubbo.server.serviceimpl"/>
```

对于上述的 xml 配置，分成三个部分:

* 命名空间 `namespace`，如 `context`
* 元素 `element`，如 `component-scan`
* 属性 `attribute`，如 `base-package`

`spring` 定义了两个接口，来分别解析上述内容：

* `NamespaceHandler`：注册了一堆 `BeanDefinitionParser`，利用他们来进行解析
* `BeanDefinitionParser`: 用于解析每个 `element` 的内容

`spring` 会从 `jar` 包下的 `META-INF/spring.handlers` 文件下寻找 `NamespaceHandler`，所以如果需要自定义配置，只需要在 `jar` 包下加入 `META-INF/spring.handlers` 文件，其中记录 `NamespaceHandler` 的实现类，`dubbo` 的 `spring.handlers` 文件内容如下：

```
http\://dubbo.apache.org/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```

然后不同的配置分别转换成 `spring` 容器中的一个 `bean` 对象：

* `application` 对应 `ApplicationConfig`
* `registry` 对应 `RegistryConfig`
* `monitor` 对应 `MonitorConfig`
* `provider` 对应 `ProviderConfig`
* `consumer` 对应 `ConsumerConfig`
* `protocol` 对应 `ProtocolConfig`
* `service` 对应 `ServiceBean`(继承 `ServiceConfig`)
* `reference` 对应 `ReferenceBean`(继承 `ReferenceConfig`)

### 概念介绍

`Invoker` 是实体域，它是 `Dubbo` 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 `invoke` 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

```java
public interface Invoker<T> {

    Class<T> getInterface();

    URL getUrl();

    Result invoke(Invocation invocation) throws RpcException;

	void destroy();

}
```

而 `Invocation` 则包含了需要执行的方法、参数等信息，接口定义简略如下：

```java
public interface Invocation {

    URL getUrl();

	String getMethodName();

	Class<?>[] getParameterTypes();

	Object[] getArguments();

}
```

### 发布服务

![dubbo rpc export](/assets/img/dubbo_rpc_export.jpg)

`ServiceConfig` 通过配置文件拿到对外提供服务的实现类 `ref`(如：`DemoServiceImpl`),然后通过 `ProxyFactory` 的 `getInvoker` 方法根据 `ref` 生成一个 `AbstractProxyInvoker` 实例，然后在通过 `Protocol` 的 `export` 方法将 `Invoker`  转换为 `Exporter`。

这里举一个将 `Invoker` 转为 `Exporter` 的例子，`DubboProtocol`：

```java
public class DubboProtocol extends AbstractProxyProtocol {
    //other code
    ...

    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {

        // export service.
        String key = serviceKey(url);
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        ...
        return exporter;
    }
}
```

### 消费服务

![dubbo rpc refer](/assets/img/dubbo_rpc_refer.jpg)

`ReferenceConfig` 类的 `init` 方法调用 `Protocol` 的 `refer` 方法生成 `Invoker` 实例(如上图中的红色部分)，这是服务消费的关键。接下来把 `Invoker` 转换为客户端需要的接口(如`DemoService`)。而客户端需要调用的时候只需要调用这个接口(如`DemoService`)，就能够间接使用 `Invoker` 来调用远程的方法。

### Invoke 的过程

![dubbo rpc invoke](/assets/img/dubbo_rpc_invoke.jpg)

<!-- #### 远程过程调用总结

总的来看，可以将 `dubbo` 中的远程过程调用总结为：服务端根据读取的配置将接口一个个封装起来并暴露(`Exporter`)，启动服务并遵守定义的协议；客户端通过配置文件的配置，接着根据定义的协议生成对应的调用者(`Invoker`)，调用者中隐藏了真正的远程调用细节，之后便可以通过调用者调用远程的方法了。

例如，通过 `dubbo://11.240.240.118:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=11.240.240.118&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=34270&qos.port=22222&side=provider&timestamp=1527037622366` 可以发现，服务器提供一个 `DemoService` 的接口，和一个 `sayHello` 的方法；通过 `dubbo://11.240.240.118:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-consumer&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=34452&qos.port=33333&register.ip=11.240.240.118&remote.timestamp=1527037622366&side=consumer&timestamp=1527038134040` 客户端拿到对应的可调用的接口 `DemoService`，然后调用 `DemoService` 中的方法(`sayHello`)，接着调用到客户端的 `Invoker`(这里为`DubboInvoker`)，`Invoker` 再根据协议将包装的信息传递到服务器，服务器解析信息找到对应的 `Exporter`，再找到 `AbstractProxyInvoker` 的实例，然后成功调用接口的实现类 `DemoServiceImpl`，完成调用，这就是整个过程。 -->

###  扩展点加载

`Dubbo` 的扩展点加载从 `JDK` 标准的 `SPI` (Service Provider Interface) 扩展点发现机制加强而来。

传统的 `SPI` 发现机制是根据在 `jar` 包中的 `META-INF/services/` 配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk 提供服务实现查找的一个工具类： `java.util.ServiceLoader`。

#### Dubbo 的 ExtensionLoader 解析扩展过程

`jdk` 使用 `ServiceLoader`， `Dubbo` 使用`com.alibaba.dubbo.common.extension.ExtensionLoader` 来提供服务实现查找，`ExtensionLoader` 注入的依赖扩展点是一个 `Adaptive` 实例，直到扩展点方法执行时才决定调用是一个扩展点实现。

以下面例子为例：

```java
ExtensionLoader<Protocol> protocolLoader=ExtensionLoader.getExtensionLoader(Protocol.class);
Protocol  protocol=protocolLoader.getAdaptiveExtension();
```

```java
@Extension("dubbo")
public interface Protocol {

    int getDefaultPort();

    @Adaptive
	<T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

	void destroy();

}
```

首先根据要加载的接口创建出一个 ExtensionLoader 实例，然后再获取自适应的 `Protocol` 实现类(`DubboProtocol$Adaptive`)。

`getAdaptiveExtension` 会根据 `@Adaptive` 注解去动态生成 `DubboProtocol$Adaptive` 实例， `DubboProtocol$Adaptive` 代码如下：

```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
}
```

可以发现在使用过程中，如 `export()` 方法在调用过程中或通过 `ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName)` 来获取到真正的实例再进行调用，这就保证了在真正调用之前，实例是不会被真正创建的(只创建了对应的`Adaptive`实例)，如果有扩展实现初始化很耗时，没用上也不会加载，从而减少资源浪费。

#### ExtensionLoader 实例是如何来加载 Protocol 的实现类的：

1.先解析 Protocol 上的 Extension 注解的 name,存至 String cachedDefaultName 属性中，作为默认的实现

2.到类路径下的加载所有的 META-INF/dubbo.interval.com.alibaba.dubbo.rpc.Protocol 文件，例如 `dubbo-rpc-dubbo` 模块下的 Protocol 文件内容如下：

```
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
```

然后读取内容加载对应的 `class`(`DubboProtocol`)，并和对应的 `name`(上面`=`前面的字符`dubbo`) 做关联，为以后根据 `name` 找具体实现类做铺垫。

#### ExtensionLoader 获取扩展过程

```java
    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```

大致分成 4 步：

1.根据 name 获取对应的 class

2.根据获取到的 class 创建一个实例

3.对获取到的实例，进行依赖注入

4.对于上述经过依赖注入的实例，再次进行包装，实现 `AOP`。以 `Protocol` 为例，`ProtocolFilterWrapper`、`ProtocolListenerWrapper` 会对 `DubboProtocol` 进行包装：

```java
public class XxxProtocolWrapper implemenets Protocol {
    Protocol impl;

    public XxxProtocol(Protocol protocol) { impl = protocol; }

    // 接口方法做一个操作后，再调用extension的方法
    public void refer() {
        //... 一些操作
        impl.refer();
        // ... 一些操作
    }

    // ...
}
```

#### 一个扩展点可以直接 `setter` 注入其它扩展点

对应的处理在 `ExtensionLoader` 中：

```java
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

### Dubbo 通信

`Dubbo` 已经集成的有 `Netty`、`Mina`，默认是 `Netty`，这里主要介绍 `Netty`。

#### NIO

`Netty` 使用 `Reactor` 主从模型结构(三种 `Reactor` 模型详情请看[这里](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf))的变种:

![reactor3](/assets/img/reactor3.png)

去掉上面的线程池就为 `Netty` 的默认模式了。

`Netty` 里对应 `mainReactor` 的角色叫做 `Boss`，而对应 `subReactor` 的角色叫做 `Worker`。`Boss` 负责分配请求，创建 Selector，用于不断监听 Socket 连接、客户端的读写操作等；`Worker` 负责执行，负责处理 Selector 派发的读写操作。

`Netty` 中 `Reactor` 模式的参与者主要有下面一些组件：

![reactor](/assets/img/reactor.jpg)

* `Selector`(对应多路复用器 `Demultiplexer`)
* `EventLoopGroup/EventLoop`(对应 `Reactor` 模式中的分发者 `Dispatcher`)
* `ChannelPipeline`(对应请求处理器 `Handler`，真正干活的)

不管是 `Boos` 线程还是 `Worker` 线程，所做的事情均分为以下三个步骤: 
1. 轮询注册在 `selector` 上的 `I/O` 事件 
2. 处理 `I/O` 事件 
3. 执行异步 `task`

对于 `Boos` 线程来说，第一步轮询出来的基本都是 `accept` 事件，表示有新的连接，而 `Worker` 线程轮询出来的基本都是 `read/write` 事件，表示网络的读写事件。

新连接的建立 
1. `Boss` 的 `Selector` 检测到有新的连接 
2. 将新的连接注册到 `Worker` 线程组 
3. 注册新连接的读事件到 `Worker` 的 `Selector` 中

新连接的读取和请求处理

1.  数据准备好了
2.  `Worker` 知道了，同步调用 `unsafe.read` 获得客户端传输的数据，交给 `ChannelPipeline` 处理
3.  `ChannelPipeline` 处理，`decode` -> 处理数据 -> `encode` 结果，这些过程都是异步的
4.  用户调用 `channel.writeAndFlush`，写就绪
5.  `Worker` 知道了，`unsafe.forceflush` 写回结果给客户端

`ChannelPipeline` 处理过程类似下面这样，一般会有 `decode`，用户自定义的 `handler`，和 `encode`：

![netty pipelin](/assets/img/netty_pipeline.png)

`ChannelInBoundHandler` 对从客户端发往服务器的报文进行处理，一般用来执行拆包/粘包，解码，读取数据，业务处理等；`ChannelOutBoundHandler` 对从服务器发往客户端的报文进行处理，一般用来进行编码，发送报文到客户端。

#### 编码与解码(序列化与反序列化)

`Dubbo` 默认是使用 `Hessian` 作为序列化与反序列化的工具的：

```java
@SPI("hessian2")
public interface Serialization {

    /**
     * get content type id
     *
     * @return content type id
     */
    byte getContentTypeId();

    /**
     * get content type
     *
     * @return content type
     */
    String getContentType();

    /**
     * create serializer
     *
     * @param url
     * @param output
     * @return serializer
     * @throws IOException
     */
    @Adaptive
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;

    /**
     * create deserializer
     *
     * @param url
     * @param input
     * @return deserializer
     * @throws IOException
     */
    @Adaptive
    ObjectInput deserialize(URL url, InputStream input) throws IOException;

}
```

