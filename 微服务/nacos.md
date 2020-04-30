# nacos学习

[阿里巴巴为什么不用 ZooKeeper 做服务发现？](https://yq.aliyun.com/articles/601745)

在粗粒度分布式锁，分布式选主，主备高可用切换等不需要高TPS 支持的场景下有不可替代的作用，而这些需求往往多集中在大数据、离线任务等相关的业务领域，因为大数据领域，讲究分割数据集，并且大部分时间分任务多进程/线程并行处理这些数据集，但是总是有一些点上需要将这些任务和进程统一协调，这时候就是 ZooKeeper 发挥巨大作用的用武之地。

但是在交易场景交易链路上，在主业务数据存取，大规模服务发现、大规模健康监测等方面有天然的短板，应该竭力避免在这些场景下引入 ZooKeeper，在阿里巴巴的生产实践中，应用对ZooKeeper申请使用的时候要进行严格的场景、容量、SLA需求的评估。

zookeeper的写不可扩展。

所以可以使用 ZooKeeper，但是大数据请向左，而交易则向右，分布式协调向左，服务发现向右。

[nacos重要文章](https://www.liaochuntao.cn/categories/nacos/)

[支持Dubbo生态发展，阿里巴巴启动新的开源项目 Nacos](https://yq.aliyun.com/articles/604028)

[Nacos](https://github.com/alibaba/Nacos) 是阿里巴巴开源的一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

关于 Nacos Spring Cloud 的详细文档请参看：[Nacos Config](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/wiki/Nacos-config) 和 [Nacos Discovery](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/wiki/Nacos-discovery)。

- 通过 Nacos Server 和 spring-cloud-starter-alibaba-nacos-config 实现配置的动态变更。
- 通过 Nacos Server 和 spring-cloud-starter-alibaba-nacos-discovery 实现服务的注册与发现。

## 安装

1.直接下载：[Nacos Server 下载页](https://github.com/alibaba/nacos/releases)

2.启动 Server，进入解压后文件夹或编译打包好的文件夹，找到如下相对文件夹 nacos/bin，并对照操作系统实际情况之下如下命令。

1. Linux/Unix/Mac 操作系统，执行命令 `sh startup.sh -m standalone`
2. Windows 操作系统，执行命令 `cmd startup.cmd`

## 注册中心使用

### 1.引入依赖

```xml
 <dependency>
     <groupId>com.alibaba.cloud</groupId>
     <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
 </dependency>
```

### 2.配置nacos server地址

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 119.29.246.230:8848
        namespace: 423905ea-6688-434e-a178-952ae4517b71
```

### 3.开启服务注册与发现功能

```java
 @SpringBootApplication
 @EnableDiscoveryClient
 public class ProviderApplication {

 	public static void main(String[] args) {
 		SpringApplication.run(Application.class, args);
 	}
 }
```

### 4.服务发现

#### 4.1使用RestTemplate

```java
@Configuration
public class ServerConfig {

    /**
     * 在定义RestTemplate的时候，增加了@LoadBalanced注解，而在真正调用服务接口的时候，
     * 原来host部分是通过手工拼接ip和端口的，直接采用服务名的时候来写请求路径即可。
     * 在真正调用的时候，Spring Cloud会将请求拦截下来，然后通过负载均衡器选出节点，
     * 并替换服务名部分为具体的ip和端口，从而实现基于服务名的负载均衡调用。
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
}
```

```java
@Slf4j
@RestController
public class RestTemplateCotroller {

    @Autowired
    RestTemplate restTemplate;

    @GetMapping("restTemplate")
    public User restTemplate(){
        return restTemplate.getForObject("http://alibaba-nacos-discovery-server/hello?name=gaga&id=1",User.class);
    }

}
```

#### 4.2使用feign

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <version>2.1.2.RELEASE</version>
</dependency>
```

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class AlibabaNacosDiscoveryClientCommonApplication {

    public static void main(String[] args) {
        SpringApplication.run(AlibabaNacosDiscoveryClientCommonApplication.class, args);
    }

}
```

```java
@FeignClient("alibaba-nacos-discovery-server")
public interface AlibabaNacosDiscoveryServerService {

    @GetMapping("/hello")
    User hello(@SpringQueryMap User user);

}
```

```java
@RestController
public class FeignController {

    @Autowired
    AlibabaNacosDiscoveryServerService alibabaNacosDiscoveryServerService;

    @GetMapping("feign")
    public User feign(){
        User user=new User();
        user.setId(123L);
        user.setName("hehe");
        return alibabaNacosDiscoveryServerService.hello(user);
    }

}
```

#### 4.3负载均衡ribbon

[Spring Cloud源码分析（二）Ribbon](http://blog.didispace.com/springcloud-sourcecode-ribbon/)

[Spring Cloud Alibaba之负载均衡组件 - Ribbon详解（三）](https://www.jianshu.com/p/eb4d4ce0b751)

| 规则名称                  | 特点                                                         |
| ------------------------- | ------------------------------------------------------------ |
| AvailabilityFilteringRule | 过滤掉一直连接失败的被标记为circuit tripped（电路跳闸）的后端Service，并过滤掉那些高并发的后端Server或者使用一个AvailabilityPredicate来包含过滤Server的逻辑，其实就是检查status的记录的各个Server的运行状态 |
| BestAvailableRule         | 选择一个最小的并发请求的Server，逐个考察Server，如果Server被tripped了，则跳过 |
| RandomRule                | 随机选择一个Server                                           |
| ResponseTimeWeightedRule  | 已废弃，作用同WeightedResponseTimeRule                       |
| RetryRule                 | 对选定的负责均衡策略机上充值机制，在一个配置时间段内当选择Server不成功，则一直尝试使用subRule的方式选择一个可用的Server |
| RoundRobinRule            | 轮询选择，轮询index，选择index对应位置Server                 |
| WeightedResponseTimeRule  | 根据相应时间加权，相应时间越长，权重越小，被选中的可能性越低 |
| ZoneAvoidanceRule         | `（默认是这个）`负责判断Server所Zone的性能和Server的可用性选择Server，在没有Zone的环境下，类似于轮询（RoundRobinRule） |

##### 细粒度配置

```java
@Configuration
@RibbonClient(name ="服务名", configuration = GoodsRibbonRuleConfig.class)
public class GoodsRibbonConfig {
}

@Configuration
public class GoodsRibbonRuleConfig {
    @Bean
    public IRule ribbonRulr() {
        return new RandomRule();
    }
}
```



```yaml
server-1: # 服务名称 Service-ID
  ribbon:
    #  配置文件配置负载均衡算法-我这里使用的是自定义的Ribbon的负载均衡算法，默认
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule 
```



##### 全局配置

```java
@Configuration
@RibbonClients(defaultConfiguration = GoodsRibbonRuleConfig.class)
public class RibbonConfig {
}
```





##### nacos权重负载均衡

因为ribbon没有支持nacos权重的负载均衡规则，所有可以自定义一个负载均衡规则，使用nacos的自己的查找实例的方式，而feign也是使用的ribbon。

[扩展Ribbon支持Nacos权重的三种方式](https://blog.csdn.net/lilizhou2008/article/details/94369069)

 [SpringCloud-Ribbon自定义负载均衡算法](https://blog.csdn.net/www1056481167/article/details/81151064)

```java
@Slf4j
public class NacosWeightRandomV2Rule extends AbstractLoadBalancerRule {
    @Autowired
    private NacosDiscoveryProperties discoveryProperties;

    @Override
    public Server choose(Object key) {
        DynamicServerListLoadBalancer loadBalancer = (DynamicServerListLoadBalancer) getLoadBalancer();
        String name = loadBalancer.getName();
        try {
            Instance instance = discoveryProperties.namingServiceInstance()
                    .selectOneHealthyInstance(name);

            log.info("选中的instance = {}", instance);

            /*
             * instance转server的逻辑参考自：
             * org.springframework.cloud.alibaba.nacos.ribbon.NacosServerList.instancesToServerList
             */
            return new NacosServer(instance);
        } catch (NacosException e) {
            log.error("发生异常", e);
            return null;
        }
    }

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {
    }
}
```



```java
@Configuration
public class ServerConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    public IRule myRule() {
        return new NacosWeightRandomV2Rule();
    }

}
```

## 配置中心使用

![数据模型](assets/20190623205127975.jpg)

### 1.引入依赖

```xml
 <dependency>
     <groupId>com.alibaba.cloud</groupId>
     <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
 </dependency>
```

### 2.属性配置

bootstrap 配置文件中配置 Nacos Config 元数据

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 119.29.246.230:8848
        file-extension: yaml
        namespace: 423905ea-6688-434e-a178-952ae4517b71
```





## 源码分析

### 配置管理naming

[nacos源码分析(二)--------客户端获取配置中心的实现](https://juejin.im/post/5db79507f265da4d34299cc4)

#### example

```java
public class ConfigExample {

    public static void main(String[] args) throws NacosException, InterruptedException {
        String serverAddr = "localhost:8848";
        String namespace="8f8bc91f-c55d-400d-b4ad-b80e637c5d96";
        String dataId="test-lcm";
        String group="DEFAULT_GROUP";
        Properties properties = new Properties();
        properties.put("serverAddr", serverAddr);
        properties.put("namespace", namespace);
        //根据属性构造一个ConfigService
        ConfigService configService = NacosFactory.createConfigService(properties);
        //更加group和dataId获取一个配置
        String content = configService.getConfig(dataId, group, 5000);
        System.out.println(content);
        //增加一个listener
        configService.addListener(dataId, group, new Listener() {
            @Override
            public void receiveConfigInfo(String configInfo) {
                System.out.println("receive:" + configInfo);
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        });

        //推送一个配置
        boolean isPublishOk = configService.publishConfig(dataId, group, "content");
        System.out.println(isPublishOk);

        Thread.sleep(3000);
        //获取一个配置
        content = configService.getConfig(dataId, group, 5000);
        System.out.println(content);

        //删除一个配置
        boolean isRemoveOk = configService.removeConfig(dataId, group);
        System.out.println(isRemoveOk);
        Thread.sleep(3000);

        content = configService.getConfig(dataId, group, 5000);
        System.out.println(content);
        Thread.sleep(300000);

    }
}
```

```java
//ConfigFactory.java
public static ConfigService createConfigService(Properties properties) throws NacosException {
    try {
        Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
        //使用一个Properties参数的构造方法，使用反射构造一个ConfigService
        Constructor constructor = driverImplClass.getConstructor(Properties.class);
        ConfigService vendorImpl = (ConfigService)constructor.newInstance(properties);
        return vendorImpl;
    } catch (Throwable var4) {
        throw new NacosException(-400, var4);
    }
}
```

```java
//NacosConfigService.java
public NacosConfigService(Properties properties) throws NacosException {
    String encodeTmp = properties.getProperty("encode");
    if (StringUtils.isBlank(encodeTmp)) {
        this.encode = "UTF-8";
    } else {
        this.encode = encodeTmp.trim();
    }

    this.initNamespace(properties);
    //一个http客户端工具，HttpURLConnection
    this.agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
    //通过GetServerListTask后服务中心去拉取配置中心的地址列表
    this.agent.start();
    this.worker = new ClientWorker(this.agent, this.configFilterChainManager, properties);
}
```

#### 拉取配置中心地址

```java
public synchronized void start() throws NacosException {

    if (isStarted || isFixed) {
        return;
    }

    GetServerListTask getServersTask = new GetServerListTask(addressServerUrl);
    for (int i = 0; i < initServerlistRetryTimes && serverUrls.isEmpty(); ++i) {
        getServersTask.run();
        try {
            this.wait((i + 1) * 100L);
        } catch (Exception e) {
            LOGGER.warn("get serverlist fail,url: {}", addressServerUrl);
        }
    }

    if (serverUrls.isEmpty()) {
        LOGGER.error("[init-serverlist] fail to get NACOS-server serverlist! env: {}, url: {}", name,
            addressServerUrl);
        throw new NacosException(NacosException.SERVER_ERROR,
            "fail to get NACOS-server serverlist! env:" + name + ", not connnect url:" + addressServerUrl);
    }
    //定时任务每隔30s进行获取配置中心地址列表
    TimerService.scheduleWithFixedDelay(getServersTask, 0L, 30L, TimeUnit.SECONDS);
    isStarted = true;
}
```



#### 拉取配置

```java
//ClientWorker.java
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager, final Properties properties) {
    this.agent = agent;
    this.configFilterChainManager = configFilterChainManager;

    // Initialize the timeout parameter

    init(properties);

    //单线程定时任务
    executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
            t.setDaemon(true);
            return t;
        }
    });

    //多线程长轮询，核心线程数为cpu逻辑处理器数量
    //去配置中心拉取配置
    executorService = Executors.newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
            t.setDaemon(true);
            return t;
        }
    });
    
    //固定延时重复执行任务，每10ms检查一次checkConfigInfo()
    //第一次为initialDelay后，第二次为第一次任务执行结束后再加上delay
    executor.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            try {
                checkConfigInfo();
            } catch (Throwable e) {
                LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
            }
        }
    }, 1L, 10L, TimeUnit.MILLISECONDS);
}

//任务分批，默认每个线程最多处理3000个listener，可配置
public void checkConfigInfo() {
    // 分任务
    int listenerSize = cacheMap.get().size();
    // 向上取整为批数
    int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize());
    if (longingTaskCount > currentLongingTaskCount) {
        for (int i = (int) currentLongingTaskCount; i < longingTaskCount; i++) {
            // 要判断任务是否在执行 这块需要好好想想。 任务列表现在是无序的。变化过程可能有问题
            executorService.execute(new LongPollingRunnable(i));
        }
        currentLongingTaskCount = longingTaskCount;
    }
}


```



```JAVA
class LongPollingRunnable implements Runnable {
    private int taskId;

    public LongPollingRunnable(int taskId) {
        this.taskId = taskId;
    }

    @Override
    public void run() {

        List<CacheData> cacheDatas = new ArrayList<CacheData>();
        List<String> inInitializingCacheList = new ArrayList<String>();
        try {
            // check failover config
            //本地检查主要是做一个故障容错，当服务端挂掉后，Nacos 客户端可以从本地的文件系统中获取相关的配置信息
            for (CacheData cacheData : cacheMap.get().values()) {
                if (cacheData.getTaskId() == taskId) {
                    cacheDatas.add(cacheData);
                    try {
                        //检查本地缓存信息是否变更
                        checkLocalConfig(cacheData);
                        //如果使用本地缓存，需要检查md5
                        if (cacheData.isUseLocalConfigInfo()) {
                            cacheData.checkListenerMd5();
                        }
                    } catch (Exception e) {
                        LOGGER.error("get local config info error", e);
                    }
                }
            }

            // check server config
            //从远程获取值变化了的DataID列表。返回的对象里只有dataId和group是有效的。 保证不返回NULL。
            List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);

            //遍历值变化了的DataID列表，从远端获取新值，修改CacheData
            for (String groupKey : changedGroupKeys) {
                String[] key = GroupKey.parseKey(groupKey);
                String dataId = key[0];
                String group = key[1];
                String tenant = null;
                if (key.length == 3) {
                    tenant = key[2];
                }
                try {
                    String content = getServerConfig(dataId, group, tenant, 3000L);
                    CacheData cache = cacheMap.get().get(GroupKey.getKeyTenant(dataId, group, tenant));
                    cache.setContent(content);
                    LOGGER.info("[{}] [data-received] dataId={}, group={}, tenant={}, md5={}, content={}",
                        agent.getName(), dataId, group, tenant, cache.getMd5(),
                        ContentUtils.truncateContent(content));
                } catch (NacosException ioe) {
                    String message = String.format(
                        "[%s] [get-update] get changed config exception. dataId=%s, group=%s, tenant=%s",
                        agent.getName(), dataId, group, tenant);
                    LOGGER.error(message, ioe);
                }
            }
            for (CacheData cacheData : cacheDatas) {
                if (!cacheData.isInitializing() || inInitializingCacheList
                    .contains(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant))) {
                    cacheData.checkListenerMd5();
                    cacheData.setInitializing(false);
                }
            }
            inInitializingCacheList.clear();

            //长轮询
            executorService.execute(this);

        } catch (Throwable e) {

            // If the rotation training task is abnormal, the next execution time of the task will be punished
            LOGGER.error("longPolling error : ", e);
            //如果轮询异常，下次请求会收到惩罚，默认2000ms，这个值也是可以配置的
            executorService.schedule(this, taskPenaltyTime, TimeUnit.MILLISECONDS);
        }
    }
}
```



#### 拉取的优势

客户端拉取服务端的数据与服务端推送数据给客户端相比，优势在哪呢，为什么 Nacos  不设计成主动推送数据，而是要客户端去拉取呢？如果用推的方式，服务端需要维持与客户端的长连接，这样的话需要耗费大量的资源，并且还需要考虑连接的有效性，例如需要通过心跳来维持两者之间的连接。而用拉的方式，客户端只需要通过一个无状态的  http 请求即可获取到服务端的数据。



### 服务发现config



#### example

```java
public class NamingExample {

    public static void main(String[] args) throws NacosException {

        Properties properties = new Properties();
        properties.setProperty("serverAddr", System.getProperty("serverAddr"));
        properties.setProperty("namespace", System.getProperty("namespace"));

        NamingService naming = NamingFactory.createNamingService(properties);

        naming.registerInstance("nacos.test.3", "11.11.11.11", 8888, "TEST1");

        naming.registerInstance("nacos.test.3", "2.2.2.2", 9999, "DEFAULT");

        System.out.println(naming.getAllInstances("nacos.test.3"));

        naming.deregisterInstance("nacos.test.3", "2.2.2.2", 9999, "DEFAULT");

        System.out.println(naming.getAllInstances("nacos.test.3"));

        naming.subscribe("nacos.test.3", new EventListener() {
            @Override
            public void onEvent(Event event) {
                System.out.println(((NamingEvent)event).getServiceName());
                System.out.println(((NamingEvent)event).getInstances());
            }
        });
    }
}
```



#### 构造service

```java
public static NamingService createNamingService(Properties properties) throws NacosException {
    try {
        Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.naming.NacosNamingService");
        Constructor constructor = driverImplClass.getConstructor(Properties.class);
        NamingService vendorImpl = (NamingService)constructor.newInstance(properties);
        return vendorImpl;
    } catch (Throwable e) {
        throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
    }
}
```



```java
//NacosNamingService
private void init(Properties properties) {
    namespace = InitUtils.initNamespaceForNaming(properties);
    initServerAddr(properties);
    InitUtils.initWebRootContext();
    initCacheDir();
    initLogName(properties);

    eventDispatcher = new EventDispatcher();
    serverProxy = new NamingProxy(namespace, endpoint, serverList);
    serverProxy.setProperties(properties);
    beatReactor = new BeatReactor(serverProxy, initClientBeatThreadCount(properties));
    hostReactor = new HostReactor(eventDispatcher, serverProxy, cacheDir, isLoadCacheAtStart(properties), initPollingThreadCount(properties));
}
```



##### 发送心跳

```java
//BeatReactor.java
public BeatReactor(NamingProxy serverProxy, int threadCount) {
    this.serverProxy = serverProxy;

    executorService = new ScheduledThreadPoolExecutor(threadCount, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r);
            thread.setDaemon(true);
            thread.setName("com.alibaba.nacos.naming.beat.sender");
            return thread;
        }
    });
}

public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
    NAMING_LOGGER.info("[BEAT] adding beat: {} to beat map.", beatInfo);
    String key = buildKey(serviceName, beatInfo.getIp(), beatInfo.getPort());
    BeatInfo existBeat = null;
    //fix #1733
    if ((existBeat = dom2Beat.remove(key)) != null) {
        existBeat.setStopped(true);
    }
    dom2Beat.put(key, beatInfo);
    //单次执行指定延迟时间，默认5s，执行心跳任务
    executorService.schedule(new BeatTask(beatInfo), beatInfo.getPeriod(), TimeUnit.MILLISECONDS);
    MetricsMonitor.getDom2BeatSizeMonitor().set(dom2Beat.size());
}

class BeatTask implements Runnable {

    BeatInfo beatInfo;

    public BeatTask(BeatInfo beatInfo) {
        this.beatInfo = beatInfo;
    }

    @Override
    public void run() {
        if (beatInfo.isStopped()) {
            return;
        }
        //sendBeat就是请求了/instance/beat接口，只返回了一个心跳间隔时长
        long result = serverProxy.sendBeat(beatInfo);
        long nextTime = result > 0 ? result : beatInfo.getPeriod();
        executorService.schedule(new BeatTask(beatInfo), nextTime, TimeUnit.MILLISECONDS);
    }
}
```

##### 服务发现

```java
public HostReactor(EventDispatcher eventDispatcher, NamingProxy serverProxy, String cacheDir,
                   boolean loadCacheAtStart, int pollingThreadCount) {

    executor = new ScheduledThreadPoolExecutor(pollingThreadCount, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r);
            thread.setDaemon(true);
            thread.setName("com.alibaba.nacos.client.naming.updater");
            return thread;
        }
    });

    this.eventDispatcher = eventDispatcher;
    this.serverProxy = serverProxy;
    this.cacheDir = cacheDir;
    if (loadCacheAtStart) {
        this.serviceInfoMap = new ConcurrentHashMap<String, ServiceInfo>(DiskCache.read(this.cacheDir));
    } else {
        this.serviceInfoMap = new ConcurrentHashMap<String, ServiceInfo>(16);
    }

    this.updatingMap = new ConcurrentHashMap<String, Object>();
    this.failoverReactor = new FailoverReactor(this, cacheDir);
    this.pushReceiver = new PushReceiver(this);
}

public class UpdateTask implements Runnable {
    long lastRefTime = Long.MAX_VALUE;
    private String clusters;
    private String serviceName;

    public UpdateTask(String serviceName, String clusters) {
        this.serviceName = serviceName;
        this.clusters = clusters;
    }

    @Override
    public void run() {
        try {
            ServiceInfo serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));

            if (serviceObj == null) {
                updateServiceNow(serviceName, clusters);
                executor.schedule(this, DEFAULT_DELAY, TimeUnit.MILLISECONDS);
                return;
            }

            if (serviceObj.getLastRefTime() <= lastRefTime) {
                updateServiceNow(serviceName, clusters);
                serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));
            } else {
                // if serviceName already updated by push, we should not override it
                // since the push data may be different from pull through force push
                refreshOnly(serviceName, clusters);
            }

            executor.schedule(this, serviceObj.getCacheMillis(), TimeUnit.MILLISECONDS);

            lastRefTime = serviceObj.getLastRefTime();
        } catch (Throwable e) {
            NAMING_LOGGER.warn("[NA] failed to update serviceName: " + serviceName, e);
        }

    }
}
```



#### 服务注册

```java
//NacosNamingService.java
@Override
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {

    if (instance.isEphemeral()) {
        BeatInfo beatInfo = new BeatInfo();
        beatInfo.setServiceName(NamingUtils.getGroupedName(serviceName, groupName));
        beatInfo.setIp(instance.getIp());
        beatInfo.setPort(instance.getPort());
        beatInfo.setCluster(instance.getClusterName());
        beatInfo.setWeight(instance.getWeight());
        beatInfo.setMetadata(instance.getMetadata());
        beatInfo.setScheduled(false);
        long instanceInterval = instance.getInstanceHeartBeatInterval();
        //心跳间隔默认5s
        beatInfo.setPeriod(instanceInterval == 0 ? DEFAULT_HEART_BEAT_INTERVAL : instanceInterval);
        //心跳定时任务
        beatReactor.addBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), beatInfo);
    }
	//服务注册
    serverProxy.registerService(NamingUtils.getGroupedName(serviceName, groupName), groupName, instance);
}
```



```java
//NamingProxy.java
public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {

    NAMING_LOGGER.info("[REGISTER-SERVICE] {} registering service {} with instance: {}",
        namespaceId, serviceName, instance);

    final Map<String, String> params = new HashMap<String, String>(9);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put(CommonParams.GROUP_NAME, groupName);
    params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
    params.put("ip", instance.getIp());
    params.put("port", String.valueOf(instance.getPort()));
    params.put("weight", String.valueOf(instance.getWeight()));
    params.put("enable", String.valueOf(instance.isEnabled()));
    params.put("healthy", String.valueOf(instance.isHealthy()));
    params.put("ephemeral", String.valueOf(instance.isEphemeral()));
    params.put("metadata", JSON.toJSONString(instance.getMetadata()));

    reqAPI(UtilAndComs.NACOS_URL_INSTANCE, params, HttpMethod.POST);

}
```



#### 监听服务

```java
//NacosNamingService.java
@Override
public void subscribe(String serviceName, String groupName, List<String> clusters, EventListener listener) throws NacosException {
    eventDispatcher.addListener(hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName),
        StringUtils.join(clusters, ",")), StringUtils.join(clusters, ","), listener);
}
```



```java
//EventDispatcher.java 监听器调度类
public EventDispatcher() {
    //单线程线程池
    executor = Executors.newSingleThreadExecutor(new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r, "com.alibaba.nacos.naming.client.listener");
            thread.setDaemon(true);

            return thread;
        }
    });

    executor.execute(new Notifier());
}

private class Notifier implements Runnable {
    @Override
    public void run() {
        //轮询
        while (true) {
            ServiceInfo serviceInfo = null;
            try {
                //出队，如果队列空，最多等待5分钟，如果超时还是空，返回null
                serviceInfo = changedServices.poll(5, TimeUnit.MINUTES);
            } catch (Exception ignore) {
            }

            if (serviceInfo == null) {
                continue;
            }

            try {
                List<EventListener> listeners = observerMap.get(serviceInfo.getKey());
                //监听器分发事件
                if (!CollectionUtils.isEmpty(listeners)) {
                    for (EventListener listener : listeners) {
                        List<Instance> hosts = Collections.unmodifiableList(serviceInfo.getHosts());
                        listener.onEvent(new NamingEvent(serviceInfo.getName(), serviceInfo.getGroupName(), serviceInfo.getClusters(), hosts));
                    }
                }

            } catch (Exception e) {
                NAMING_LOGGER.error("[NA] notify error for service: "
                                    + serviceInfo.getName() + ", clusters: " + serviceInfo.getClusters(), e);
            }
        }
    }
}
```



### 一致性协议

[Nacos 注册中心的设计原理详解](https://www.infoq.cn/article/B*6vyMIKao9vAKIsJYpE)

数据一致性是分布式系统永恒的话题，Paxos 协议的艰深更让数据一致性成为程序员大牛们吹水的常见话题。不过从协议层面上看，一致性的选型已经很长时间没有新的成员加入了。目前来看基本可以归为两家：一种是基于 Leader 的非对等部署的单点写一致性，一种是对等部署的多写一致性。当我们选用服务注册中心的时候，并没有一种协议能够覆盖所有场景，例如当注册的服务节点不会定时发送心跳到注册中心时，强一致协议看起来是唯一的选择，因为无法通过心跳来进行数据的补偿注册，第一次注册就必须保证数据不会丢失。而当客户端会定时发送心跳来汇报健康状态时，第一次的注册的成功率并不是非常关键（当然也很关键，只是相对来说我们容忍数据的少量写失败），因为后续还可以通过心跳再把数据补偿上来，此时 Paxos 协议的单点瓶颈就会不太划算了，这也是 Eureka 为什么不采用 Paxos 协议而采用自定义的 Renew 机制的原因。

这两种数据一致性协议有各自的使用场景，对服务注册的需求不同，就会导致使用不同的协议。Nacos 因为要支持多种服务类型的注册，并能够具有机房容灾、集群扩展等必不可少的能力，在 1.0.0 正式支持 AP 和 CP 两种一致性协议并存。1.0.0 重构了数据的读写和同步逻辑，将与业务相关的 CRUD 与底层的一致性同步逻辑进行了分层隔离。然后将业务的读写（主要是写，因为读会直接使用业务层的缓存）抽象为 Nacos 定义的数据类型，调用一致性服务进行数据同步。在决定使用 CP 还是 AP 一致性时，使用一个代理，通过可控制的规则进行转发。

目前的一致性协议实现，一个是基于简化的 Raft 的 CP 一致性，一个是基于自研协议 Distro 的 AP 一致性。Raft 协议不必多言，基于 Leader 进行写入，其 CP 也并不是严格的，只是能保证一半所见一致，以及数据的丢失概率较小。Distro 协议则是参考了内部 ConfigServer 和开源 Eureka，在不借助第三方存储的情况下，实现基本大同小异。Distro 重点是做了一些逻辑的优化和性能的调优。



#### Distro算法实现ap

Eureka是一个AP模式的服务发现框架，在Eureka集群模式下，Eureka采取的是Server之间互相广播各自的数据进行数据复制、更新操作；并且Eureka在客户端与注册中心出现网络故障时，依然能够获取服务注册信息——Eureka实现了客户端对于服务注册信息的缓存

Nacos在AP模式下的一致性策略就类似于Eureka，采用`Server`之间互相的数据同步来实现数据在集群中的同步、复制操作。



#### raft算法实现cp

[Spring Cloud Alibaba Nacos（心跳与选举）](https://segmentfault.com/a/1190000019698113)

[Nacos数据一致性](https://blog.csdn.net/liyanan21/article/details/89320872)

[Nacos中Raft算法的实现](https://blog.csdn.net/smlCSDN/article/details/100099207)

[动画演示](<http://raft.taillog.cn/>)

在Raft中，节点有三种角色：

- Leader：领导者
- Candidate：候选人
- Follower：跟随者

##### 选举

服务启动和leader挂了，会发生选举：

所有节点启动的时候，都是follower状态。 如果在一段时间内如果没有收到leader的心跳（可能是没有leader，也可能是leader挂了），那么follower会变成Candidate。然后发起选举，选举之前，会增加term，这个term和zookeeper中的epoch的道理是一样的。

leader会向follower发送心跳，一段时间follower获取不到心跳，那么follower会变成Candidate，进行选举，选举之前，Candidate的term任期会增加，给自己投一票，发送给其他的follower，其他follower发现Candidate的term比自己的大，就投票给这个Candidate，否则投票给自己，投票超过半数就选举成功。

在Raft算法中，会有两个超时时间设置来控制选举过程

首先是 竞选超时：竞选超时是指跟随者成为候选人的时间，竞选超时一般是150毫秒到300毫秒之间的随机数，当到达竞选超时时间后，跟随者转变为候选人角色，并进入到 选举周期，为自己发起投票，此时候选人将发送vote请求给其它节点，如果收到请求的节点在当前选举周期中还没有投过票，则这个节点会投票给这个候选人，然后这个节点重置它的选举周期时间，重新计时，一旦候选人获得半数以上的赞成投票，那么它将成为领导人，之后领导人将发送 附加日志 指令给跟随者，这些消息是周期性发送，也叫 心跳包（以此来保证它的领导人地位），跟随者将响应 附加日志 消息，选举周期将一直持续直到某个跟随者没有收到心跳包并成为候选人

在同一个周期里有两个节点同时发起了竞选请求，并且每个都收到了一个跟随者的投票响应，现在，每个候选人都有两票，并且都无法获得更多选票，这种情况下，两个节点将等待一轮竞选超时后重新发起竞选请求

##### 日志复制

一旦选举出了领导者，我们需要向所有节点通知这一消息，并需要持续维持领导人地位，这是通过周期性的发送 ﻿附加日志 消息（心跳包）实现的

首先，一个客户端发送变化的数据给领导人，这条变更记录被添加到领导人的日志里，然后在下一个心跳中将变更记录发送给跟随者，一旦大多数的跟随者确认了这条记录，那么这条记录就会被提交，最后将响应客户端。

## 云服务

### ACM

Spring Cloud AliCloud ACM 是阿里云提供的商业版应用配置管理(Application Configuration Management) 产品 在 Spring Cloud 应用侧的客户端实现，且目前完全免费。

外网无法访问，只有阿里云的云服务器上才能直接访问。只有配置中心功能，没有服务管理功能。



### 微服务引擎

[微服务引擎  ( MSE )](https://www.aliyun.com/product/mse?spm=nacos-website.topbar.0.0.0) 是开源注册、配置中心的全托管平台，提供高可用、免运维的 ZooKeeper、Nacos 和 Eureka  等集群，完全兼容开源产品标准接口，无需修改代码、开箱即用，并为客户提供相应的监控和运维工具 　　　  目前公测已结束，将于2020年1月1日商业化。对Nacos的配置中心托管功能正在开发中，上线后不会收取额外费用。