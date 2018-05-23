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

## Dubbo 简介

`Apache Dubbo|ˈdʌbəʊ|` 是阿里开源的一个 RPC 框架。

和大多数 RPC 系统一样， `dubbo` 基于一个理念：定义一个服务，确定远程调用的方法，并且包含他们的参数和返回类型。在服务端，服务器实现接口并且运行一个 `dubbo` 的服务来处理来自客户端的请求；在客户端，客户端持有提供与服务端方法一模一样的桩。

<!-- more -->

![dubbo 架构](/assets/img/dubbo_architecture.png)

`Apache Dubbo(incubating)`提供三个关键的功能:

* 基于接口的远程调用
* 错误容忍和负载均衡
* 自动化服务注册和发现

详细的架构描述请查看[官方文档](http://dubbo.apache.org/books/dubbo-user-book/preface/architecture.html)。

## Dubbo 实现

### Invoker

Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

### 远程调用细节

#### 服务提供者暴露一个服务的详细过程

![dubbo rpc export](/assets/img/dubbo_rpc_export.jpg)

`ServiceConfig` 类拿到对外提供服务的实际类 `ref`(如：`DemoServiceImpl`),然后通过 `ProxyFactory` 类的 `getInvoker` 方法使用 `ref` 生成一个 `AbstractProxyInvoker` 实例，然后在通过 `Protocol` 中的 `export` 方法将 `Invoker`  转换为 `Exporter`。

这里举一个将 `Invoker` 转为 `Exporter` 的例子，`HttpProtocol`：

```java
public class HttpProtocol extends AbstractProxyProtocol {
    //other code
    @Override
    protected <T> Runnable doExport(final T impl, Class<T> type, URL url) throws RpcException {
        String addr = getAddr(url);
        HttpServer server = serverMap.get(addr);
        if (server == null) {
            server = httpBinder.bind(url, new InternalHandler());
            serverMap.put(addr, server);
        }
        final HttpInvokerServiceExporter httpServiceExporter = new HttpInvokerServiceExporter();
        httpServiceExporter.setServiceInterface(type);
        httpServiceExporter.setService(impl);
        try {
            httpServiceExporter.afterPropertiesSet();
        } catch (Exception e) {
            throw new RpcException(e.getMessage(), e);
        }
        final String path = url.getAbsolutePath();
        skeletonMap.put(path, httpServiceExporter);
        return new Runnable() {
            @Override
            public void run() {
                skeletonMap.remove(path);
            }
        };
    }
}
```

#### 服务消费者消费一个服务的详细过程

![dubbo rpc refer](/assets/img/dubbo_rpc_refer.jpg)

`ReferenceConfig` 类的 `init` 方法调用 `Protocol` 的 `refer` 方法生成 `Invoker` 实例(如上图中的红色部分)，这是服务消费的关键。接下来把 `Invoker` 转换为客户端需要的接口(如`DemoService`)。而客户端需要调用的时候只需要调用这个接口(如`DemoService`)，就能够间接使用 `Invoker` 来调用远程的方法。

#### Invoke 的过程

![dubbo rpc invoke](/assets/img/dubbo_rpc_invoke.jpg)

为了更好的解释上面这张图，我们结合服务消费和提供者的代码示例来进行说明：

服务消费者代码：

```java
public class DemoClientAction {

    private DemoService demoService;

    public void setDemoService(DemoService demoService) {
        this.demoService = demoService;
    }

    public void start() {
        String hello = demoService.sayHello("world" + i);
    }
}
```

上面代码中的 `DemoService` 就是上图中服务消费端的 `proxy`，用户代码通过这个 `proxy` 调用其对应的 `Invoker`，而该 `Invoker` 实现了真正的远程服务调用。

我们看一下 `DubboInvoker`(基于 dubbo 协议) 的具体实现：

```java
public class DubboInvoker<T> extends AbstractInvoker<T> {
    //other code
    //...
    @Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);

        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
                ResponseFuture future = currentClient.request(inv, timeout);
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
                RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
}
```

可以看到是 `Invoker` 真正实现了远程调用。

服务提供者代码：

```java
public class DemoServiceImpl implements DemoService {

    public String sayHello(String name) throws RemoteException {
        return "Hello " + name;
    }
}
```

上面这个类会被封装成为一个 `AbstractProxyInvoker` 实例，并新生成一个 `Exporter` 实例。这样当网络通讯层收到一个请求后，会找到对应的 `Exporter` 实例，并调用它所对应的 `AbstractProxyInvoker` 实例，从而真正调用了服务提供者的代码。

#### 远程过程调用总结

总的来看，可以将 `dubbo` 中的远程过程调用总结为：服务端通过配置文件或代码做一个配置，接着根据读取的配置将接口一个个封装起来并暴露(`Exporter`)，启动服务并遵守定义的协议；客户端通过配置文件或代码做类似的配置，接着根据定义的协议生成对应的调用者(`Invoker`)，调用者中隐藏了真正的远程调用细节，之后便可以通过调用者调用远程的方法了。

例如，通过 `dubbo://11.240.240.118:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=11.240.240.118&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=34270&qos.port=22222&side=provider&timestamp=1527037622366` 可以发现，服务器提供一个 `DemoService` 的接口，和一个 `sayHello` 的方法；通过 `dubbo://11.240.240.118:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-consumer&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=34452&qos.port=33333&register.ip=11.240.240.118&remote.timestamp=1527037622366&side=consumer&timestamp=1527038134040` 客户端拿到对应的可调用的接口 `DemoService`，然后调用 `DemoService` 中的方法(`sayHello`)，接着调用到客户端的 `Invoker`(这里为`DubboInvoker`)，`Invoker` 再根据协议将包装的信息传递到服务器，服务器解析信息找到对应的 `Exporter`，再找到 `AbstractProxyInvoker` 的实例，然后成功调用接口的实现类 `DemoServiceImpl`，完成调用，这就是整个过程。

###  扩展点加载

`Dubbo` 的扩展点加载从 `JDK` 标准的 `SPI` (Service Provider Interface) 扩展点发现机制加强而来。

传统的 `SPI` 发现机制是根据在 `jar` 包中的 `META-INF/services/` 配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk 提供服务实现查找的一个工具类： `java.util.ServiceLoader`。

我们看看是如何加强的:

#### 自适应加强

`jdk` 使用 `ServiceLoader`， `Dubbo` 使用`com.alibaba.dubbo.common.extension.ExtensionLoader` 来提供服务实现查找，`ExtensionLoader` 注入的依赖扩展点是一个 `Adaptive` 实例，直到扩展点方法执行时才决定调用是一个扩展点实现。

看一个利用 `Adaptive` 生成的 `DubboProtocol$Adaptive` 代码：

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

注入的就是上面这类的实例，可以发现在使用过程中，如 `export()` 方法在调用过程中或通过 `ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName)` 来获取到真正的实例再进行调用，这就保证了在真正调用之前，实例是不会被真正创建的(只创建了对应的`Adaptive`实例)，如果有扩展实现初始化很耗时，没用上也不会加载，从而减少资源浪费。

<blockquote>另外：在阅读源码的过程中，受到了很大的阻碍，这里很大一部分原因是在 `ServiceConfig` 中有段代码：
```java
final int defaultPort = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(name).getDefaultPort();
```
当我看到这段代码时有个疑惑就冒出来了，如果在调用的时候都使用 `getExtension` 来获取实例的话那还需要 `Adaptive` 实例干嘛？我看了下 `Protocol` 的代码，发现里面 `getDefaultPort()` 是没有加 `@Adaptive` 注解的，这可能就是这段代码的来源了。我把 `getDefaultPort` 加上注解后运行报错：`Can not create adaptive extension interface com.alibaba.dubbo.rpc.Protocol, cause: fail to create adaptive class for interface com.alibaba.dubbo.rpc.Protocol: not found url parameter or url attribute in parameters of method getDefaultPort`，仔细研究后发现，`Adaptive` 动态加载实例的前提是通过url参数来找到对应的具体类的，而在 `getDefaultPort` 方法中是不需要url参数的，所以是不能通过 `Adaptive` 实例来动态调用的，这就是上述代码出现的真正原因了！

随后发现，其他使用 `@Adaptive` 注解的类都是所有方法都加上注解的，没有像 `Protocol` 这种有部分方法是无 `@Adaptive` 注解的，这也是符合逻辑的，因为如果一个类中的方法要分是否加了 `@Adaptive` 注解来用不同的方式调用这可读性以及可维护性就非常差了(就像我刚开始  看到那段代码一样是非常疑惑的)。

</blockquote>

#### 增加 IOC 与 AOP

**IOC**

只需要简单配置(如：`dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol`)就可以自动地创建想要的对象。

**AOP**

自动包装扩展点的 `Wrapper` 类(如 `ProtocolListenerWrapper`)，在 `Wrapper` 中可以做一些公共的逻辑。例如：

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
