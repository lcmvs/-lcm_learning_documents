# RPC

[从零开始搭建公司后台技术栈](https://www.kancloud.cn/dargon/stack_development/1134453)

远程过程调用（Remote Procedure Call，RPC）是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员无需额外地为这个交互作用编程。

通俗来讲，一个完整的RPC调用过程，就是 Server 端实现了一个函数，客户端使用 RPC 框架提供的接口，调用这个函数的实现，并获取返回值的过程。

业界 RPC 框架大致分为两大流派，一种侧重跨语言调用，另一种是偏重服务治理。

跨语言调用型的 RPC 框架有 Thrift、gRPC、Hessian、Hprose 等。这类 RPC 框架侧重于服务的跨语言调用，能够支持大部分的语言进行语言无关的调用，非常适合多语言调用场景。但这类框架没有服务发现相关机制，实际使用时需要代理层进行请求转发和负载均衡策略控制。

其中，gRPC 是 Google 开发的高性能、通用的开源 RPC 框架，其由 Google 主要面向移动应用开发并基于 HTTP/2 协议标准而设计，基于 ProtoBuf(Protocol Buffers) 序列化协议开发，且支持众多开发语言。本身它不是分布式的，所以要实现框架的功能需要进一步的开发。

Hprose(High Performance Remote Object Service Engine) 是一个 MIT 开源许可的新型轻量级跨语言跨平台的面向对象的高性能远程动态通讯中间件。

服务治理型的 RPC 框架的特点是功能丰富，提供高性能的远程调用、服务发现及服务治理能力，适用于大型服务的服务解耦及服务治理，对于特定语言(Java)的项目可以实现透明化接入。缺点是语言耦合度较高，跨语言支持难度较大。国内常见的冶理型 RPC 框架如下：

- Dubbo: Dubbo 是阿里巴巴公司开源的一个 Java 高性能优秀的服务框架，使得应用可通过高性能的 RPC 实现服务的输出和输入功能，可以和 Spring 框架无缝集成。当年在淘宝内部，Dubbo 由于跟淘宝另一个类似的框架 HSF 有竞争关系，导致 Dubbo 团队解散，最近又活过来了，有专职同学投入。
- DubboX: DubboX 是由当当在基于 Dubbo 框架扩展的一个 RPC 框架，支持 REST 风格的远程调用、Kryo/FST 序列化，增加了一些新的feature。
- Motan: Motan 是新浪微博开源的一个 Java 框架。它诞生的比较晚，起于 2013 年，2016 年 5 月开源。Motan 在微博平台中已经广泛应用，每天为数百个服务完成近千亿次的调用。
- rpcx: rpcx 是一个类似阿里巴巴[Dubbo](http://dubbo.io/) 和微博 [Motan](https://github.com/weibocom/motan) 的分布式的 RPC 服务框架，基于 Golang net/rpc 实现。但是 rpcx 基本只有一个人在维护，没有完善的社区，使用前要慎重，之前做 Golang 的 RPC 选型时也有考虑这个，最终还是放弃了，选择了 gRPC，如果想自己自研一个 RPC 框架，可以参考学习一下。