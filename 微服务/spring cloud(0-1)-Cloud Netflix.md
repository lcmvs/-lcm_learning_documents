# Spring Cloud Netflix

Spring Cloud Netflix 是 Spring Cloud 中的一套框架，由 Netflix 开发后来又并入 Spring Cloud 大家庭，它主要提供的模块包括：服务发现、断路器和监控、智能路由、客户端负载均衡等。

本文从 Spring Cloud 中的核心项目 Spring Cloud Netflix 入手，阐述了 Spring Cloud Netflix 的优势，介绍了 Spring Cloud Netflix 进行服务治理的技术原理。

Spring Cloud Netflix 的核心是用于服务注册与发现的 Eureka，接下来我们将以 Eureka 为线索，介绍 Eureka、Ribbon、Hystrix、Feign 这些 Spring Cloud Netflix 主要组件。

## 优势

对于微服务的治理而言，核心就是服务的注册和发现。所以选择哪个组件，很大程度上要看它对于服务注册与发现的解决方案。在这个领域，开源架构很多，最常见的是 Zookeeper，但这并不是一个最佳选择。

在分布式系统领域有个著名的 CAP 定理：**C—— 数据一致性，A—— 服务可用性，P—— 服务对网络分区故障的容错性**。这三个特性在任何分布式系统中不能同时满足，最多同时满足两个。

Zookeeper 是著名 Hadoop 的一个子项目，很多场景下 Zookeeper 也作为 Service 发现服务解决方案。**Zookeeper 保证的是 CP**，即任何时刻对 Zookeeper 的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性，但是它不能保证每次服务请求的可用性。从实际情况来分析，在使用 Zookeeper 获取服务列表时，如果 zookeeper 正在选主，或者 Zookeeper 集群中半数以上机器不可用，那么将就无法获得数据了。所以说，Zookeeper 不能保证服务可用性。

诚然，对于大多数分布式环境，尤其是涉及到数据存储的场景，数据一致性应该是首先被保证的，这也是 zookeeper 设计成 CP 的原因。但是对于服务发现场景来说，情况就不太一样了：针对同一个服务，即使注册中心的不同节点保存的服务提供者信息不尽相同，也并不会造成灾难性的后果。因为对于服务消费者来说，能消费才是最重要的 —— 拿到可能不正确的服务实例信息后尝试消费一下，也好过因为无法获取实例信息而不去消费。**所以，对于服务发现而言，可用性比数据一致性更加重要 ——AP 胜过 CP**。而 Spring Cloud Netflix 在设计 Eureka 时遵守的就是 AP 原则。

Eureka 本身是 Netflix 开源的一款提供服务注册和发现的产品，并且提供了相应的 Java 封装。在它的实现中，节点之间是相互平等的，部分注册中心的节点挂掉也不会对集群造成影响，即使集群只剩一个节点存活，也可以正常提供发现服务。哪怕是所有的服务注册节点都挂了，Eureka Clients 上也会缓存服务调用的信息。这就保证了我们微服务之间的互相调用是足够健壮的。

除此之外，Spring Cloud Netflix 背后强大的开源力量，也促使我们选择了 Spring Cloud Netflix：

- 前文提到过，Spring Cloud 的社区十分活跃，其在业界的应用也十分广泛（尤其是国外），而且整个框架也经受住了 Netflix 严酷生产环境的考验。
- 除了服务注册和发现，Spring Cloud Netflix 的其他功能也十分强大，包括 Ribbon，hystrix，Feign，Zuul 等组件，结合到一起，让服务的调用、路由也变得异常容易。
- Spring Cloud Netflix 作为 Spring 的重量级整合框架，使用它也意味着我们能从 Spring 获取到巨大的便利。Spring Cloud 的其他子项目，比如 Spring Cloud Stream、Spring Cloud Config 等等，都为微服务的各种需求提供了一站式的解决方案。

> Netflix 和 Spring Cloud 是什么关系呢？
> 我第一次 Netflix 这个单词时，是在美剧《纸牌屋》的片头。对，这是一家互联网流媒体播放商，可以这么说该网站上的美剧应该是最火的。由于是美国视频巨头，访问量非常的大，从而促使其技术快速的发展在背后支撑着，也正是如此，Netflix 开始把整体的系统往微服务上迁移。
> 他家的微服务做的不是最早的，但是却是最大规模的在生产级别微服务的尝试。几年前，Netflix 就把它的几乎整个微服务框架栈开源贡献给了社区，叫做 Netflix OSS。
> Spring 背后的 Pivotal 在 2015 年推出的 Spring Cloud 开源产品，主要对 Netflix 开源组件的进一步封装，方便 Spring 开发人员构建微服务基础框架。(虽然 Spring Cloud 到现在为止不只有 Netflix 提供的方案可以集成，还有很多方案，但 Netflix 是最成熟的。)

