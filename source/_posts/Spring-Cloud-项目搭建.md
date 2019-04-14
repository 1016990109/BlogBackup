---
title: Spring Cloud 项目搭建(基于 Greenwich.M3 版本)
date: 2019-01-29 17:44:41
tags:
  - spring cloud
---

`Spring Cloud` 是一个基于 `Spring Boot` 实现的云应用开发工具，它为基于 `JVM` 的云应用开发中的配置管理、服务发现、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。

`Spring Cloud` 包含了多个子项目（针对分布式系统中涉及的多个不同开源产品），比如：`Spring Cloud Config`、`Spring Cloud Netflix`、`Spring Cloud Data`、`Spring Cloud AWS`、`Spring Cloud Security`、`Spring Cloud Commons`、`Spring Cloud Gateway`、`Spring Cloud CLI` 等项目。

下面分别从以下几块来开始搭建我们的 `Spring Cloud` 应用（基于 `Greenwich.M3` 版本，建议读者使用稳定版本，这里只是尝试新特性，文末有项目地址）：

- 公共父项目（`spring-cloud-example`）
- 注册中心（`eureka`）
- 配置中心（`config-server`）
- 网关（`gateway`）
- 服务提供者（`feign-server`）
- 服务消费者（`feign-client`）

<!-- more -->

## 公共父项目

开发 `Java` 项目一般使用 `Intellij Idea` 开发，创建项目的时候勾选 `Spring Initializr`，使用默认的服务 `https://start.spring.io` 即可。接下来一步步按照提示操作，选择 `Spring Boot` 版本为 `2.1.1.RELEASE`，接着选择需要的依赖，勾选 `Ops` 的 `Actuator`（可检测服务的健康状态等等，具体功能可查看 [https://cloud.spring.io/spring-cloud-static/Greenwich.RELEASE/multi/multi\_\_actuator_api.html](https://cloud.spring.io/spring-cloud-static/Greenwich.RELEASE/multi/multi__actuator_api.html)）。生成的 `pom.xml` 文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.edu.nju</groupId>
    <artifactId>spring-cloud-example</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>

    <name>${artifactId}</name>
    <description>spring cloud parent</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.1.RELEASE</version><!-- !!!important!!! 2.1.0.RELEASE 版本 spring-cloud-bus 的 spring-cloud-config-monitor 不会发送消息到bus上-->
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.M3</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <reporting>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-pmd-plugin</artifactId>
            </plugin>
        </plugins>
    </reporting>
</project>
```

![Idea Spring IO](/assets/img/idea_spring_io.png)
![Idea Spring IO](/assets/img/idea_spring_io_metadata.png)
![Idea Spring IO](/assets/img/idea_spring_io_dependencies.png)

## 注册中心

微服务框架的核心就是注册中心，有了注册中心后各个服务可以将多个实例都注册到注册中心，需要调用时可根据需要选择对应的服务实例，比如实现负载均衡等。

1.右键项目新建新的 `module`，同样选择 `Spring initializr`，选择需要的依赖，注册中心我们勾选 `Cloud Discovery` 中的 `EurekaServer`（当然也可以选择其他注册中心，如 `Zookeeper`、`Consul` 等）和 `Cloud Core` 中的 `Cloud Security`（需要验证才能注册到注册中心，否则泄漏注册中心地址后会有危险服务注册从而被攻击），当然也可以手动新建模块，具体读者可参考 `maven` 的文档 [https://maven.apache.org/guides/mini/guide-multiple-modules.html](https://maven.apache.org/guides/mini/guide-multiple-modules.html)。生成的 `pom.xml` 文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<parent>
		<groupId>cn.edu.nju</groupId>
		<artifactId>spring-cloud-example</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>

	<artifactId>eureka</artifactId>
	<packaging>jar</packaging>

	<name>eureka</name>
	<description>Spring Cloud Eureka</description>

	<dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>
	</dependencies>
</project>
```

2.在启动类上加上注解，如下：

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaApplication.class, args);
	}
}
```

3.配置 `Security`，新建类 `WebSecurityConfig` 继承 `WebSecurityConfigurerAdapter`，设置 `csrf` 关闭，否则服务不能正常注册：

```java
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        http.csrf().disable();
    }
}
```

4.配置文件 `application.yml`。

- `eureka.instance.prefer-ip-address` 设置为 `true` 使服务都是以 `ip` 地址形式提供的；
- `eureka.client.register-with-eureka` 表示是否将自己注册到服务中心，默认为 `true`，因为这里就是注册中心，所以设置为 `false`；
- `eureka.client.fetch-registry` 表示是否从其他注册中心同步信息，这里我们是单点的注册中心，所以不需要同步，设置为 `false`；
- `eureka.client.service-url.defaultZone` 设置与 `Eureka Server` 交互的地址，查询服务和注册服务都需要依赖这个地址；在开发过程中，我们常常希望 `Eureka Server` 能够迅速有效地踢出已关停的节点，但是由于 `Eureka` 自我保护模式，以及心跳周期长的原因，常常会遇到`Eureka Server`不踢出已关停的节点的问题，解决方法如下:
  `Eureka Server` 端：配置关闭自我保护，并按需配置 `Eureka Server` 清理无效节点的时间间隔。
  ```yml
  eureka.server.enable-self-preservation			# 设为false，关闭自我保护
  eureka.server.eviction-interval-timer-in-ms     # 清理间隔（单位毫秒，默认是60*1000）
  ```
  `Eureka Client` 端：配置开启健康检查，并按需配置续约更新时间和到期时间。
  ```yml
  eureka.client.healthcheck.enabled			# 开启健康检查（需要spring-boot-starter-actuator依赖）
  eureka.instance.lease-renewal-interval-in-seconds		# 续约更新时间间隔（默认30秒）
  eureka.instance.lease-expiration-duration-in-seconds 	# 续约到期时间（默认90秒）
  ```
- `spring.security.user` 配置鉴权有关信息，这里使用简单的用户名密码的形式鉴权，更复杂的安全机制可查看 `Spring Security` 的文档。配置用户名为 `user`，密码为 `123456`，则客户端的注册地址需要写成 `http://user:123456@localhost:8761/eureka/`。

