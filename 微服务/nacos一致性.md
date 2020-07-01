# nacos 一致性协议

[Nacos 注册中心的设计原理详解](https://www.infoq.cn/article/B*6vyMIKao9vAKIsJYpE)

数据一致性是分布式系统永恒的话题，Paxos 协议的艰深更让数据一致性成为程序员大牛们吹水的常见话题。不过从协议层面上看，一致性的选型已经很长时间没有新的成员加入了。目前来看基本可以归为两家：一种是基于 Leader 的非对等部署的单点写一致性，一种是对等部署的多写一致性。当我们选用服务注册中心的时候，并没有一种协议能够覆盖所有场景，例如当注册的服务节点不会定时发送心跳到注册中心时，强一致协议看起来是唯一的选择，因为无法通过心跳来进行数据的补偿注册，第一次注册就必须保证数据不会丢失。而当客户端会定时发送心跳来汇报健康状态时，第一次的注册的成功率并不是非常关键（当然也很关键，只是相对来说我们容忍数据的少量写失败），因为后续还可以通过心跳再把数据补偿上来，此时 Paxos 协议的单点瓶颈就会不太划算了，这也是 Eureka 为什么不采用 Paxos 协议而采用自定义的 Renew 机制的原因。

这两种数据一致性协议有各自的使用场景，对服务注册的需求不同，就会导致使用不同的协议。Nacos 因为要支持多种服务类型的注册，并能够具有机房容灾、集群扩展等必不可少的能力，在 1.0.0 正式支持 AP 和 CP 两种一致性协议并存。1.0.0 重构了数据的读写和同步逻辑，将与业务相关的 CRUD 与底层的一致性同步逻辑进行了分层隔离。然后将业务的读写（主要是写，因为读会直接使用业务层的缓存）抽象为 Nacos 定义的数据类型，调用一致性服务进行数据同步。在决定使用 CP 还是 AP 一致性时，使用一个代理，通过可控制的规则进行转发。

目前的一致性协议实现，一个是基于简化的 Raft 的 CP 一致性，一个是基于自研协议 Distro 的 AP 一致性。Raft 协议不必多言，基于 Leader 进行写入，其 CP 也并不是严格的，只是能保证一半所见一致，以及数据的丢失概率较小。Distro 协议则是参考了内部 ConfigServer 和开源 Eureka，在不借助第三方存储的情况下，实现基本大同小异。Distro 重点是做了一些逻辑的优化和性能的调优。

临时实例使用ap，nacos自己实现的raft算法

持久实例使用cp，nacos自己实现的Distro 协议

# Distro算法实现ap

[Nacos一致性协议实现之Distro协议浅析](https://www.liaochuntao.cn/2019/09/16/java-web-55/)

Eureka是一个AP模式的服务发现框架，在Eureka集群模式下，Eureka采取的是Server之间互相广播各自的数据进行数据复制、更新操作；并且Eureka在客户端与注册中心出现网络故障时，依然能够获取服务注册信息——Eureka实现了客户端对于服务注册信息的缓存

Nacos在AP模式下的一致性策略就类似于Eureka，采用`Server`之间互相的数据同步来实现数据在集群中的同步、复制操作。

`DistroConsistency`是针对注册形式为**临时实例**的`Instance`，该一致性策略其实是阿里`Nacos`团队根据他们内部的`ConfigServer`以及`Eureka`的一致性策略来实现的，`Distro`一致性策略的实现，在构造函数执行完后，会执行一个`init`函数，`init`函数内部提交了一个任务，任务里面执行了一个`load`函数以及提交了一个`Notifier`的`Runnable`；而这个`load`函数，会先判断`Nacos`的启动是单机模式还是集群模式，如果是集群模式那就很简单了；但是如果是集群模式下的话，就比较多了，首先会判断当前`Nacos`集群中的`server`数量是否为1，如果为1的话，表示当前集群中只有一个`server`实例，因此会让线程休眠一下，等待其他的`server`加入然后根据远程的`server`进行数据的远程同步

## 数据同步

```java
//DistroConsistencyServiceImpl
public void load() throws Exception {
  if (SystemUtils.STANDALONE_MODE) {
    initialized = true;
    return;
  }
  // size = 1 means only myself in the list, we need at least one another server alive:
  // 集群模式下，需要等待至少两个节点才可以将逻辑进行
  while (serverListManager.getHealthyServers().size() <= 1) {
    Thread.sleep(1000L);
    Loggers.DISTRO.info("waiting server list init...");
  }

  // 获取所有健康的集群节点
  for (Server server : serverListManager.getHealthyServers()) {
    // 自己则不需要进行数据同步广播操作
    if (NetUtils.localServer().equals(server.getKey())) {
      continue;
    }
    if (Loggers.DISTRO.isDebugEnabled()) {
      Loggers.DISTRO.debug("sync from " + server);
    }
    // 从别的服务器进行全量数据拉取操作，只需要执行一次即可，剩下的交由增量同步任务去完成
    if (syncAllDataFromRemote(server)) {
      initialized = true;
      return;
    }
  }
}
```

## 全量拉取

### 拉取

```java
public boolean syncAllDataFromRemote(Server server) {
  try {
    // 获取数据
    byte[] data = NamingProxy.getAllData(server.getKey());
    // 接收到的数据进行处理
    processData(data);
    return true;
  } catch (Exception e) {
    Loggers.DISTRO.error("sync full data from " + server + " failed!", e);
    return false;
  }
}

public void processData(byte[] data) throws Exception {
        if (data.length > 0) {
            // 先将数据进行反序列化
            Map<String, Datum<Instances>> datumMap =
                serializer.deserializeMap(data, Instances.class);

            // 对数据进行遍历处理
            for (Map.Entry<String, Datum<Instances>> entry : datumMap.entrySet()) {
                // 数据放入数据存储容器——DataStore中
                dataStore.put(entry.getKey(), entry.getValue());
                // 判断监听器是否包含了对这个Key的监听，如果没有，表明是一个新的数据
                if (!listeners.containsKey(entry.getKey())) {
                    // pretty sure the service not exist:
                    if (switchDomain.isDefaultInstanceEphemeral()) {
                        // create empty service
                        Loggers.DISTRO.info("creating service {}", entry.getKey());
                        Service service = new Service();
                        String serviceName = KeyBuilder.getServiceName(entry.getKey());
                        String namespaceId = KeyBuilder.getNamespace(entry.getKey());
                        service.setName(serviceName);
                        service.setNamespaceId(namespaceId);
                        service.setGroupName(Constants.DEFAULT_GROUP);
                        // now validate the service. if failed, exception will be thrown
                        service.setLastModifiedMillis(System.currentTimeMillis());
                        service.recalculateChecksum();
                        // 回调 Listener 监听器，告知新的Service数据
                        listeners.get(KeyBuilder.SERVICE_META_KEY_PREFIX).get(0)
                            .onChange(KeyBuilder.buildServiceMetaKey(namespaceId, serviceName), service);
                    }
                }
            }
            // 进行 Listener 的监听回调
            for (Map.Entry<String, Datum<Instances>> entry : datumMap.entrySet()) {
                if (!listeners.containsKey(entry.getKey())) {
                    // Should not happen:
                    Loggers.DISTRO.warn("listener of {} not found.", entry.getKey());
                    continue;
                }

                try {
                    for (RecordListener listener : listeners.get(entry.getKey())) {
                        listener.onChange(entry.getKey(), entry.getValue().value);
                    }
                } catch (Exception e) {
                    Loggers.DISTRO.error("[NACOS-DISTRO] error while execute listener of key: {}", entry.getKey(), e);
                    continue;
                }

                // Update data store if listener executed successfully:
                dataStore.put(entry.getKey(), entry.getValue());
            }
        }
    }
```

### 响应

```java
@RequestMapping(value = "/datums", method = RequestMethod.GET)
public ResponseEntity getAllDatums(HttpServletRequest request, HttpServletResponse response) throws Exception {
  // 直接将存储的数据容器——Map进行序列化传输
  String content = new String(serializer.serialize(dataStore.getDataMap()), StandardCharsets.UTF_8);
  return ResponseEntity.ok(content);
}
```



## 增量同步

### 同步

```java
public class TaskDispatcher {

    @Autowired
    private GlobalConfig partitionConfig;

    @Autowired
    private DataSyncer dataSyncer;

    private List<TaskScheduler> taskSchedulerList = new ArrayList<>();

    private final int cpuCoreCount = Runtime.getRuntime().availableProcessors();

    @PostConstruct
    public void init() {
        // 构建任务执行器
        for (int i = 0; i < cpuCoreCount; i++) {
            TaskScheduler taskScheduler = new TaskScheduler(i);
            taskSchedulerList.add(taskScheduler);
            // 任务调度执行器提交
            GlobalExecutor.submitTaskDispatch(taskScheduler);
        }
    }

    public void addTask(String key) {
        // 根据 Key 进行 Hash 找到一个 TaskScheduler 进行任务提交
        taskSchedulerList.get(UtilsAndCommons.shakeUp(key, cpuCoreCount)).addTask(key);
    }

    public class TaskScheduler implements Runnable {

        private int index;

        private int dataSize = 0;

        private long lastDispatchTime = 0L;

        private BlockingQueue<String> queue = new LinkedBlockingQueue<>(128 * 1024);

        public TaskScheduler(int index) {
            this.index = index;
        }

        public void addTask(String key) {
            queue.offer(key);
        }

        public int getIndex() {
            return index;
        }

        @Override
        public void run() {
            List<String> keys = new ArrayList<>();
            while (true) {
                try {
                    // 从任务缓存队列中获取一个任务（存在超时设置）
                    String key = queue.poll(partitionConfig.getTaskDispatchPeriod(),
                        TimeUnit.MILLISECONDS);
                    if (Loggers.DISTRO.isDebugEnabled() && StringUtils.isNotBlank(key)) {
                        Loggers.DISTRO.debug("got key: {}", key);
                    }
                    // 如果不存在集群或者集群节点为空
                    if (dataSyncer.getServers() == null || dataSyncer.getServers().isEmpty()) {
                        continue;
                    }
                    if (StringUtils.isBlank(key)) {
                        continue;
                    }
                    if (dataSize == 0) {
                        keys = new ArrayList<>();
                    }
                    // 进行批量任务处理，这里做一次暂存操作，为了避免
                    keys.add(key);
                    dataSize++;
                    // 如果此时的任务暂存数量达到了指定的批量，或者任务的时间达到了最大设定，进行数据同步任务
                    if (dataSize == partitionConfig.getBatchSyncKeyCount() ||
                        (System.currentTimeMillis() - lastDispatchTime) > partitionConfig.getTaskDispatchPeriod()) {
                        // 为每一个server创建一个SyncTask任务
                        for (Server member : dataSyncer.getServers()) {
                            if (NetUtils.localServer().equals(member.getKey())) {
                                continue;
                            }
                            SyncTask syncTask = new SyncTask();
                            syncTask.setKeys(keys);
                            syncTask.setTargetServer(member.getKey());
                            if (Loggers.DISTRO.isDebugEnabled() && StringUtils.isNotBlank(key)) {
                                Loggers.DISTRO.debug("add sync task: {}", JSON.toJSONString(syncTask));
                            }
                            // 进行任务提交，同时设置任务延迟执行时间，这里设置为立即执行
                            dataSyncer.submit(syncTask, 0);
                        }
                        lastDispatchTime = System.currentTimeMillis();
                        dataSize = 0;
                    }
                } catch (Exception e) {
                    Loggers.DISTRO.error("dispatch sync task failed.", e);
                }
            }
        }
    }
}
```

#### DataSyncer

```java
public class DataSyncer {

    ...

    @PostConstruct
    public void init() {
        // 执行定期的数据同步任务（每五秒执行一次）
        startTimedSync();
    }

    // 任务提交
    public void submit(SyncTask task, long delay) {
        // If it's a new task:
        if (task.getRetryCount() == 0) {
            // 遍历所有的任务 Key
            Iterator<String> iterator = task.getKeys().iterator();
            while (iterator.hasNext()) {
                String key = iterator.next();
                // 数据任务放入 Map 中，避免数据同步任务重复提交
                if (StringUtils.isNotBlank(taskMap.putIfAbsent(buildKey(key, task.getTargetServer()), key))) {
                    // associated key already exist:
                    if (Loggers.DISTRO.isDebugEnabled()) {
                        Loggers.DISTRO.debug("sync already in process, key: {}", key);
                    }
                    // 如果任务已经存在，则移除该任务的 Key
                    iterator.remove();
                }
            }
        }
        // 如果所有的任务都已经移除了，结束本次任务提交
        if (task.getKeys().isEmpty()) {
            // all keys are removed:
            return;
        }
        // 异步任务执行数据同步
        GlobalExecutor.submitDataSync(() -> {
            // 1. check the server
            if (getServers() == null || getServers().isEmpty()) {
                Loggers.SRV_LOG.warn("try to sync data but server list is empty.");
                return;
            }
            // 获取数据同步任务的实际同步数据
            List<String> keys = task.getKeys();
            if (Loggers.SRV_LOG.isDebugEnabled()) {
                Loggers.SRV_LOG.debug("try to sync data for this keys {}.", keys);
            }
            // 2. get the datums by keys and check the datum is empty or not
            // 通过key进行批量数据获取
            Map<String, Datum> datumMap = dataStore.batchGet(keys);
            // 如果数据已经被移除了，取消本次任务
            if (datumMap == null || datumMap.isEmpty()) {
                // clear all flags of this task:
                for (String key : keys) {
                    taskMap.remove(buildKey(key, task.getTargetServer()));
                }
                return;
            }
            // 数据序列化
            byte[] data = serializer.serialize(datumMap);
            long timestamp = System.currentTimeMillis();
            // 进行增量数据同步提交给其他节点
            boolean success = NamingProxy.syncData(data, task.getTargetServer());
            // 如果本次数据同步任务失败，则重新创建SyncTask，设置重试的次数信息
            if (!success) {
                SyncTask syncTask = new SyncTask();
                syncTask.setKeys(task.getKeys());
                syncTask.setRetryCount(task.getRetryCount() + 1);
                syncTask.setLastExecuteTime(timestamp);
                syncTask.setTargetServer(task.getTargetServer());
                retrySync(syncTask);
            } else {
                // clear all flags of this task:
                for (String key : task.getKeys()) {
                    taskMap.remove(buildKey(key, task.getTargetServer()));
                }
            }
        }, delay);
    }

    // 任务重试
    public void retrySync(SyncTask syncTask) {
        Server server = new Server();
        server.setIp(syncTask.getTargetServer().split(":")[0]);
        server.setServePort(Integer.parseInt(syncTask.getTargetServer().split(":")[1]));
        if (!getServers().contains(server)) {
            // if server is no longer in healthy server list, ignore this task:
            return;
        }
        // TODO may choose other retry policy.
        // 自动延迟重试任务的下次执行时间
        submit(syncTask, partitionConfig.getSyncRetryDelay());
    }

    public void startTimedSync() {
        GlobalExecutor.schedulePartitionDataTimedSync(new TimedSync());
    }

    // 执行周期任务
    // 每次将自己负责的数据进行广播到其他的 Server 节点
    public class TimedSync implements Runnable {

        @Override
        public void run() {
            try {
                if (Loggers.DISTRO.isDebugEnabled()) {
                    Loggers.DISTRO.debug("server list is: {}", getServers());
                }
                // send local timestamps to other servers:
                Map<String, String> keyChecksums = new HashMap<>(64);
                // 对数据存储容器的
                for (String key : dataStore.keys()) {
                    // 如果自己不是负责此数据的权威 Server，则无权对此数据做集群间的广播通知操作
                    if (!distroMapper.responsible(KeyBuilder.getServiceName(key))) {
                        continue;
                    }
                    // 获取数据操作，
                    Datum datum = dataStore.get(key);
                    if (datum == null) {
                        continue;
                    }
                    // 放入数据广播列表
                    keyChecksums.put(key, datum.value.getChecksum());
                }
                if (keyChecksums.isEmpty()) {
                    return;
                }
                if (Loggers.DISTRO.isDebugEnabled()) {
                    Loggers.DISTRO.debug("sync checksums: {}", keyChecksums);
                }
                // 对集群的所有节点（除了自己），做数据广播操作
                for (Server member : getServers()) {
                    if (NetUtils.localServer().equals(member.getKey())) {
                        continue;
                    }
                    // 集群间的数据广播操作
                    NamingProxy.syncCheckSums(keyChecksums, member.getKey());
                }
            } catch (Exception e) {
                Loggers.DISTRO.error("timed sync task failed.", e);
            }
        }
    }

    public List<Server> getServers() {
        return serverListManager.getHealthyServers();
    }

    public String buildKey(String key, String targetServer) {
        return key + UtilsAndCommons.CACHE_KEY_SPLITER + targetServer;
    }
}
```

### 响应

```java
`@RequestMapping(value = "/checksum", method = RequestMethod.PUT)public ResponseEntity syncChecksum(HttpServletRequest request, HttpServletResponse response) throws Exception {    // 由那个节点传输而来的数据    String source = WebUtils.required(request, "source");    String entity = IOUtils.toString(request.getInputStream(), "UTF-8");    // 数据序列化    Map<String, String> dataMap = serializer.deserialize(entity.getBytes(), new TypeReference<Map<String, String>>() {});    // 数据接收操作    consistencyService.onReceiveChecksums(dataMap, source);    return ResponseEntity.ok("ok");}`
```

# raft算法实现cp

[Spring Cloud Alibaba Nacos（心跳与选举）](https://segmentfault.com/a/1190000019698113)

[Nacos数据一致性](https://blog.csdn.net/liyanan21/article/details/89320872)

[Nacos中Raft算法的实现](https://blog.csdn.net/smlCSDN/article/details/100099207)

[动画演示](<http://raft.taillog.cn/>)

在Raft中，节点有三种角色：

- Leader：领导者
- Candidate：候选人
- Follower：跟随者

## 选举

服务启动和leader挂了，会发生选举：

所有节点启动的时候，都是follower状态。 如果在一段时间内如果没有收到leader的心跳（可能是没有leader，也可能是leader挂了），那么follower会变成Candidate。然后发起选举，选举之前，会增加term，这个term和zookeeper中的epoch的道理是一样的。

leader会向follower发送心跳，一段时间follower获取不到心跳，那么follower会变成Candidate，进行选举，选举之前，Candidate的term任期会增加，给自己投一票，发送给其他的follower，其他follower发现Candidate的term比自己的大，就投票给这个Candidate，否则投票给自己，投票超过半数就选举成功。

在Raft算法中，会有两个超时时间设置来控制选举过程

首先是 竞选超时：竞选超时是指跟随者成为候选人的时间，竞选超时一般是150毫秒到300毫秒之间的随机数，当到达竞选超时时间后，跟随者转变为候选人角色，并进入到 选举周期，为自己发起投票，此时候选人将发送vote请求给其它节点，如果收到请求的节点在当前选举周期中还没有投过票，则这个节点会投票给这个候选人，然后这个节点重置它的选举周期时间，重新计时，一旦候选人获得半数以上的赞成投票，那么它将成为领导人，之后领导人将发送 附加日志 指令给跟随者，这些消息是周期性发送，也叫 心跳包（以此来保证它的领导人地位），跟随者将响应 附加日志 消息，选举周期将一直持续直到某个跟随者没有收到心跳包并成为候选人

在同一个周期里有两个节点同时发起了竞选请求，并且每个都收到了一个跟随者的投票响应，现在，每个候选人都有两票，并且都无法获得更多选票，这种情况下，两个节点将等待一轮竞选超时后重新发起竞选请求

## 日志复制

一旦选举出了领导者，我们需要向所有节点通知这一消息，并需要持续维持领导人地位，这是通过周期性的发送 ﻿附加日志 消息（心跳包）实现的

首先，一个客户端发送变化的数据给领导人，这条变更记录被添加到领导人的日志里，然后在下一个心跳中将变更记录发送给跟随者，一旦大多数的跟随者确认了这条记录，那么这条记录就会被提交，最后将响应客户端。



# eureka数据一致性

![img](https:////upload-images.jianshu.io/upload_images/15462057-98273a771c7cb009?imageMogr2/auto-orient/strip|imageView2/2/w/1080)



服务注册中心不可能是单点的，一定会有一个集群，那么集群中的服务注册信息如何在集群中保持一致的呢？

首先要明确的是 Eureka 是**弱数据一致性**的。

下面从2个方面来说明：

1. 什么是弱数据一致性
2. Eureka 是如何同步数据的

## 弱一致性

我们知道 ZooKeeper 也可以实现数据中心，ZooKeeper 就是强一致性的。

分布式系统中有一个重要理论：CAP。



![img](https:////upload-images.jianshu.io/upload_images/15462057-b458f30e35e7d2f1?imageMogr2/auto-orient/strip|imageView2/2/w/1080)



该理论提到了分布式系统中的3个特性：

- Consistency 数据一致性

分布式系统中，数据会存在多个副本中，有一些问题会导致写入数据时，一部分副本成功、一部分副本失败，造成数据不一致。

满足一致性就要求对数据的更新操作成功后，多副本的数据必须保持一致。

- Availability 可用性

在任何时候客户端对集群进行读写操作时，请求能够正常响应。

- Partition Tolerance 分区容忍性

发生通信故障时，集群被分割为多个无法通信的分区时，集群仍然可用。

> CAP 理论指出：这3个特性不可能同时满足，最多满足2个。

**P** 是客观存在的，*不可绕过*，那么就是选择 **C** 还是选择 **A**。

ZooKeeper 选择了 **C**，就是尽可能的保证数据一致性，某些情况下可以牺牲可用性。

Eureka 则选择了 **A**，所以 Eureka 具有高可用性，在任何时候，服务消费者都能正常获取服务列表，但不保证数据的强一致性，消费者可能会拿到过期的服务列表。



![img](https:////upload-images.jianshu.io/upload_images/15462057-1be9b8ed0f306066?imageMogr2/auto-orient/strip|imageView2/2/w/1054)



> Eureka 的设计理念：保留可用及过期的数据总比丢掉可用的数据好。

## Eureka 的数据同步方式

### 复制方式

分布式系统的数据在多个副本之间的复制方式，主要有：

- 主从复制

就是 **Master-Slave** 模式，有一个主副本，其他为从副本，所有写操作都提交到主副本，再由主副本更新到其他从副本。

写压力都集中在主副本上，是系统的瓶颈，从副本可以分担读请求。

- 对等复制

就是 **Peer to Peer** 模式，副本间不分主从，任何副本都可以接收写操作，然后每个副本间互相进行数据更新。

对等复制模式，任何副本都可以接收写请求，不存在写压力瓶颈，但各个副本间数据同步时可能产生数据冲突。

Eureka 采用的就是 **Peer to Peer** 模式。

### 同步过程

Eureka Server 本身依赖了 Eureka Client，也就是每个 Eureka Server 是作为其他 Eureka Server 的 Client。

Eureka Server 启动后，会通过 Eureka Client 请求其他 Eureka Server 节点中的一个节点，获取注册的服务信息，然后复制到其他 peer 节点。

Eureka Server 每当自己的信息变更后，例如 Client 向自己发起*注册、续约、注销*请求， 就会把自己的最新信息通知给其他 Eureka Server，保持数据同步。



![img](https:////upload-images.jianshu.io/upload_images/15462057-85c320cc033e1489?imageMogr2/auto-orient/strip|imageView2/2/w/1080)



如果自己的信息变更是另一个Eureka Server同步过来的，这是再同步回去的话就出现**数据同步死循环**了。



![img](https:////upload-images.jianshu.io/upload_images/15462057-984ff24cbd57da49?imageMogr2/auto-orient/strip|imageView2/2/w/1080)



Eureka Server 在执行复制操作的时候，使用 `HEADER_REPLICATION` 这个 http header 来区分普通应用实例的正常请求，说明这是一个复制请求，这样其他 peer 节点收到请求时，就不会再对其进行复制操作，从而避免死循环。

还有一个问题，就是**数据冲突**，比如 server A 向 server B 发起同步请求，如果 A 的数据比 B 的还旧，B 不可能接受 A 的数据，那么 B 是如何知道 A 的数据是旧的呢？这时 A 又应该怎么办呢？

数据的新旧一般是通过*版本号*来定义的，Eureka 是通过 `lastDirtyTimestamp` 这个类似版本号的属性来实现的。

> `lastDirtyTimestamp` 是注册中心里面服务实例的一个属性，表示此服务实例最近一次变更时间。

比如 Eureka Server A 向 Eureka Server B 复制数据，数据冲突有2种情况：

（1）A 的数据比 B 的新，B 返回 404，A 重新把这个应用实例注册到 B。

（2）A 的数据比 B 的旧，B 返回 409，要求 A 同步 B 的数据。



![img](https:////upload-images.jianshu.io/upload_images/15462057-11138eda064fe055?imageMogr2/auto-orient/strip|imageView2/2/w/1080)



还有一个重要的机制：**hearbeat 心跳**，即续约操作，来进行数据的最终修复，因为节点间的复制可能会出错，通过心跳就可以发现错误，进行弥补。

例如发现某个应用实例数据与某个server不一致，则server放回404，实例重新注册即可。