## 服务注册与发现Eureka

> Eureka 这个词来源于古希腊语，意为 “我找到了！我发现了！”，据传，阿基米德在洗澡时发现浮力原理，高兴得来不及穿上裤子，跑到街上大喊：“Eureka (我找到了)！”。

Spring Cloud Eureka 是 Spring Cloud Netflix 微服务套件的一部分，基于 Netflix Eureka 做了二次封装，主要负责完成微服务架构中的服务治理功能，服务治理可以说是微服务架构中最为核心和基础的模块，他主要用来实现各个微服务实例的自动化注册与发现。

- **服务注册**：在服务治理框架中，通常都会构建一个注册中心，每个服务单元向注册中心登记自己提供的服务，将主机与端口号、版本号、通信协议等一些附加信息告知注册中心，注册中心按照服务名分类组织服务清单，服务注册中心还需要以心跳的方式去监控清单中的服务是否可用，若不可用需要从服务清单中剔除，达到排除故障服务的效果。
- **服务发现**：由于在服务治理框架下运行，服务间的调用不再通过指定具体的实例地址来实现，而是通过向服务名发起请求调用实现。

Spring Cloud Eureka 使用 Netflix Eureka 来实现服务注册与发现，即包括了服务端组件，也包含了客户端组件，并且服务端和客户端均采用 Java 编写，所以 Eureka 主要适用与通过 Java 实现的分布式系统，或是与 JVM 兼容语言构建的系统，但是，由于 Eureka 服务端的服务治理机制提供了完备的 RESTful API，所以他也支持将非 Java 语言构建的微服务纳入 Eureka 的服务治理体系中来。