示例：
服务端配置：

```yml
server:
  port: 8761
eureka:
  instance:
    appname: eureka-server
    prefer-ip-address: true
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${spring.cloud.client.ip-address}:${server.port}/eureka/
  server:
    eviction-interval-timer-in-ms: 5000
    enable-self-preservation: false
spring:
  application:
    name: eureka-server
  security:
    user:
      name: user
      password: 123456
info:
  app:
    name: Eureka Server
```

客户端配置：

```yml
spring:
  application:
    name: feign-server
  cloud:
    config:
      retry:
        initial-interval: 2000
        max-attempts: 3
      discovery:
        enabled: true
        service-id: config-server
      username: config
      password: 123456
  profiles:
    active: dev

eureka:
  client:
    healthcheck:
      enabled: true
    serviceUrl:
      defaultZone: http://user:123456@localhost:8761/eureka/
  instance:
    lease-expiration-duration-in-seconds: 30
    lease-renewal-interval-in-seconds: 10
```

> 注意：更改 Eureka 更新频率将打破服务器的自我保护功能，生产环境下不建议自定义这些配置。详见：[https://github.com/spring-cloud/spring-cloud-netflix/issues/373](https://github.com/spring-cloud/spring-cloud-netflix/issues/373)

## 配置中心

配置管理在分布式项目中有着非常重要的作用，在 `Spring Cloud` 应用的开发过程中，我们通常使用配置文件 `application.yml` 或 `bootstrap.yml` 来管理项目配置，对于单体应用来说，一次配置的修改并不会耗费多少的时间，但是对于许多个微服务都需要更新某项配置时，就需要更改每个服务的配置并重启服务完成更新。而有了配置中心统一管理项目配置后，则无需变动服务本身，只需更改配置中心对应配置即可，提高系统的可扩展性。`Spring Cloud Config` 提供了这样一种配置中心服务，并且能够通过消息总线实时更新已启动的服务的配置（除在项目启动时就已经读取并不能再更新的配置，如 `port`）。

`Spring Cloud Config` 可以使用 `Git` 仓库作为配置仓库，也可以使用本地的文件系统作为仓库（本质上是在本地创建了 `Git` 仓库并管理），提供服务端和客户端的支持。

1.右键项目新建新的 `module`，同样选择 `Spring initializr`，选择需要的依赖，配置服务端勾选 `Cloud Config` 中的 `Config Server`，配置客户端勾选 `Cloud Config` 中的 `Config Client`。

2.配置中心服务端配置。生成的依赖如下：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
</dependencies>
```

启动类添加 `@EnableConfigServer`：

```java
@SpringBootApplication
@EnableConfigServer
public class Application {
  ...
}
```

编写配置文件：

```yml
server.port=8888

spring
  application
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
          username: your git username
          password: your git password
          repos: #配置从 git 仓库获取的路径，这里为 https://github.com/spring-cloud-samples/config-repo/multi-repo-demo-*，*为通配符
            - patterns: multi-repo-demo-*
```

3.配置中心客户端配置。生成的依赖如下：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

编写配置文件（注意这里一定要为 `bootstrap.yml` 而不能是 `application.yml`，因为需要从引导上下文中读取配置中心的配置数据，作为应用上下文的配置，两者有先后关系）:

```yml
spring:
  application:
    name: config-client
  cloud:
    config:
      retry:
        initial-interval: 2000 #初始重试间隔时间
        max-attempts: 3 #重试次数
      discovery: #使用注册中心查找具体的配置中心地址
        enabled: true
        service-id: config-server
      username: config #如果配置中心服务端配置了安全限制，则需要提供用户名和密码
      password: 123456
```

如果需要配置重试，则需要添加依赖：

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

4.动态刷新配置。有时动态更新了 `Git` 仓库的配置，需要在不重启服务的情况下完成配置的动态更新，这需要引入另一个依赖 `actuator`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

假设客户端在 `Git` 仓库中的配置文件名为 `config-client-dev`（`config-client` 为服务名，`dev` 为活跃的 `profile`，这些在 `bootstrap.yml` 中配置，则可将在仓库中的对应的文件结合仓库中默认的 `application.yml` 作为客户端的配置文件 `appilcation.yml` 使用），为实现动态更新需要暴露 `refresh` 端点，这里简单起见开放所有端点，在 `bootstrap.yml` 或 `Git` 仓库中的 `config-client-dev.yml` 中配置：

```yml
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

在修改了远程仓库的配置文件（如 `application-client.yml`）后，需要使用 `POST` 请求访问服务的 `/actuator/refresh` 端点完成更新。

动态刷新配置对使用配置的方式有限制，客户端需通过 `@ConfigurationProperties`、`@Value("${…​}")` 或 `Environment` 使用配置中心的配置，这些配置在服务启动时已经加载，而为了强制刷新这些已读取的配置需要加上 `@RefreshScope` 注解，该注解可对类或者方法作用。

> 一般配置中心会配合消息总线一起使用，以便通知所有服务某项配置的更新。需要引入消息总线的依赖，具体如何使用可参考 [https://github.com/1016990109/spring-cloud-example](https://github.com/1016990109/spring-cloud-example) 中的 `config-server` 和 `feign-server` 的相关配置及代码，这里的例子基于 `RabbitMQ` 消息队列实现。

## 网关

服务网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供 `REST API` 的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制等功能。`Spring` 官方有推出自己的组件 `Spring Cloud Gateway`，相比于之前的 `Zuul(1.x)` 有许多优势：`Zuul` 基于 `servlet 2.5`（使用 3.x），使用阻塞 `API`。 它不支持任何长连接，如 `websockets`。而 `Gateway` 建立在 `Spring Framework 5`，`Project Reactor` 和 `Spring Boot 2` 之上，使用非阻塞 `API`。`Websockets` 得到支持，并且由于它与 `Spring` 紧密集成，所以将会是一个更好的开发体验。`Spring Cloud Gateway` 为微服务架构提供了前门保护的作用，同时将权限控制这些较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。

1.右键项目新建新的 `module`，同样选择 `Spring initializr`，选择需要的依赖，勾选 `Cloud Routing` 中的 `Gateway`。生成的依赖如下所示：

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

2.`application.yml` 配置示例：

```yml
server:
  port: 6606
spring:
  application:
    name: gateway
  cloud:
    gateway:
      filter:
        # 一些过滤器的配置
        remove-non-proxy-headers: #以下配置的 Header 不会转发到其他服务去
          headers: # 配置 RemoveNonProxyHeaders GatewayFilter Factory ?? 好像没有这个工厂
            - dummy
            - Connection
        secure-headers: # 配置 SecureHeaders GatewayFilter Factory，所有返回的响应都带上下面的头部
          content-security-policy: "default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src https:; style-src 'self' https: 'unsafe-inline'"
      routes:
        - id: apifeign
          # 重点！/info必须使用http进行转发，lb代表从注册中心获取服务
          uri: lb://feign_server
          predicates:
            # 重点！转发该路径！,/feignapi/**,也可以对 Method、Cookie、Header、Query、Host、Time、RemoteAddr 做匹配
            - Path=/feignapi/**
          filters: # 使用 name 配置 filter，自定义（内置不带）的 filter 可在 Config.java 中使用 @Bean 注解获取一个 RouteLocator
            - name: SecureHeaders
            - name: RequestRateLimiter #配置请求频率限制
              args:
                key-resolver: '#{@userKeyResolver}' # 一定需要 key-resolver 不然默认的 PrincipalNameKeyResolver 会失败
                redis-rate-limiter.replenishRate: 10 #允许用户每秒处理多少个请求
                redis-rate-limiter.burstCapacity: 20 #令牌桶的容量，允许在一秒钟内完成的最大请求数
            - StripPrefix=1 # http://localhost:6604/feignapi/instance/2, 必须加上StripPrefix=1，否则访问服务时会带上 feignapi 而不是我们期望的去掉 feign，只保留**部分
            - PrefixPath=/feign-server # 请求会转发到 /feign-server/** 上
```

上述的配置可以达到以下效果：凡是以 `/feignapi/` 开头的请求都会路由到注册中心的 `feign-server` 服务，并自动实现负载均衡，转发后的请求不再有 `/feignapi` 前缀但是会添加 `/feign-server` 作为前缀，且每个用户请求频率不得超过每秒 10 个，总请求数不得超过 20。转发请求时，不再携带头部 `dummy`、`Connection`，响应附带头部 `default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src https:; style-src 'self' https: 'unsafe-inline'`。

注意：实现网关限流需额外安装依赖以及配置 `key-resolver`，详情可移步作者 `Github` 仓库 [https://github.com/1016990109/spring-cloud-example](https://github.com/1016990109/spring-cloud-example)：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```java
@Configuration
public class Config {

    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress());
    }
}
```

> 除了根据请求路径路由外，`Spring Cloud Gateway` 还可以根据时间、`Cookie`、`Header`、`Host`、请求方式、请求参数、请求 `ip` 地址，或者将它们组合使用。具体使用方式读者可以自行查询 `Predicate` 的介绍。

## 服务提供者（基于 Fiegn 服务调用中的服务端）

服务的提供者，提供接口供调用者调用，是服务调用中的“服务端”。

1.服务端依赖。包括 `web`、`open-feign`、注册中心：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2.`application.yml` 配置：

```yml
feign:
  compression:
    request:
      enabled: true
    response:
      enabled: true
```

目的是为了启动服务间调用的压缩（`GZIP`），减少通信中的性能损耗。

3.启动类配置：需加上 `@EnableFeignClients` 注解

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class FeignServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(FeignServerApplication.class, args);
	}
}
```

4.提供接口。

```java
@RestController
@RequestMapping("/feign-server")
@RefreshScope
public class FeignServiceController {
    @Value("${feign-server.version}")
    String version;

    @RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.GET)
    public InstanceDTO getInstanceByServiceId(@PathVariable("serviceId") String serviceId) {
        return new InstanceDTO("version:" + version + ",serviceId:" + serviceId);
    }
}
```

以上即完成了基于 `Spring Cloud OpenFeign` 的服务间调用的服务端配置。

## 服务调用者（基于 Fiegn 服务调用中的客户端）

服务的调用者，是服务间调用的发起者。

1.客户端依赖。包括 `web`、注册中心、熔断器等等。

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

2.`application.yml` 配置：

```yml
feign:
  hystrix:
    enabled: true # feign 集成了熔断器，通过 feign.hystrix.enabled=true 启动
  compression:
    # 启用 gzip 压缩
    response:
      enabled: true
    request:
      enabled: true

management:
  endpoints:
    web:
      exposure:
        include: '*' # 默认只有 info 和 health， 允许 hystrix.stream 等
  endpoint:
    health:
      show-details: always # 在 health 的信息中显示详情，包括hystrix
```

上述出了配置 `Feign` 的相关配置、熔断外，还配置了 `hystrix` 监控即 `hystrix.stream`(访问 `http://localhost:port/actuator/hystrix.stream` 可查看相关统计数据，当然也可以配合 `turbine` 统一监控应用，这需要加入 `hystrix-dashboard` 依赖，又在上文提到，本项目采用单体应用的监控，可访问 `http://localhost:post/hystrix` 进入 `Hystrix DashBoard` 界面)。

3.启动类配置：

```java
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker
@EnableFeignClients
@EnableHystrixDashboard
public class FeignClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(FeignClientApplication.class, args);
    }
}
```

使用 `EnableCircuitBreaker` 配置 `Feign` 的熔断，使用 `EnableHystrixDashBoard` 启用熔断监控，使用 `@EnableFeignClients` 启动 `Open Fiegn`。

4.声明调用的服务接口：

```java
@FeignClient(value = "feign-server", fallback = FeignServiceClientFallBack.class)
public interface FeignServiceClient {
    /**
     * 调用 feign-server 的 feign-server/instance/{serviceId} 接口，获得返回值（类型 InstanceDTO）
     * @param serviceId
     * @return
     */
    @RequestMapping(value = "/feign-server/instance/{serviceId}", method = RequestMethod.GET)
    TestDTO getInstanceByServiceId(@PathVariable("serviceId") String serviceId);
}
```

`@FeignClient(value = "feign-server", fallback = FeignServiceClientFallBack.class)` 配置调用服务的名称（在注册中心的名称），以便像注册中心获取相应的服务实例地址（负载均衡的），接口主体部分保持和服务端相同即可，即 `Test getInstanceByServiceId(..)`。

而 `fallback` 则是断路器的配置，当服务（服务端）的错误率超过一定阀值时，`Hystrix` 可以触发断路机制，停止向该服务请求一段时间，而直接使用 `fallback` 指定的类作为接口实现，直接返回数据。

阀值有几个指标： 1.一定时间（默认 10s）内错误一定数量（默认 20 次）； 2.请求错误数量超过一定百分比（默认 50%）。

当某个服务的断路器打开后，`Hystrix` 将不会请求至该服务，直接 `fallback`，这样对于已经确定的故障在一定时间内不会再尝试。当断路器打开一段时间后，`Hystrix` 会进入"半开"状态，断路器会允许一个请求尝试对服务进行请求，如果该服务可以调用成功，则关闭断路器，否则将继续保持断路器打开，并进入倒计时，倒计时结束后继续尝试自动恢复。

5.开始服务调用。使用非常简单，像使用普通的本地接口一般使用即可：

```java
@Autowired
FeignServiceClient feignServiceClient;

@RequestMapping(value = "/instance/{serviceId}", method = RequestMethod.GET)
public TestDTO getInstanceByServiceId(@PathVariable("serviceId") String serviceId) {
    return feignServiceClient.getInstanceByServiceId(serviceId);
}
```

>注意：这里声明的接口返回数据类型的名称可以不相同，也可以只保留部分字段，如服务端提供的是 `InstanceDTO` 而客户端调用时声明的为 `TestDTO`，`Feign` 会自动地序列化与反序列化（`JSON`），不像 `Java` 自带的序列化工具那般要求类完全相同（序列号即成员变量等等都要求相同）。具体使用可参考 [https://github.com/1016990109/spring-cloud-example](https://github.com/1016990109/spring-cloud-example)。
