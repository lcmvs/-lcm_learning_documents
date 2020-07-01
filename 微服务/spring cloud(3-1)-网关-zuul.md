通过之前几篇 Spring Cloud 中几个核心组件的介绍，我们已经可以构建一个简略的微服务架构了，可能像下图这样：

![img](https://src.windmt.com/img/006tNc79ly1fqmei2skktj30ma0k2myr.jpg)

我们使用 Spring Cloud Netflix 中的 Eureka 实现了服务注册中心以及服务注册与发现；而服务间通过 Ribbon 或 Feign 实现服务的消费以及均衡负载；通过 Spring Cloud Config 实现了应用多环境的外部化配置以及版本管理。为了使得服务集群更为健壮，使用 Hystrix 的融断机制来避免在微服务架构中个别服务出现异常时引起的故障蔓延。似乎一个微服务框架已经完成了。

我们还是少考虑了一个问题，外部的应用如何来访问内部各种各样的微服务呢？在微服务架构中，后端服务往往不直接开放给调用端，而是通过一个 API 网关根据请求的 URL，路由到相应的服务。当添加 API 网关后，在第三方调用端和服务提供方之间就创建了一面墙，这面墙直接与调用方通信进行权限控制，后将请求均衡分发给后台服务端。



# 为什么需要 API Gateway

**1、简化客户端调用复杂度**
在微服务架构模式下后端服务的实例数一般是动态的，对于客户端而言很难发现动态改变的服务实例的访问地址信息。因此在基于微服务的项目中为了简化前端的调用逻辑，通常会引入 API Gateway 作为轻量级网关，同时 API Gateway 中也会实现相关的认证逻辑从而简化内部服务之间相互调用的复杂度。

![img](https://src.windmt.com/img/006tNc79ly1fqmintl4smj30pi0e6mxw.jpg)

**2、数据裁剪以及聚合**
通常而言不同的客户端对于显示时对于数据的需求是不一致的，比如手机端或者 Web 端又或者在低延迟的网络环境或者高延迟的网络环境。

因此为了优化客户端的使用体验，API Gateway 可以对通用性的响应数据进行裁剪以适应不同客户端的使用需求。同时还可以将多个 API 调用逻辑进行聚合，从而减少客户端的请求数，优化客户端用户体验

**3、多渠道支持**
当然我们还可以针对不同的渠道和客户端提供不同的 API Gateway, 对于该模式的使用由另外一个大家熟知的方式叫 Backend for front-end, 在 Backend for front-end 模式当中，我们可以针对不同的客户端分别创建其 BFF，进一步了解 BFF 可以参考这篇文章：[Pattern: Backends For Frontends](http://samnewman.io/patterns/architectural/bff/)

[![img](https://src.windmt.com/img/006tNc79ly1fqmdulzjxuj30ke0dc0tp.jpg)](https://src.windmt.com/img/006tNc79ly1fqmdulzjxuj30ke0dc0tp.jpg)

**4、遗留系统的微服务化改造**
对于系统而言进行微服务改造通常是由于原有的系统存在或多或少的问题，比如技术债务，代码质量，可维护性，可扩展性等等。API Gateway 的模式同样适用于这一类遗留系统的改造，通过微服务化的改造逐步实现对原有系统中的问题的修复，从而提升对于原有业务响应力的提升。通过引入抽象层，逐步使用新的实现替换旧的实现。

[![img](https://src.windmt.com/img/006tNc79ly1fqmduv84imj30v20hejta.jpg)](https://src.windmt.com/img/006tNc79ly1fqmduv84imj30v20hejta.jpg)

在 Spring Cloud 体系中， Spring Cloud Zuul 就是提供负载均衡、反向代理、权限认证的一个 API Gateway。

> 我们这篇说的 Zuul 是 Zuul 1，实际上 Netflix 已经发布了 Zuul 2，不过 Spring 好像并没有将 Zuul 2 整合到 Spring Cloud 生态中的意思，因为它自己做了一个 Spring Cloud Gateway（估计是因为之前 Zuul 2 一直跳票导致等不及了吧）。关于 Spring Cloud Gateway 这个高性能的网关我们以后再说。