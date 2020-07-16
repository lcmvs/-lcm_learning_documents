# Kafka

[Kafka、RabbitMQ、RocketMQ消息中间件的对比 —— 消息发送性能](http://jm.taobao.org/2016/04/01/kafka-vs-rabbitmq-vs-rocketmq-message-send-performance/)

• 使用推送和拉取模型解耦生产者和消费者；
• 为消息传递系统中的消息提供数据持久化，以便支持多个消费者；
• 通过系统优化实现高吞吐量；
• 系统可以随着数据流的增长进行横向扩展。 

## 安装部署

1.官网[下载](http://kafka.apache.org/downloads)

2.解压缩tar -xzvf kafka_2.11-2.4.1.tgz

| 目录   | 说明                                                         |
| ------ | ------------------------------------------------------------ |
| bin    | 操作kafka的可执行脚本，还包含windows下脚本                   |
| config | 配置文件所在目录                                             |
| libs   | 依赖库目录                                                   |
| logs   | 日志数据目录，目录kafka把server端日志分为5种类型，分为:server,request,state，log-cleaner，controller |

3.修改配置文件config/server.properties

[server.properties配置文件参数说明](https://blog.csdn.net/lizhitao/article/details/25667831)

```properties
broker.id=0
num.network.threads=2
num.io.threads=8
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
socket.request.max.bytes=104857600
log.dirs=/tmp/kafka-logs
num.partitions=2
log.retention.hours=168

log.segment.bytes=536870912
log.retention.check.interval.ms=60000
log.cleaner.enable=false

zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=1000000
```

测试环境可能jvm可以申请的内存不够，可以打开/config/kafka-server-start.sh

修改 `export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"` 
`export KAFKA_HEAP_OPTS="-Xmx256M -Xms128M"`

启动zookeeper 

4.启动服务 nohup ./bin/kafka-server-start.sh config/server.properties &

外网设置 advertised.listeners=PLAINTEXT://外网ip:9092

5.验证服务

```shell
# 创建一个主题
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
# 查看已创建的topic信息
bin/kafka-topics.sh --list --zookeeper localhost:2181
# 发送消息
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
# 消费消息
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

## broker配置



### 常规配置

1. broker.id
   每个 broker 都需要有一个标识符，使用 broker.id 来表示。它的默认值是 0，也可以被设置成其他任意整数。这个值在整个 Kafka 集群里必须是唯一的。这个值可以任意选定，如果出于维护的需要，可以在服务器节点间交换使用这些 ID。建议把它们设置成与机器名具有相关性的整数，这样在进行维护时，将 ID 号映射到机器名就没那么麻烦了。例如，如果机器名包含唯一性的数字（比如 host1.example.com、 host2.example.com），那么用这些数字来设置 broker.id 就再好不过了。
2. port
   如果使用配置样本来启动 Kafka，它会监听 9092 端口。修改 port 配置参数可以把它设置成其他任意可用的端口。要注意，如果使用 1024 以下的端口，需要使用 root 权限启动Kafka，不过不建议这么做。
3. zookeeper.connect
   用 于 保 存 broker 元 数 据 的 Zookeeper 地 址 是 通 过 zookeeper.connect 来 指 定 的。localhost:2181 表示这个 Zookeeper 是运行在本地的 2181 端口上。该配置参数是用冒号分隔的一组 hostname:port/path 列表，每一部分的含义如下：
   • hostname 是 Zookeeper 服务器的机器名或 IP 地址；
   • port 是 Zookeeper 的客户端连接端口；
   • /path 是可选的 Zookeeper 路径，作为 Kafka 集群的 chroot 环境。如果不指定，默认使用根路径。如果指定的 chroot 路径不存在， broker 会在启动的时候创建它。 

   为什么使用 chroot 路径？
   在 Kafka 集群里使用 chroot 路径是一种最佳实践。 Zookeeper 群组可以共享给其他应用程序，即使还有其他 Kafka 集群存在，也不会产生冲突。最好是在配置文件里指定一组 Zookeeper 服务器，用分号把它们隔开。一旦有一个 Zookeeper 服务器宕机， broker 可以连接到 Zookeeper 群组的另一个节点上。 

4. log.dirs
   Kafka 把所有消息都保存在磁盘上，存放这些日志片段的目录是通过 log.dirs 指定的。它是一组用逗号分隔的本地文件系统路径。如果指定了多个路径，那么 broker 会根据“最少使用”原则，把同一个分区的日志片段保存到同一个路径下。要注意， broker 会往拥有最少数目分区的路径新增分区，而不是往拥有最小磁盘空间的路径新增分区。
5. num.recovery.threads.per.data.dir
   对于如下 3 种情况， Kafka 会使用可配置的线程池来处理日志片段：
   • 服务器正常启动，用于打开每个分区的日志片段；
   • 服务器崩溃后重启，用于检查和截短每个分区的日志片段；
   • 服务器正常关闭，用于关闭日志片段。
   默认情况下，每个日志目录只使用一个线程。因为这些线程只是在服务器启动和关闭时会用到，所以完全可以设置大量的线程来达到并行操作的目的。特别是对于包含大量分区的服务器来说，一旦发生崩溃，在进行恢复时使用并行操作可能会省下数小时的时间。设置此参数时需要注意，所配置的数字对应的是 log.dirs 指定的单个日志目录。也就是说，如果 num.recovery.threads.per.data.dir 被设为 8，并且 log.dir 指定了 3 个路径，那么总共需要 24 个线程。
6. auto.create.topics.enable
   默认情况下， Kafka 会在如下几种情形下自动创建主题：
   • 当一个生产者开始往主题写入消息时；
   • 当一个消费者开始从主题读取消息时；
   • 当任意一个客户端向主题发送元数据请求时。
   很多时候，这些行为都是非预期的。而且，根据 Kafka 协议，如果一个主题不先被创建，根本无法知道它是否已经存在。如果显式地创建主题，不管是手动创建还是通过其他配置系统来创建，都可以把 auto.create.topics.enable 设为 false。 

### Topic默认配置

1. num.partitions
   num.partitions 参数指定了新创建的主题将包含多少个分区。如果启用了主题自动创建功能（该功能默认是启用的），主题分区的个数就是该参数指定的值。该参数的默认值是 1。要注意，我们可以增加主题分区的个数，但不能减少分区的个数。所以，如果要让一个主题的分区个数少于 num.partitions 指定的值，需要手动创建该主题（将在第 9 章讨论）。第 1 章里已经提到， Kafka 集群通过分区对主题进行横向扩展，所以当有新的 broker 加入集群时，可以通过分区个数来实现集群的负载均衡。当然，这并不是说，在存在多个主题的情况下（它们分布在多个 broker 上），为了能让分区分布到所有 broker 上，主题分区的个数必须要大于 broker 的个数。不过，拥有大量消息的主题如果要进行负载分散，就需要大量的分区。
     如何选定分区数量？
   为主题选定分区数量并不是一件可有可无的事情，在进行数量选择时，需要考虑如下几个因素。
     • 主题需要达到多大的吞吐量？例如，是希望每秒钟写入 100KB 还是 1GB ？
     • 从单个分区读取数据的最大吞吐量是多少？每个分区一般都会有一个消费者，如果你知道消费者将数据写入数据库的速度不会超过每秒 50MB，那么你也该知道，从一个分区读取数据的吞吐量不需要超过每秒 50MB。
     • 可以通过类似的方法估算生产者向单个分区写入数据的吞吐量，不过生产者的速度一般比消费者快得多，所以最好为生产者多估算一些吞吐量。
     • 每个 broker 包含的分区个数、可用的磁盘空间和网络带宽。
     • 如果消息是按照不同的键来写入分区的，那么为已有的主题新增分区就会很困难。
     • 单个 broker 对分区个数是有限制的，因为分区越多，占用的内存越多，完
     成首领选举需要的时间也越长。
     很显然，综合考虑以上几个因素，你需要很多分区，但不能太多。如果你估算出主题的吞
     吐量和消费者吞吐量，可以用主题吞吐量除以消费者吞吐量算出分区的个数。也就是说，
     如果每秒钟要从主题上写入和读取 1GB 的数据，并且每个消费者每秒钟可以处理 50MB
     的数据，那么至少需要 20 个分区。 这样就可以让 20 个消费者同时读取这些分区，从而达
     到每秒钟 1GB 的吞吐量。 如果不知道这些信息，那么根据经验，把分区的大小限制在 25GB 以内可以得到比较理想的效果。
2. log.retention.ms
   Kafka 通常根据时间来决定数据可以被保留多久。默认使用 log.retention.hours 参数来配置时间，默认值为 168 小时，也就是一周。除此以外，还有其他两个参数 log.retention.minutes 和 log.retention.ms。这 3 个参数的作用是一样的，都是决定消息多久以后会被删除，不过还是推荐使用 log.retention.ms。如果指定了不止一个参数， Kafka 会优先使用具有最小值的那个参数。根据时间保留数据和最后修改时间根据时间保留数据是通过检查磁盘上日志片段文件的最后修改时间来实现的。一般来说，最后修改时间指的就是日志片段的关闭时间，也就是文件里最后一个消息的时间戳。不过，如果使用管理工具在服务器间移动分区，最后修改时间就不准确了。时间误差可能导致这些分区过多地保留数据。在第9 章讨论分区移动时会提到更多这方面的内容。
3. log.retention.bytes
   另一种方式是通过保留的消息字节数来判断消息是否过期。它的值通过参数 log.retention.bytes 来指定，作用在每一个分区上。也就是说，如果有一个包含 8 个分区的主题，并且 log.retention.bytes 被设为 1GB，那么这个主题最多可以保留 8GB 的数据。所以，当主题的分区个数增加时，整个主题可以保留的数据也随之增加。根据字节大小和时间保留数据如果同时指定了 log.retention.bytes 和 log.retention.ms（或者另一个时
   间参数），只要任意一个条件得到满足，消息就会被删除。例如，假设 log.retention.ms 设置86400000（也就是 1 天）， log.retention.bytes 设置为 1 000 000 000（也就是 1GB），如果消息字节总数在不到一天的时间就超过了 1GB，那么多出来的部分就会被删除。相反，如果消息字节总数小于1GB，那么一天之后这些消息也会被删除，尽管分区的数据总量小于 1GB。
4. log.segment.bytes
   以上的设置都作用在日志片段上，而不是作用在单个消息上。当消息到达 broker 时，它
   们被追加到分区的当前日志片段上。当日志片段大小达到 log.segment.bytes 指定的上限
   （默认是 1GB）时，当前日志片段就会被关闭，一个新的日志片段被打开。如果一个日志
   片段被关闭，就开始等待过期。这个参数的值越小，就会越频繁地关闭和分配新文件，从
   而降低磁盘写入的整体效率。
   如果主题的消息量不大，那么如何调整这个参数的大小就变得尤为重要。例如，如果一个
   主题每天只接收 100MB 的消息，而 log.segment.bytes 使用默认设置，那么需要 10 天时 间才能填满一个日志片段。因为在日志片段被关闭之前消息是不会过期的，所以如果 log.
   retention.ms 被设为 604 800 000（也就是 1 周），那么日志片段最多需要 17 天才会过期。
   这是因为关闭日志片段需要 10 天的时间，而根据配置的过期时间，还需要再保留 7 天时
   间（要等到日志片段里的最后一个消息过期才能被删除）。
   使用时间戳获取偏移量
   日志片段的大小会影响使用时间戳获取偏移量。在使用时间戳获取日志偏移
   量时， Kafka 会检查分区里最后修改时间大于指定时间戳的日志片段（已经
   被关闭的），该日志片段的前一个文件的最后修改时间小于指定时间戳。然
   后， Kafka 返回该日志片段（也就是文件名）开头的偏移量。对于使用时间
   戳获取偏移量的操作来说，日志片段越小，结果越准确。
5. log.segment.ms
   另一个可以控制日志片段关闭时间的参数是 log.segment.ms，它指定了多长时间之后日
   志片段会被关闭。就像 log.retention.bytes 和 log.retention.ms 这两个参数一样， log.
   segment.bytes 和 log.retention.ms 这两个参数之间也不存在互斥问题。日志片段会在大
   小或时间达到上限时被关闭，就看哪个条件先得到满足。默认情况下， log.segment.ms 没
   有设定值，所以只根据大小来关闭日志片段。
   基于时间的日志片段对磁盘性能的影响
   在使用基于时间的日志片段时，要着重考虑并行关闭多个日志片段对磁盘性
   能的影响。如果多个分区的日志片段永远不能达到大小的上限，就会发生这
   种情况，因为 broker 在启动之后就开始计算日志片段的过期时间，对于那些
   数据量小的分区来说，日志片段的关闭操作总是同时发生。
6. message.max.bytes
   broker 通过设置 message.max.bytes 参数来限制单个消息的大小，默认值是 1 000 000，也
   就是 1MB。如果生产者尝试发送的消息超过这个大小，不仅消息不会被接收，还会收到
   broker 返回的错误信息。跟其他与字节相关的配置参数一样，该参数指的是压缩后的消息
   大小，也就是说，只要压缩后的消息小于 message.max.bytes 指定的值，消息的实际大小
   可以远大于这个值。
   这个值对性能有显著的影响。值越大，那么负责处理网络连接和请求的线程就需要花越多
   的时间来处理这些请求。它还会增加磁盘写入块的大小，从而影响 IO 吞吐量。
   在服务端和客户端之间协调消息大小的配置
   消费者客户端设置的 fetch.message.max.bytes 必须与服务器端设置的消息
   大小进行协调。如果这个值比 message.max.bytes 小，那么消费者就无法读
   取比较大的消息，导致出现消费者被阻塞的情况。在为集群里的 broker 配置
   replica.fetch.max.bytes 参数时，也遵循同样的原则。 

## 注意事项



### 垃圾回收器

正常情况下， G1 只需要很少的配置就能完成这些工作。以下是 G1 的两个调整参数。

MaxGCPauseMillis：
	该参数指定每次垃圾回收默认的停顿时间。该值不是固定的， G1 可以根据需要使用更长的时间。它的默认值是 200ms。也就是说， G1 会决定垃圾回收的频率以及每一轮需要回收多少个区域，这样算下来，每一轮垃圾回收大概需要 200ms 的时间。

InitiatingHeapOccupancyPercent：
	该参数指定了在 G1 启动新一轮垃圾回收之前可以使用的堆内存百分比，默认值是 45。也就是说，在堆内存的使用率达到 45% 之前， G1 不会启动垃圾回收。这个百分比包括新生代和老年代的内存。

Kafka 对堆内存的使用率非常高，容易产生垃圾对象，所以可以把这些值设得小一些。如果一台服务器有 64GB 内存，并且使用 5GB 堆内存来运行 Kafka，那么可以参考以下的配置： MaxGCPauseMillis 可以设为 20ms； InitiatingHeapOccupancyPercent 可以设为 35，这样可以让垃圾回收比默认的要早一些启动。 

### 数据中心布局



### 共享zookeeper

Kafka 使用 Zookeeper 来保存 broker、主题和分区的元数据信息。对于一个包含多个节点的Zookeeper 群组来说， Kafka 集群的这些流量并不算多，那些写操作只是用于构造消费者群组或集群本身。实际上，在很多部署环境里，会让多个 Kafka 集群共享一个 Zookeeper 群组（每个集群使用一个 chroot 路径）。 

虽 然 多 个 Kafka 集 群 可 以 共 享 一 个 Zookeeper 群 组， 但 如 果 有 可 能 的 话， 不 建 议把 Zookeeper 共享给其他应用程序。 Kafka 对 Zookeeper 的延迟和超时比较敏感，与Zookeeper 群组之间的一个通信异常就可能导致 Kafka 服务器出现无法预测的行为。这样很容易让多个 broker 同时离线，如果它们与 Zookeeper 之间断开连接，也会导致分区离线。这也会给集群控制器带来压力，在服务器离线一段时间之后，当控制器尝试关闭一个
服务器时，会表现出一些细小的错误。其他的应用程序因重度使用或进行不恰当的操作给Zookeeper 群组带来压力，所以最好让它们使用自己的 Zookeeper 群组 。

# Kafka和zookeeper

**生产者-producer**，负责生成消息并发送到Kafka服务器。

**消费者-consumer**，消费Kafka服务器上的消息。

**主题-Topic**，用户自定义配置，建立生产者和消费者之间的订阅关系：生产者发送消息到指定topic，消费者从这个topic下消费消息。

**消息分区-partition**，一个topic下面可以分为多个分区，如‘kafka-test’这个topic下面，可以分为10个分区，分别由两台服务器提供，那么通常可以配置每台服务器提供5个分区。消息分区机制和分区的数量与消费者的负载均衡有很大关系。

**Broker-Kafka的服务器**，用于存储消息。

**消费者分组-Group**，用于归类同类消费者。

**Offset**  消息存储再Kafka的broker上，消费者拉取消息数据的过程中需要指定消息在文件中的偏移量。

## Broker注册

zookeeper上面会有一个专门用于broker服务器列表记录的节点。

每个broker服务器启动时，都回到zookeeper上注册一个临时节点，如/broker/ids/0

创建完broker节点后，每个broker会将自己的ip地址和端口信息写入到该节点。

因为是临时节点，所有可以通过监控broker节点的变化来动态表征broker服务器的可用性。

## Topic注册

Kafka中，会将一个topic消息分成多个分区，并将其分布到多个broker，而这些分区信息以及与broker的对应关系，也由zookeeper维护。

例如：/brokers/topics/login，表示登录的topic。

broker启动后，会到对应的topic节点下注册自己的brokerID，并写入针对该topic的分区总数。

例如：/brokers/topics/login3 -> 2，表示broker ID为3的broker服务器，为login这个topic，提供了2个分区进行消息存储。

## 负载均衡

### 生产者负载均衡

#### 四层负载均衡

传统的四层负载均衡，一般就是通过ip+端口，来唯一确定一个相关联的生产者-broker。

优点：简单，不需要引入第三方系统，只需要和一个broker服务器建立tcp连接。

缺点：无法真正的负载均衡，每个生产者生成的消息量和每个消息所需要的存储量不一样，可能导致不同的broker收到的消息总数非常不均匀。生产者无法实时感知到broker的新增和删除。

#### zookeeper负载均衡

Kafka中，客户端使用了基于zookeeper的负载均衡策略，来解决生产者的负载均衡问题。

Kafka的生产者会对zookeeper上面的事件注册watcher监听，实现动态的负载均衡机制：

broker增删

topic增删

broker和topic关联关系变化

允许开发人员控制生产者根据一定规则来进行数据分区，而不仅仅是随机-“语义分区”。

### 消费者负载均衡

对于每个消费者分组，Kafka都会为其分配一个全局唯一的Group ID。

同时，Kafka也为每个消费者分配一个Consumer ID，通常采用"Hostname:UUID"的形式来表示。

Kafka规定每个消息分区有且只能有一个消费者进行消息的消费，因此需要zookeeper记录下消息分区和消费者之间的对应关系。

例如：/consumers/[group_id]/owners/[topic]/[broker_id-partition_id]，就是一个消息分区的标识，节点内容就是消费该分区上消息的消费者的Consummer ID。

#### 消费进度offset记录

消费者进行消费的时候，需要定时将消费进度offset记录到zookeeper，以便在消费者重启或者其他消费者重新接管该消息分区后，能够从之前的进度开始继续进行消费。

/consumers/[group_id]/offsets/[topic]/[broker_id-partition_id]节点内容就是offset值

#### 消费者注册

1.注册的到消费者分组。

消费者服务启动的时候，都回到zookeeper指定节点创建一个属于自己的消费者节点，例如：/consumers/[group_id]/ids/[consumer_id]

2.对消费者组中消费者变化注册监听。

消费者需要关注所属消费者分组中消费者服务器的变化情况，注册/consumers/[group_id]/ids节点注册子节点的watcher监听。一旦发现消费者新增或者减少，就会触发消费者的负载均衡。

3.对Broker服务器的变化注册监听

对/broker/ids的子节点进行监听，如果发现broker服务器列表发生变化，那就根据具体情况来决定是否需要进行消费者的负载均衡。

4.进行消费者负载均衡

让同一个topic下不同分区的消息尽量均衡的被多个消费者消费。



#### 负载均衡算法

[Kafka的Consumer负载均衡算法](https://juejin.im/post/5baca032e51d450e735e74af)



# 生产者



![1585127536381](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%88%86%E5%B8%83%E5%BC%8F/assets/1585127536381.png)

## 使用

```java
public class Producer extends Thread {
    private final KafkaProducer<Integer, String> producer;
    private final String topic;
    private final Boolean isAsync;

    public Producer(String topic, Boolean isAsync) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("client.id", "DemoProducer");
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        producer = new KafkaProducer<>(props);
        this.topic = topic;
        this.isAsync = isAsync;
    }

    public void run() {
        int messageNo = 1;
        while (true) {
            String messageStr = "Message_" + messageNo;
            long startTime = System.currentTimeMillis();
            if (isAsync) { // Send 异步
                producer.send(new ProducerRecord<>(topic,
                    messageNo,
                    messageStr), new DemoCallBack(startTime, messageNo, messageStr));
            } else { // Send 同步
                try {
                    producer.send(new ProducerRecord<>(topic,
                        messageNo,
                        messageStr)).get();
                    System.out.println("Sent message: (" + messageNo + ", " + messageStr + ")");
                } catch (InterruptedException | ExecutionException e) {
                    e.printStackTrace();
                }
            }
            ++messageNo;
        }
    }
}

class DemoCallBack implements Callback {

    private final long startTime;
    private final int key;
    private final String message;

    public DemoCallBack(long startTime, int key, String message) {
        this.startTime = startTime;
        this.key = key;
        this.message = message;
    }

    public void onCompletion(RecordMetadata metadata, Exception exception) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        if (metadata != null) {
            System.out.println(
                "message(" + key + ", " + message + ") sent to partition(" + metadata.partition() +
                    "), " +
                    "offset(" + metadata.offset() + ") in " + elapsedTime + " ms");
        } else {
            exception.printStackTrace();
        }
    }
}
```

发送消息主要有以下 3 种方式。
发送并忘记（fire-and-forget）
	我们把消息发送给服务器，但并不关心它是否正常到达。大多数情况下，消息会正常到达，因为 Kafka 是高可用的，而且生产者会自动尝试重发。不过，使用这种方式有时候也会丢失一些消息。
同步发送
	我们使用 send() 方法发送消息，它会返回一个 Future 对象，调用 get() 方法进行等待，就可以知道消息是否发送成功。
异步发送
	我们调用 send() 方法，并指定一个回调函数，服务器在返回响应时调用该函数。 

## 配置

1. acks

acks 参数指定了必须要有多少个分区副本收到消息，生产者才会认为消息写入是成功的。

这个参数对消息丢失的可能性有重要影响。 该参数有如下选项。
• 如果 acks=0， 生产者在成功写入消息之前不会等待任何来自服务器的响应。也就是说， 如果当中出现了问题 ， 导致服务器没有收到消息，那么生产者就无从得知，消息也就丢 失了。不过，因为生产者不需要等待服务器的响应，所以它可以以网络能够支持的最大 速度发送消息，从而达到很高的吞吐量。

• 如果 acks=1，只要集群的首领节点收到消息，生产者就会收到 一个来自服务器的成功 响应。如果消息无撞到达首领节点(比如首领节点崩愤，新的首领还没有被选举出来)， 生产者会收到一个错误响应，为了避免数据丢失，生产者会重发消息。不过，如果一个 没有收到消息的节点成为新首领，消息还是会丢失。这个时候的吞吐量取决于使用的是 同步发送还是异步发送。如果让发送客户端等待服务器的响应(通过调用 Future对象 的 get()方法)，显然会增加延迟(在网络上传输一个来回的延迟)。如果客户端使用异步回调，延迟问题就可以得到缓解，不过吞吐量还是会受发送中消息数量的限制(比如，生 产者在收到服务器响应之前可以发送多少个消息)。

• 如果 acks=all，只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应。这种模式是最安全的，它可以保证不止一个服务器收到消息，就算有服务器发生崩溃，整个集群仍然可以运行(第 5 章将讨论更多的细节)。不过，它的延迟比 acks=1时更高，因为我们要等待不只一个服务器节点接收消息。

2. buffer.memory

该参数用来设置生产者内存缓冲区的大小，生产者用它缓冲要发送到服务器的消息。如果 应用程序发送消息的速度超过发送到服务器的速度，会导致生产者空间不足。这个时候， send()方法调用要么被阻塞，要么抛出异常，取决于如何设置 block.on.buffe.full 参数 (在0.9.0.0版本里被替换成了max.block.ms，表示在抛出异常之前可以阻塞一段时间)。

3. compression.type

默认情况下，消息发送时不会被压缩。该参数可以设置为 snappy、 gzip 或 lz4，它指定了消息被发送给 broker之前使用哪一种压缩算法进行压缩。 snappy 压缩算怯由 Google巳发明， 它占用较少 的 CPU，却能提供较好的性能和相当可观的压缩比，如果比较关注性能和网络带宽，可以使用这种算法。 gzip压缩算法一般会占用较多的 CPU，但会提供更高的压缩比，所以如果网络带宽比较有限，可以使用这种算法。使用压缩可以降低网络传输开销和存储开销，而这往往是向 Kafka发送消息的瓶颈所在。

4. retries

生产者从服务器收到的错误有可能是临时性的错误(比如分区找不到首领)。在这种情况下， retries参数的值决定了生产者可以重发消息的次数，如果达到这个次数，生产者会放弃重试并返回错误。默认情况下，生产者会在每次重试之间等待 1OOms，不过可以通过 retries.backoff.ms 参数来改变这个时间间隔。建议在设置重试次数和重试时间间隔之前， 先测试一下恢复一个崩溃节点需要多少时间(比如所有分区选举出首领需要多长时间)， 让总的重试时间比 Kafka集群从崩溃中恢复的时间长，否则生产者会过早地放弃重试。不过有些错误不是临时性错误，没办怯通过重试来解决(比如“悄息太大”错误)。一般情 况下，因为生产者会自动进行重试，所以就没必要在代码逻辑里处理那些可重试的错误。 你只需要处理那些不可重试的错误或重试次数超出上限的情况。

5. batch.size

当有多个消息需要被发送到同一个分区时，生产者会把它们放在放一个批次里。该参数指定了一个批次可以使用的内存大小，按照字节数计算(而不是消息个数)。当批次被填满，批次里的所有消息会被发送出去。不过生产者井不一定都会等到批次被填满才发送，半捕 的批次，甚至只包含一个消息的批次也有可能被发送。所以就算把批次大小设置得很大， 也不会造成延迟，只是会占用更多的内存而已。但如果设置得太小，因为生产者需要更频繁地发送消息，会增加一些额外的开销。

6. linger.ms

该参数指定了生产者在发送批次之前等待更多消息加入批次的时间。 KafkaProducer 会在批次填满或 linger.ms达到上限时把批次发送出去。默认情况下，只要有可用的线程， 生产者就会把消息发送出去，就算批次里只有一个消息。把 linger.ms设置成比0大的数， 让生产者在发送批次之前等待一会儿，使更多的消息加入到这个批次 。虽然这样会增加延迟，但也会提升吞吐量(因为一次性发送更多的消息，每个消息的开销就变小了)。

7. client.id

该参数可以是任意的字符串，服务器会用它来识别消息的来源，还可以用在日志和配额指标里。

8. max.in.flight.requests.per.connection

该参数指定了生产者在收到服务器晌应之前可以发送多少个消息。它的值越高，就会占用越多的内存，不过也会提升吞吐量。 把它设为 1 可以保证消息是按照发送的顺序写入服务器的，即使发生了重试。

9. timeout.ms、 request.timeout.ms 和 metadata.fetch.timeout.ms

request.timeout.ms指定了生产者在发送数据时等待服务器返回响应的时间，metadata.fetch.timeout.ms指定了生产者在获取元数据(比如目标分区的首领是谁)时等待服务器返回响应的时间。如果等待响应超时，那么生产者要么重试发送数据，要么返回 一个错误 (抛出异常或执行回调)。timeout.ms 指定了 broker 等待同步副本返回消息确认的时间，与 asks 的配置相匹配一一如果在指定时间内没有收到同步副本的确认，那么 broker就会返回 一个错误 。

10. max.block.ms

该参数指定了在调用 send() 方法或使用 parttitionFor() 方能获取元数据时生产者的阻塞 时间。当生产者的发送缓冲区已捕，或者没有可用的元数据时，这些方屈就会阻塞。在阻塞时间达到 max.block.ms时，生产者会抛出超时异常。

11 . max.request.size

该参数用于控制生产者发送的请求大小。它可以指能发送的单个消息的最大值，也可以指单个请求里所有消息总的大小。例如，假设这个值为 1MB，那么可以发送的单个最大消息为 1MB，或者生产者可以在单个请求里发送一个批次，该批次包含了 1000个消息，每个消息大小为 1KB 。另外， broker对可接收的消息最大值也有自己的限制( message.max.bytes)，所以两边的配置最好可以匹配，避免生产者发送的消息被 broker拒绝 。

12. receive.buffer.bytes 和 send.buffer.bytes

这两个参数分别指定了 TCP socket接收和发送数据包的缓冲区大小 。 如果它们被设为 -1 , 就使用操作系统的默认值。如果生产者或消费者与 broker处于不同的数据中心，那么可以适当增大这些值，因为跨数据中心的网络一般都有比较高的延迟和比较低的带宽。

```
顺序保证
Kafka可以保证同一个分区里的消息是有序的。也就是说，如果生产者按照一定的顺序发送消息， broker就会按照这个顺序把它们写入分区，消费者也会按照同样的顺序读取它们。在某些情况下 ， 顺序是非常重要的。如果把retries 设为非零整数，同时把 max.in.flight.requests.per.connection 设为比 1大的数，那么，如果第一个批次消息写入失败，而第二个批次写入成功， broker会重试写入第一个批次。如果此时第一个批次也写入成功，那 么两个批次的顺序就反过来了。

一般来说，如果某些场景要求消息是有序的，那么消息是否写入成功也是 很关键的，所以不建议把顺序是非常重要的。如果把retries 设为 0。可以把 max.in.flight.requests.per.connection设为 1，这样在生产者尝试发送第一批消息时，就不会有其他的消息发送给 broker。不过这样会严重影响生产者的吞吐量 ，所以只有在 对消息的顺序有严格要求的情况下才能这么做。
```

## 序列化



### IntegerDeserializer



### StringDeserializer



### KafkaAvroSerializer



## 分区

​	在之前的例子里， ProducerRecord 对象包含了目标主题、键和值。 Kafka 的消息是一个个键值对， ProducerRecord 对象可以只包含目标主题和值，键可以设置为默认的 null，不过大多数应用程序会用到键。

键有两个用途：可以作为消息的附加信息，也可以用来决定消息该被写到主题的哪个分区。

拥有相同键的消息将被写到同一个分区。

也就是说，如果一个进程只从一个主题的分区读取数据，那么具有相同键的所有记录都会被该进程读取。 

1.如果**键值为 null**，并且使用了**默认的分区器**，那么记录将被随机地发送到主题内各个可用的分区上。分区器使用轮询（Round Robin）算法将消息均衡地分布到各个分区上。

2.如果**键不为空**，并且使用了**默认的分区器**，那么 Kafka 会对键进行散列（使用 Kafka 自己的散列算法，即使升级 Java 版本，散列值也不会发生变化），然后根据散列值把消息映射到特定的分区上。这里的关键之处在于，同一个键总是被映射到同一个分区上，所以在进行映射时，我们会使用主题所有的分区，而不仅仅是可用的分区。这也意味着，如果写入数据的分区是不可用的，那么就会发生错误。但这种情况很少发生。

只有在不改变主题分区数量的情况下，键与分区之间的映射才能保持不变。举个例子，在分区数量保持不变的情况下，可以保证用户 045189 的记录总是被写到分区 34。在从分区读取数据时，可以进行各种优化。不过，一旦主题增加了新的分区，这些就无法保证了——旧数据仍然留在分区 34，但新的记录可能被写到其他分区上。如果要使用键来映射分区，那么最好在创建主题的时候就把分区规划好（第 2 章介绍了如何确定合适的分区数量），而且永远不要增加新分区。 

3.**自定义分区策略**，我们已经讨论了默认分区器的特点，它是使用次数最多的分区器。不过，除了散列分区之外，有时候也需要对数据进行不一样的分区。假设你是一个 B2B 供应商，你有一个大客户，它是手持设备 Banana 的制造商。 Banana 占据了你整体业务 10% 的份额。如果使用默认的散列分区算法， Banana 的账号记录将和其他账号记录一起被分配给相同的分区，导致这个分区比其他分区要大一些。服务器可能因此出现存储空间不足、处理缓慢等问题。我们需要给 Banana 分配单独的分区，然后使用散列分区算法处理其他账号。 

```java
public class BananaPartitioner implements Partitioner {
    public void configure(Map<String, ?> configs) {} 
        
    public int partition(String topic, Object key, byte[] keyBytes,
    Object value, byte[] valueBytes,
    Cluster cluster) {
        List<PartitionInfo> partitions =
        cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if ((keyBytes == null) || (!(key instanceOf String))) 
        throw new InvalidRecordException("We expect all messages
        to have customer name as key")
        if (((String) key).equals("Banana"))
        return numPartitions; // Banana总是被分配到最后一个分区
        // 其他记录被散列到其他分区
        return (Math.abs(Utils.murmur2(keyBytes)) % (numPartitions - 1))
    }
                                     
    public void close() {}
}
```

# 消费者

消费者有线程安全问题，应该一个线程使用一个消费者。

Kafka 消费者从属于消费者群组。一个群组里的消费者订阅的是同一个主题，每个消费者接收主题一部分分区的消息。 

往群组里增加消费者是横向伸缩消费能力的主要方式。 Kafka 消费者经常会做一些高延迟的操作，比如把数据写到数据库或 HDFS，或者使用数据进行比较耗时的计算。在这些情况下，单个消费者无法跟上数据生成的速度，所以可以增加更多的消费者，让它们分担负载，每个消费者只处理部分分区的消息，这就是横向伸缩的主要手段。我们有必要为主题创建大量的分区，在负载增长时可以加入更多的消费者。不过要注意，不要让消费者的数量超过主题分区的数量，多余的消费者只会被闲置。 

除了通过增加消费者来横向伸缩单个应用程序外，还经常出现多个应用程序从同一个主题读取数据的情况。实际上， Kafka 设计的主要目标之一，就是要让 Kafka 主题里的数据能够满足企业各种应用场景的需求。在这些场景里，每个应用程序可以获取到所有的消息，而不只是其中的一部分。只要保证每个应用程序有自己的消费者群组，就可以让它们获取到主题所有的消息。不同于传统的消息系统，横向伸缩 Kafka 消费者和消费者群组并不会对性能造成负面影响。 

![1585902307188](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%88%86%E5%B8%83%E5%BC%8F/assets/1585902307188.png)

## 使用

```java
public class Consumer extends ShutdownableThread {
    private final KafkaConsumer<Integer, String> consumer;
    private final String topic;

    public Consumer(String topic) {
        super("KafkaConsumerExample", false);
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "DemoConsumer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "30000");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.IntegerDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");

        consumer = new KafkaConsumer<>(props);
        this.topic = topic;
    }

    @Override
    public void doWork() {
        //订阅topic
        consumer.subscribe(Collections.singletonList(this.topic));
        ConsumerRecords<Integer, String> records = consumer.poll(1000);
        for (ConsumerRecord<Integer, String> record : records) {
            System.out.println("Received message: (" + record.key() + ", " + record.value() + ") at offset " + record.offset());
        }
    }

    @Override
    public String name() {
        return null;
    }

    @Override
    public boolean isInterruptible() {
        return false;
    }
}
```

## 配置

到目前为止，我们学习了如何使用消费者 API，不过只介绍了几个配置属’性一一如bootstrap.servers、 key.deserializer、 value.deserializer、group.id。 Kafka的文档列出了所有与消费者相关的配置说明。大部分参数都有合理的默认值，一般不需要修改它们，不过有一些参数与消费 者的性能和可用性有很大关系。接下来介绍这些重要的属性。

1. fetch.min.bytes

该属性指定了消费者从服务器获取记录的最小字节数。 broker 在收到消费者的数据请求时， 如果可用的数据量小于 fetch.min.bytes指定的大小，那么它会等到有足够的可用数据时才把它返回给消费者。这样可以降低消费者和 broker 的工作负载，因为它们在主题不是很活跃的时候(或者一天里的低谷时段)就不需要来来回回地处理消息。如果没有很多可用数据，但消费者的 CPU 使用率却很高，那么就需要把该属性的值设得比默认值大。如果消费者的数量比较多，把该属性的值设置得大一点可以降低 broker 的工作负载。

2. fetch.max.wait.ms

我们通过 fetch.min.bytes 告诉 Kafka，等到有足够的数据时才把它返回给消费者。而 fetch.max.wait.ms则用于指定 broker的等待时间，默认是 500ms。如果没有足够的数据流入 Kafka，消费者获取最小数据量的要求就得不到满足，最终导致500ms的延迟。 如果要降低潜在的延迟(为了满足 SLA)，可以把该参数值设置得小一些。如果 fetch.max.wait.ms被设 为 100ms，并且 fetch.min.bytes 被设为 1MB，那么 Kafka在收到消费者的请求后，要么返 回 1MB 数据，要么在 100ms 后返回所有可用的数据 ， 就看哪个条件先得到满足。

3. max.parition.fetch.bytes

该属性指定了服务器从每个分区里返回给消费者的最大字节数。它的默认值是 1MB，也 就是说， KafkaConsumer.poll() 方法从每个分区里返回的记录最多不超过 max.parition.fetch.bytes 指定的字节。如果一个主题有 20个分区和 5 个消费者，那么每个消费者需要至少 4MB 的可用内存来接收记录。在为消费者分配内存时，可以给它们多分配一些，因 为如果群组里有消费者发生崩溃，剩下的消费者需要处理更多的分区。 max.parition.fetch.bytes 的值必须比 broker能够接收的最大消息的字节数(通过 max.message.size属 性配置 )大， 否则消费者可能无法读取这些消息，导致消费者一直挂起重试。在设置该属性时，另一个需要考虑的因素是消费者处理数据的时间。 消费者需要频繁调用 poll() 方法来避免会话过期和发生分区再均衡，如果单次调用 poll() 返回的数据太多，消费者需要更多的时间来处理，可能无法及时进行下一个轮询来避免会话过期。如果出现这种情况， 可以把 max.parition.fetch.bytes 值改小 ，或者延长会话过期时间。

4. session.timeout.ms

该属性指定了消费者在被认为死亡之前可以与服务器断开连接的时间，默认是 3s。如果消费者没有在 session.timeout.ms 指定的时间内发送心跳给群组协调器，就被认为已经死亡，协调器就会触发再均衡，把它的分区分配给群组里的其他消费者 。该属性与 heartbeat.interval.ms紧密相关。heartbeat.interval.ms 指定了poll()方法向协调器 发送心跳的频 率， session.timeout.ms 则指定了消费者可以多久不发送心跳。所以， 一般需要同时修改这两个属性， heartbeat.interval.ms 必须比 session.timeout.ms 小， 一般是 session.timeout.ms 的三分之一。如果 session.timeout.ms 是 3s，那么 heartbeat.interval.ms 应该是 ls。 把 session.timeout.ms 值设 得比默认值小，可以更快地检测和恢 复崩溃的节点，不过长时间的轮询或垃圾收集可能导致非预期的再均衡。把该属性的值设置得大一些，可以减少意外的再均衡 ，不过检测节点崩溃需要更长的时间。

5. auto.offset.reset

该属性指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下(因消费者长时间失效，包含偏移量的记录已经过时井被删除)该作何处理。它的默认值是latest， 意 思是说，在偏移量无效的情况下，消费者将从最新的记录开始读取数据(在消费者 启动之 后生成的记录)。另一个值是 earliest，意思是说，在偏移量无效的情况下，消费者将从 起始位置读取分区的记录。

6. enable.auto.commit

我们稍后将介绍 几种 不同的提交偏移量的方式。该属性指定了消费者是否自动提交偏移量，默认值是 true。为了尽量避免出现重复数据和数据丢失，可以把它设为 false，由自己控制何时提交偏移量。如果把它设为 true，还可以通过配置 auto.commit.interval.mls 属性来控制提交的频率。

7. partition.assignment.strategy

我们知道，分区会被分配给群组里的消费者。 PartitionAssignor 根据给定的消费者和主题，决定哪些分区应该被分配给哪个消费者。 Kafka 有两个默认的分配策略 。

```
Range
```

该策略会把主题的若干个连续的分区分配给消费者。假设悄费者 C1 和消费者 C2 同时 订阅了主题 T1 和主题 T2，井且每个主题有 3 个分区。那么消费者 C1 有可能分配到这 两个主题的分区 0 和 分区 1，而消费者 C2 分配到这两个主题 的分区 2。因为每个主题 拥有奇数个分区，而分配是在主题内独立完成的，第一个消费者最后分配到比第二个消费者更多的分区。只要使用了 Range策略，而且分区数量无法被消费者数量整除，就会出现这种情况。

```
RoundRobin
```

该策略把主题的所有分区逐个分配给消费者。如果使用 RoundRobin 策略来给消费者 C1 和消费者 C2分配分区，那么消费者 C1 将分到主题 T1 的分区 0和分区 2以及主题 T2 的分区 1，消费者 C2 将分配到主题 T1 的分区 l 以及主题T2 的分区 0和分区 2。一般 来说，如果所有消费者都订阅相同的主题(这种情况很常见), RoundRobin策略会给所 有消费者分配相同数量 的分区(或最多就差一个分区)。

可以通过设置 partition.assignment.strategy 来选择分区策略。默认使用的是 org. apache.kafka.clients.consumer.RangeAssignor， 这个类实现了 Range策略，不过也可以 把它改成 org.apache.kafka.clients.consumer.RoundRobinAssignor。我们还可以使用自定 义策略，在这种情况下 ， partition.assignment.strategy 属性的值就是自定义类的名字。

8. client.id

该属性可以是任意字符串 ， broker用它来标识从客户端发送过来的消息，通常被用在日志、度量指标和配额里。

9. max.poll.records

该属性用于控制单次调用 call() 方法能够返回的记录数量，可以帮你控制在轮询里需要处理的数据量。

10. receive.buffer.bytes 和 send.buffer.bytes

socket 在读写数据时用到的 TCP 缓冲区也可以设置大小。如果它们被设为-1，就使用操作系统的默认值。如果生产者或消费者与 broker处于不同的数据中心内，可以适当增大这些值，因为跨数据中心的网络一般都有 比较高的延迟和比较低的带宽 。



## 提交和偏移量

​	每次调用 poll() 方法，它总是返回由生产者写入 Kafka 但还没有被消费者读取过的记录，我们因此可以追踪到哪些记录是被群组里的哪个消费者读取的。之前已经讨论过， Kafka不会像其他 JMS 队列那样需要得到消费者的确认，这是 Kafka 的一个独特之处。相反，消费者可以使用 Kafka 来追踪消息在分区里的位置（偏移量）。我们把更新分区当前位置的操作叫作提交。

​	那么消费者是如何提交偏移量的呢？消费者往一个叫作 _consumer_offset 的特殊主题发送消息，消息里包含每个分区的偏移量。如果消费者一直处于运行状态，那么偏移量就没有什么用处。不过，如果消费者发生崩溃或者有新的消费者加入群组，就会触发**再均衡**，完成再均衡之后，每个消费者可能分配到新的分区，而不是之前处理的那个。为了能够继续之前的工作，消费者需要读取每个分区最后一次提交的偏移量，然后从偏移量指定的地方继续处理。 

如果提交的偏移量小于客户端处理的最后一个消息的偏移量，那么处于两个偏移量之间的消息就会被重复处理。 

![1585194971336](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%88%86%E5%B8%83%E5%BC%8F/assets/1585194971336.png)

如果提交的偏移量大于客户端处理的最后一个消息的偏移量，那么处于两个偏移量之间的消息将会丢失。 

![1585194986013](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E5%88%86%E5%B8%83%E5%BC%8F/assets/1585194986013.png)



### 自动提交

最简单的提交方式是让消费者自动提交偏移量。如果 enable.auto.commit 被设为 true，那么每过 5s，消费者会自动把从 poll() 方法接收到的最大偏移量提交上去。提交时间间隔由 auto.commit.interval.ms 控制，默认值是 5s。与消费者里的其他东西一样，自动提交也是在轮询里进行的。消费者每次在进行轮询时会检查是否该提交偏移量了，如果是，那么就会提交从上一次轮询返回的偏移量。 

假设我们仍然使用默认的 5s 提交时间间隔，在最近一次提交之后的 3s 发生了再均衡，再均衡之后，消费者从最后一次提交的偏移量位置开始读取消息。这个时候偏移量已经落后了 3s，所以在这 3s 内到达的消息会被**重复处理**。可以通过修改提交时间间隔来更频繁地提交偏移量，减小可能出现重复消息的时间窗，不过这种情况是无法完全避免的。 

### 提交当前偏移量

把 auto.commit.offset 设为 false，让应用程序决定何时提交偏移量。使用 commitSync()提交偏移量最简单也最可靠。这个 API 会提交由 poll() 方法返回的最新偏移量，提交成功后马上返回，如果提交失败就抛出异常。

要记住， commitSync() 将会提交由 poll() 返回的最新偏移量，所以在处理完所有记录后要确保调用了 commitSync()，否则还是会有丢失消息的风险。

如果发生了再均衡，从最近一批消息到发生再均衡之间的所有消息都将被重复处理。 

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records){
        System.out.printf("topic = %s, partition = %s, offset = %d, customer = %s, country = %s\n",record.topic(), record.partition(),
        record.offset(), record.key(), record.value()); 
    }
    try {
    	consumer.commitSync(); 
    } catch (CommitFailedException e) {
    	log.error("commit failed", e) 
    }
} 

```

### 异步提交

手动提交有一个不足之处，在 broker 对提交请求作出回应之前，应用程序会一直阻塞，这样会限制应用程序的吞吐量。我们可以通过降低提交频率来提升吞吐量，但如果发生了再均衡，会增加重复消息的数量。 

在成功提交或碰到无法恢复的错误之前， commitSync() 会一直重试，但是 commitAsync()不会，这也是 commitAsync() 不好的一个地方。

它之所以不进行重试，是因为在它收到服务器响应的时候，可能有一个更大的偏移量已经提交成功。假设我们发出一个请求用于提交偏移量 2000，这个时候发生了短暂的通信问题，服务器收不到请求，自然也不会作出任何响应。与此同时，我们处理了另外一批消息，并成功提交了偏移量 3000。如果
commitAsync() 重新尝试提交偏移量 2000，它有可能在偏移量 3000 之后提交成功。这个时候如果发生再均衡，就会出现重复消息。 

```JAVA
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records){
        System.out.printf("topic = %s, partition = %s,offset = %d, customer = %s, country = %s\n",record.topic(), record.partition(), record.offset(),
        record.key(), record.value());
    }
    consumer.commitAsync(); 
} 

```

我们之所以提到这个问题的复杂性和提交顺序的重要性，是因为 commitAsync() 也支持回调，在 broker 作出响应时会执行回调。回调经常被用于记录提交错误或生成度量指标，不过如果你要用它来进行重试，一定要注意提交的顺序。 

```JAVA
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %s,
        offset = %d, customer = %s, country = %s\n",
        record.topic(), record.partition(), record.offset(),
        record.key(), record.value());
    }
    consumer.commitAsync(new OffsetCommitCallback() {
      public void onComplete(Map<TopicPartition,OffsetAndMetadata> offsets, Exception e){
          //发送提交请求然后继续做其他事情，如果提交失败，错误信息和偏移量会被记录下来。
          if (e != null) log.error("Commit failed for offsets {}", offsets, e);
      }}); 
} 

```

### 同步和异步组合提交

一般情况下，针对偶尔出现的提交失败，不进行重试不会有太大问题，因为如果提交失败是因为临时问题导致的，那么后续的提交总会有成功的。但如果这是发生在关闭消费者或再均衡前的最后一次提交，就要确保能够提交成功。
因此，在消费者关闭前一般会组合使用 commitAsync() 和 commitSync()。 

```JAVA
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records) {
            System.out.println("topic = %s, partition = %s, offset = %d,
            customer = %s, country = %s\n",
            record.topic(), record.partition(),
            record.offset(), record.key(), record.value());
        }
        //先异步提交，允许偶尔提交失败
        consumer.commitAsync(); 
    }
} catch (Exception e) {
	log.error("Unexpected error", e);
} finally {
    try {
        //如果要关闭消费者，则需要保证提交成功
    	consumer.commitSync(); 
    } finally {
    	consumer.close();
    }
} 

```

### 提交特定的偏移量

消费者 API 允许在调用 commitSync() 和 commitAsync() 方法时传进去希望提交的分区和偏移量的 map。假设你处理了半个批次的消息，最后一个来自主题“customers”分区 3 的消息的偏移量是 5000，你可以调commitSync() 方法来提交它。不过，因为消费者可能不只读取一个分区，你需要跟踪所有分区的偏移量，所以在这个层面上控制偏移量的提交会让代码变复杂 

```java
private Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>(); ➊
int count = 0;
...
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records){
        System.out.printf("topic = %s, partition = %s, offset = %d, customer = %s, country = %s\n",record.topic(), record.partition(), record.offset(),
        record.key(), record.value());
        currentOffsets.put(new TopicPartition(record.topic(),
        record.partition()), 
   //在读取每条记录之后，使用期望处理的下一个消息的偏移量更新 map 里的偏移量。下一次就从这里开始读取消息。
        new OffsetAndMetadata(record.offset()+1, "no metadata"));
        //每处理 1000 条记录就提交一次偏移量。实际可以根据时间或记录的内容进行提交。
        if (count % 1000 == 0) consumer.commitAsync(currentOffsets,null);
        count++;
    }
} 

```



## 再均衡监听器

```java
private Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();
try {
    //把 ConsumerRebalanceListener 对象传给 subscribe() 方法，这是最重要的一步。
    consumer.subscribe(topics, new HandleRebalance()); 
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records){
            System.out.println("topic = %s, partition = %s, offset = %d,customer = %s, 				country =%s\n",record.topic(),record.partition(),
                               record.offset(),record.key(),record.value());
            currentOffsets.put(new TopicPartition(record.topic(),record.partition()), 
                               newOffsetAndMetadata(record.offset()+1, "no metadata"));
        }
        consumer.commitAsync(currentOffsets, null);
    }
} catch (WakeupException e) {
    // 忽略异常，正在关闭消费者
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    try { 
    	consumer.commitSync(currentOffsets);
    } finally {
        consumer.close();
        System.out.println("Closed consumer and we are done");
	}
}

//ConsumerRebalanceListener 接口
private class HandleRebalance implements ConsumerRebalanceListener { 
    //在获得新分区后开始读取消息，不需要做其他事情。
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) { }
    /*
    如果发生再均衡，我们要在即将失去分区所有权时提交偏移量。要注意，提交的是最近处理过的偏移量，而不是批次中还在处理的最后一个偏移量。因为分区有可能在我们还在处理消息的时候被撤回。我们要提交所有分区的偏移量，而不只是那些即将失去所有权的分区的偏移量——因为提交的偏移量是已经处理过的，所以不会有什么问题。调用commitSync() 方法，确保在再均衡发生之前提交偏移量
    */
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("Lost partitions in rebalance.Committing current offsets:" + currentOffsets);
        consumer.commitSync(currentOffsets); 
    }
}

```