Eureka 由多个 instance (服务实例) 组成，这些服务实例可以分为两种：Eureka Server 和 Eureka Client。为了便于理解，我们将 Eureka client 再分为 Service Provider 和 Service Consumer。如下图所示：
[![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/006tNc79ly1fqc78zi0lpj30fa08hjrj.jpg)](https://src.windmt.com/img/006tNc79ly1fqc78zi0lpj30fa08hjrj.jpg)

- **Eureka Server**：服务的注册中心，负责维护注册的服务列表，同其他服务注册中心一样，支持高可用配置。
- **Service Provider**：服务提供方，作为一个 Eureka Client，向 Eureka Server 做服务注册、续约和下线等操作，注册的主要数据包括服务名、机器 ip、端口号、域名等等。
- **Service Consumer**：服务消费方，作为一个 Eureka Client，向 Eureka Server 获取 Service Provider 的注册信息，并通过远程调用与 Service Provider 进行通信。

> Service Provider 和 Service Consumer 不是严格的概念，Service Consumer 也可以随时向 Eureka Server 注册，来让自己变成一个 Service Provider。
>
> Spring Cloud 针对服务注册与发现，进行了一层抽象，并提供了三种实现：Eureka、Consul、Zookeeper。目前支持得最好的就是 Eureka，其次是 Consul，最后是 Zookeeper。在层抽象的作用下，我们可以无缝地切换服务治理实现，并且不影响任何其他的服务注册、服务发现、服务调用等逻辑。

### Eureka Server

Eureka Server 作为一个独立的部署单元，以 REST API 的形式为服务实例提供了注册、管理和查询等操作。同时，Eureka Server 也为我们提供了可视化的监控页面，可以直观地看到各个 Eureka Server 当前的运行状态和所有已注册服务的情况。

#### 高可用集群

Eureka Server 可以运行多个实例来构建集群，解决单点问题，但不同于 ZooKeeper 的选举 leader 的过程，Eureka Server 采用的是 Peer to Peer 对等通信。这是一种去中心化的架构，无 master/slave 区分，每一个 Peer 都是对等的。在这种架构中，节点通过彼此互相注册来提高可用性，每个节点需要添加一个或多个有效的 serviceUrl 指向其他节点。每个节点都可被视为其他节点的副本。

如果某台 Eureka Server 宕机，Eureka Client 的请求会自动切换到新的 Eureka Server 节点，当宕机的服务器重新恢复后，Eureka 会再次将其纳入到服务器集群管理之中。当节点开始接受客户端请求时，所有的操作都会进行 `replicateToPeer`（节点间复制）操作，将请求复制到其他 Eureka Server 当前所知的所有节点中。

一个新的 Eureka Server 节点启动后，会首先尝试从邻近节点获取所有实例注册表信息，完成初始化。Eureka Server 通过 `getEurekaServiceUrls()` 方法获取所有的节点，并且会通过心跳续约的方式定期更新。默认配置下，如果 Eureka Server 在一定时间内没有接收到某个服务实例的心跳，Eureka Server 将会注销该实例（默认为 90 秒，通过 `eureka.instance.lease-expiration-duration-in-seconds` 配置）。当 Eureka Server 节点在短时间内丢失过多的心跳时（比如发生了网络分区故障），那么这个节点就会进入自我保护模式。下图为 Eureka 官网的架构图

[![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/006tNc79ly1fqc7gr97yuj30k00f03zi.jpg)](https://src.windmt.com/img/006tNc79ly1fqc7gr97yuj30k00f03zi.jpg)

#### 自我保护模式

如果在 Eureka Server 的首页看到以下这段提示，则说明 Eureka 已经进入了保护模式。

> EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY’RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.

默认配置下，如果 Eureka Server 每分钟收到心跳续约的数量低于一个阈值，并且持续 15 分钟，就会触发自我保护。

```
阈值 = instance的数量 × (60 / instance的心跳间隔秒数) × 自我保护系数
```

在自我保护模式中，Eureka Server 会保护服务注册表中的信息，不再注销任何服务实例。当它收到的心跳数重新恢复到阈值以上时，该 Eureka Server 节点就会自动退出自我保护模式。它的设计哲学前面提到过，那就是宁可保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例。这样做会使客户端很容易拿到实际已经不存在的服务实例，会出现调用失败的情况。因此客户端要有容错机制，比如请求重试、断路器。该模式可以通过 `eureka.server.enable-self-preservation = false` 来禁用，~~同时 `eureka.instance.lease-renewal-interval-in-seconds` 可以用来更改心跳间隔（默认 30s），~~`eureka.server.renewal-percent-threshold` 可以用来修改自我保护系数（默认 0.85）。

**特别注意**
上边的那个阈值理论上是那么计算的，但是在实际中，并不是！！！并不是！！！并不是！！！
因为实际代码中是硬编码了，每个注册的实例都是**直接 + 2**，看下边 `eureka-core-1.8.7.jar!com.netflix.eureka.registry.AbstractInstanceRegistry#register` 中的这段代码就明白了

```
// The lease does not exist and hence it is a new registration
synchronized (lock) {
    if (this.expectedNumberOfRenewsPerMin > 0) {
        // Since the client wants to cancel it, reduce the threshold
        // (1
        // for 30 seconds, 2 for a minute)
        this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin + 2;
        this.numberOfRenewsPerMinThreshold =
                (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
    }
}
```

这个等有时间再仔细研究一下这个自我保护模式。

#### Region、Zone

Region、Zone、Eureka 集群三者的关系，如下图所示：
[![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/006tNc79ly1fqc7it32axj30by09m0sm.jpg)](https://src.windmt.com/img/006tNc79ly1fqc7it32axj30by09m0sm.jpg)

region 和 zone（或者 Availability Zone）均是亚马逊网络服务 (AWS) 的概念。在非 AWS 环境下，我们可以先简单地将 region 理解为 Eureka 集群，zone 理解成机房。上图就可以理解为一个 Eureka 集群被部署在了 zone1 机房和 zone2 机房中。

> 为什么对 AWS 有如此完善的支持？
> 因为 Netflix 在服务器运维上依托 AWS 云，而上边说到 Spring Cloud Netflix 是 Spring 在 Netflix 的开源组件 Netflix OSS 上封装的。

### Service Provider

#### 服务注册

Service Provider 本质上是一个 Eureka Client。它启动时，会调用服务注册方法，向 Eureka Server 注册自己的信息。Eureka Server 会维护一个已注册服务的列表，这个列表为一个嵌套的 HashMap：

```
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
        = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
```

- 第一层，application name 和对应的服务实例。
- 第二层，服务实例及其对应的注册信息，包括 IP，端口号等。

当实例状态发生变化时（如自身检测认为 Down 的时候），也会向 Eureka Server 更新自己的服务状态，同时用 `replicateToPeers()` 向其它 Eureka Server 节点做状态同步。

[![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/006tNc79ly1fqc7qck6kdj30kw0c8glu.jpg)](https://src.windmt.com/img/006tNc79ly1fqc7qck6kdj30kw0c8glu.jpg)

#### 续约与剔除

前面提到过，服务实例启动后，会周期性地向 Eureka Server 发送心跳以续约自己的信息，避免自己的注册信息被剔除。续约的方式与服务注册基本一致：首先更新自身状态，再同步到其它 Peer。

如果 Eureka Server 在一段时间内没有接收到某个微服务节点的心跳，Eureka Server 将会注销该微服务节点（自我保护模式除外）。

[![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/006tNc79ly1fqc7qonmikj30kv0cxglq.jpg)](https://src.windmt.com/img/006tNc79ly1fqc7qonmikj30kv0cxglq.jpg)

### Service Consumer

Service Consumer 本质上也是一个 Eureka Client（它也会向 Eureka Server 注册，只是这个注册信息无关紧要罢了）。它启动后，会从 Eureka Server 上获取所有实例的注册信息，包括 IP 地址、端口等，并缓存到本地。这些信息默认每 30 秒更新一次。前文提到过，如果与 Eureka Server 通信中断，Service Consumer 仍然可以通过本地缓存与 Service Provider 通信。

实际开发 Eureka 的过程中，有时会遇见 Service Consumer 获取到 Server Provider 的信息有延迟，在 [Eureka Wiki](https://github.com/Netflix/eureka/wiki/Understanding-eureka-client-server-communication) 中有这么一段话:

> All operations from Eureka client may take some time to reflect in the Eureka servers and subsequently in other Eureka clients. This is because of the caching of the payload on the eureka server which is refreshed periodically to reflect new information. Eureka clients also fetch deltas periodically. Hence, it may take up to 2 mins for changes to propagate to all Eureka clients.

最后一句话提到，服务端的更改可能需要 2 分钟才能传播到所有客户端，至于原因并没有介绍。这是因为 Eureka 有三处缓存和一处延迟造成的。

- Eureka Server 对注册列表进行缓存，默认时间为 30s。
- Eureka Client 对获取到的注册信息进行缓存，默认时间为 30s。
- Ribbon 会从上面提到的 Eureka Client 获取服务列表，将负载均衡后的结果缓存 30s。
- 如果不是在 Spring Cloud 环境下使用这些组件 (Eureka, Ribbon)，服务启动后并不会马上向 Eureka 注册，而是需要等到第一次发送心跳请求时才会注册。心跳请求的发送间隔默认是 30s。Spring Cloud 对此做了修改，服务启动后会马上注册。

基于 Service Consumer 获取到的服务实例信息，我们就可以进行服务调用了。而 Spring Cloud 也为 Service Consumer 提供了丰富的服务调用工具：

- Ribbon，实现客户端的负载均衡。
- Hystrix，断路器。
- Feign，RESTful Web Service 客户端，整合了 Ribbon 和 Hystrix。

接下来我们就一一介绍。

## 负载均衡 ——Ribbon

Ribbon 是 Netflix 发布的开源项目，主要功能是为 REST 客户端实现负载均衡。它主要包括六个组件：

- ServerList，负载均衡使用的服务器列表。这个列表会缓存在负载均衡器中，并定期更新。当 Ribbon 与 Eureka 结合使用时，ServerList 的实现类就是 DiscoveryEnabledNIWSServerList，它会保存 Eureka Server 中注册的服务实例表。

- ServerListFilter，服务器列表过滤器。这是一个接口，主要用于对 Service Consumer 获取到的服务器列表进行预过滤，过滤的结果也是 ServerList。Ribbon 提供了多种过滤器的实现。

- IPing，探测服务实例是否存活的策略。

- IRule，负载均衡策略，其实现类表述的策略包括：轮询、随机、根据响应时间加权等，其类结构如下图所示。

  ![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/006tNc79ly1fqc7ssqkaxj30mj0alq3y.jpg)

  > 我们也可以自己定义负载均衡策略，比如我们就利用自己实现的策略，实现了服务的版本控制和直连配置。实现好之后，将实现类重新注入到 Ribbon 中即可。

- ILoadBalancer，负载均衡器。这也是一个接口，Ribbon 为其提供了多个实现，比如 ZoneAwareLoadBalancer。而上层代码通过调用其 API 进行服务调用的负载均衡选择。一般 ILoadBalancer 的实现类中会引用一个 IRule。

- RestClient，服务调用器。顾名思义，这就是负载均衡后，Ribbon 向 Service Provider 发起 REST 请求的工具。

Ribbon 工作时会做四件事情：

1. 优先选择在同一个 Zone 且负载较少的 Eureka Server；
2. 定期从 Eureka 更新并过滤服务实例列表；
3. 根据用户指定的策略，在从 Server 取到的服务注册列表中选择一个实例的地址；
4. 通过 RestClient 进行服务调用。

## 熔断 ——Hystrix

## 雪崩效应

在微服务架构中通常会有多个服务层调用，基础服务的故障可能会导致级联故障，进而造成整个系统不可用的情况，这种现象被称为服务雪崩效应。服务雪崩效应是一种因 “服务提供者” 的不可用导致 “服务消费者” 的不可用，并将不可用逐渐放大的过程。
如果下图所示：A 作为服务提供者，B 为 A 的服务消费者，C 和 D 是 B 的服务消费者。A 不可用引起了 B 的不可用，并将不可用像滚雪球一样放大到 C 和 D 时，雪崩效应就形成了。
[![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/006tNc79ly1fqdrhznd6yj30ak0cyaep.jpg)](https://src.windmt.com/img/006tNc79ly1fqdrhznd6yj30ak0cyaep.jpg)

### 断路器

Netflix 创建了一个名为 Hystrix 的库，实现了断路器的模式。“断路器” 本身是一种开关装置，当某个服务单元发生故障之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个符合预期的、可处理的备选响应（FallBack），而不是长时间的等待或者抛出调用方无法处理的异常，这样就保证了服务调用方的线程不会被长时间、不必要地占用，从而避免了故障在分布式系统中的蔓延，乃至雪崩。

[![img](https://src.windmt.com/img/006tNc79ly1fqc7u3f954j30fs09vgln.jpg)](https://src.windmt.com/img/006tNc79ly1fqc7u3f954j30fs09vgln.jpg)

当然，在请求失败频率较低的情况下，Hystrix 还是会直接把故障返回给客户端。只有当失败次数达到阈值时，断路器打开并且不进行后续通信，而是直接返回备选响应。当然，Hystrix 的备选响应也是可以由开发者定制的。

[![img](https://src.windmt.com/img/006tNc79ly1fqc7ubeb0uj30kz0drdg8.jpg)](https://src.windmt.com/img/006tNc79ly1fqc7ubeb0uj30kz0drdg8.jpg)

### 监控

除了隔离依赖服务的调用以外，Hystrix 还提供了准实时的调用监控（**Hystrix Dashboard**），Hystrix 会持续地记录所有通过 Hystrix 发起的请求的执行信息，并以统计报表和图形的形式展示给用户，包括每秒执行多少请求多少成功，多少失败等。Netflix 通过 hystrix-metrics-event-stream 项目实现了对以上指标的监控。Spring Cloud 也提供了 Hystrix Dashboard 的整合，对监控内容转化成可视化界面，[Hystrix Dashboard Wiki](https://github.com/Netflix/Hystrix/wiki/Dashboard) 上详细说明了图上每个指标的含义。

[![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%BE%AE%E6%9C%8D%E5%8A%A1/assets/006tNc79ly1fqc7ukr4mmj30hs0bf0v6.jpg)](https://src.windmt.com/img/006tNc79ly1fqc7ukr4mmj30hs0bf0v6.jpg)

## 远程调用 ——Feign

Feign 是一个声明式的 Web Service 客户端，它的目的就是让 Web Service 调用更加简单。它整合了 Ribbon 和 Hystrix，从而让我们不再需要显式地使用这两个组件。Feign 还提供了 HTTP 请求的模板，通过编写简单的接口和插入注解，我们就可以定义好 HTTP 请求的参数、格式、地址等信息。接下来，Feign 会完全代理 HTTP 的请求，我们只需要像调用方法一样调用它就可以完成服务请求。

Feign 具有如下特性：

- 可插拔的注解支持，包括 Feign 注解和 JAX-RS 注解
- 支持可插拔的 HTTP 编码器和解码器
- 支持 Hystrix 和它的 Fallback
- 支持 Ribbon 的负载均衡
- 支持 HTTP 请求和响应的压缩