

[Springcloud源码阅读](https://juejin.im/post/5de070236fb9a071bc134ce0)

SpringCloud 是一系列微服务工具项目的**集合**。这个集合包含如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等工具。

这些项目地址在：[github.com/spring-clou…](https://github.com/spring-cloud) 包含：

- Spring Cloud Commons
- SpringCloud Netflix
- SpringCloud Stream
- SpringCloud Config .....

# Spring Cloud commons



## Spring Cloud Context

为项目提供：引导上下文，加密，刷新范围和环境端点等功能。SpringCloud在构建上下文 (即ApplicationContext实例)时，采用了Spring父子容器的设计，会在 SpringBoot构建的容器(后面称之为应用容器）之上创建一父容器 Bootstrap Application Context 。



## Spring Cloud commons

主要对微服务中的服务注册与发现、负载均衡、熔断器等功能提供一个**抽象层代码**，这个抽象层与具体的实现无关。这样这些功能具体的实现上可以采用不同的技术去实现，并可以做到在使用时灵活的更换。

列如：`DiscoveryClient`的使用，我们可以选择使用`Spring Cloud Netflix Eureka，Spring Cloud Consul发现和Spring Cloud Zookeeper发现`。

- 当我们使用`@EnableDiscoveryClient`来引入`DiscoveryClient`时，具体引入哪种`DiscoveryClient`，会根据你具体引入哪个项目有关。如果我们用了`Spring Cloud Netflix Eureka`。那就会引入`Spring Cloud Netflix Eureka`的`DiscoveryClient`。
- 当我们用用`@EnableEurekaClient`来引入`DiscoveryClient`，就是明确只能使用`Spring Cloud Netflix Eureka`的`DiscoveryClient`。

> 这也是`@EnableDiscoveryClient`与`@EnableEurekaClient`的区别。

由此我们能看出Spring Cloud Commons作为一个公共项目的目的。



# Spring Cloud Netflix

Netflix公司开源一套分布式服务框架，Netflix OSS。项目地址[github.com/Netflix](https://github.com/Netflix)

Spring Cloud Netflix 项目对其进行了二次封装，使其更加适应springcloud体系。 Spring Cloud Netflix下包含了与Netflix OSS对应 子项目

- SpringCloud-Eureka -- Netflix Eureka
- SpringCloud-Ribbon -- Netflix Ribbon
- SpringCloud-Hystrix --  Netflix Hystrix
- ...

当我们一想到springcloud ,就是Eureka，Ribbon。我认为这样是不对的，Eureka,Ribbon 只是作为Springcloud体系下Spring Cloud Netflix项目的子项目存在，他们只是微服务组件中的一种实现。

因为我们常用Spring Cloud Netflix 项目下的这些实现，客观的认为Eureka,Ribbon 就是springcloud。

因此在阅读源码前，理清这些关系`springcloud，Spring Cloud Netflix ，Netflix OSS`是非常有必要的。

