# RocketMQ

RcoketMQ 是一款低延迟、高可靠、可伸缩、易于使用的消息中间件。具有以下特性：

1. 支持发布/订阅（Pub/Sub）和点对点（P2P）消息模型
2. 在一个队列中可靠的先进先出（FIFO）和严格的顺序传递
3. 支持拉（pull）和推（push）两种消息模式
4. 单一队列百万消息的堆积能力
5. 支持多种消息协议，如 JMS、MQTT 等
6. 分布式高可用的部署架构,满足至少一次消息传递语义
7. 提供 docker 镜像用于隔离测试和云集群部署
8. 提供配置、指标和监控等功能丰富的 Dashboard

## 1.名词说明

### Producer

消息生产者，生产者的作用就是将消息发送到 MQ，生产者本身既可以产生消息，如读取文本信息等。也可以对外提供接口，由外部应用来调用接口，再由生产者将收到的消息发送到 MQ。

### Producer Group

生产者组，简单来说就是多个发送同一类消息的生产者称之为一个生产者组。在这里可以不用关心，只要知道有这么一个概念即可。

### Consumer

消息消费者，简单来说，消费 MQ 上的消息的应用程序就是消费者，至于消息是否进行逻辑处理，还是直接存储到数据库等取决于业务需要。

### Consumer Group

消费者组，和生产者类似，消费同一类消息的多个 consumer 实例组成一个消费者组。

### Topic

Topic 是一种消息的逻辑分类，比如说你有订单类的消息，也有库存类的消息，那么就需要进行分类，一个是订单 Topic 存放订单相关的消息，一个是库存 Topic 存储库存相关的消息。

### Message

Message 是消息的载体。一个 Message 必须指定 topic，相当于寄信的地址。Message 还有一个可选的 tag  设置，以便消费端可以基于 tag 进行过滤消息。也可以添加额外的键值对，例如你需要一个业务 key 来查找 broker  上的消息，方便在开发过程中诊断问题。

### Tag

标签可以被认为是对 Topic 进一步细化。一般在相同业务模块中通过引入标签来标记不同用途的消息。

### Broker

Broker 是 RocketMQ 系统的主要角色，其实就是前面一直说的 MQ。Broker 接收来自生产者的消息，储存以及为消费者拉取消息的请求做好准备。

### Name Server

Name Server 为 producer 和 consumer 提供路由信息。

## RocketMQ 集群部署模式

### 1. 单 master 模式
    也就是只有一个 master 节点，称不上是集群，一旦这个 master 节点宕机，那么整个服务就不可用，适合个人学习使用。

### 2. 多 master 模式
    多个 master 节点组成集群，单个 master 节点宕机或者重启对应用没有影响。
    优点：所有模式中性能最高
    缺点：单个 master 节点宕机期间，未被消费的消息在节点恢复之前不可用，消息的实时性就受到影响。
    **注意**：使用同步刷盘可以保证消息不丢失，同时 Topic 相对应的 queue 应该分布在集群中各个节点，而不是只在某各节点上，否则，该节点宕机会对订阅该 topic 的应用造成影响。

### 3. 多 master 多 slave 异步复制模式
    在多 master 模式的基础上，每个 master 节点都有至少一个对应的 slave。master
    节点可读可写，但是 slave 只能读不能写，类似于 mysql 的主备模式。
    优点： 在 master 宕机时，消费者可以从 slave 读取消息，消息的实时性不会受影响，性能几乎和多 master 一样。
    缺点：使用异步复制的同步方式有可能会有消息丢失的问题。

### 4.  多 master 多 slave 同步双写模式

    同多 master 多 slave 异步复制模式类似，区别在于 master 和 slave 之间的数据同步方式。
    优点：同步双写的同步模式能保证数据不丢失。
    缺点：发送单个消息 RT 会略长，性能相比异步复制低10%左右。
    刷盘策略：同步刷盘和异步刷盘（指的是节点自身数据是同步还是异步存储）
    同步方式：同步双写和异步复制（指的一组 master 和 slave 之间数据的同步）
    **注意**：要保证数据可靠，需采用同步刷盘和同步双写的方式，但性能会较其他方式低。

## 2.单机部署安装

### 1. 下载[RocketMQ最新的二进制文件](https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.3.2/rocketmq-all-4.3.2-bin-release.zip)，并解压

    ```
    unzip rocketmq-all-4.3.2-bin-release.zip
    ```

### 2. 启动 Name Server

    ```
    sh bin/mqnamesrv
    ```

### 3. 启动 Broker

    ```
    sh bin/mqbroker -n localhost:9876
    ```

### 4. 关闭Name Server和Broker

   ```
   > sh bin/mqshutdown broker
   The mqbroker(36695) is running...
   Send shutdown request to mqbroker(36695) OK
   
   > sh bin/mqshutdown namesrv
   The mqnamesrv(36664) is running...
   Send shutdown request to mqnamesrv(36664) OK
   ```

### 5. 发送获取消息
   ```
   > export NAMESRV_ADDR=localhost:9876
   > sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
   SendResult [sendStatus=SEND_OK, msgId= ...
     
   > sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
   ConsumeMessageThread_%d Receive New Messages: [MessageExt...
   ```

### 6. 外网访问
    修改conf/broker.conf配置文件，增加如下配置：
    namesrvAddr = 外网ip:9876
    brokerIP1 = 外网ip
    
    ```conf
    brokerClusterName = DefaultCluster
    brokerName = broker-a
    brokerId = 0
    deleteWhen = 04
    fileReservedTime = 48
    brokerRole = ASYNC_MASTER
    flushDiskType = ASYNC_FLUSH
    
    namesrvAddr = 外网ip:9876
    brokerIP1 = 外网ip
    ```
    修改配置后，执行以下命令：
    ```
    nohup sh bin/mqbroker -c conf/broker.conf &
    ```

### 7. 内存溢出异常解决
    修改bin目录下sh文件runserver.sh和runbroker.sh的jdk内存大小

```
runserver.sh
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn125m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
JAVA_OPT="${JAVA_OPT} -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8  -XX:-UseParNewGC"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/rmq_srv_gc.log -XX:+PrintGCDetails"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT}  -XX:-UseLargePages"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"
```

```
runbroker.sh
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn125m"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:SurvivorRatio=8"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/mq_gc_%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintAdaptiveSizePolicy"
JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:+AlwaysPreTouch"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=15g"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages -XX:-UseBiasedLocking"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"
```
## 3.简单使用

### maven依赖

```xml
    <dependency>
        <groupId>org.apache.rocketmq</groupId>
        <artifactId>rocketmq-client</artifactId>
        <version>4.3.0</version>
    </dependency>
```

### 生产者

```java
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {

        //声明并初始化一个producer
        //需要一个producer group名字作为构造方法的参数，这里为producer1
        DefaultMQProducer producer = new DefaultMQProducer("producer1");

        //设置NameServer地址,此处应改为实际NameServer地址，多个地址之间用；分隔
        //NameServer的地址必须有，但是也可以通过环境变量的方式设置，不一定非得写死在代码里
        producer.setNamesrvAddr("ip:9876");

        //调用start()方法启动一个producer实例
        producer.start();

        //发送10条消息到Topic为TopicTest，tag为TagA，消息内容为“Hello RocketMQ”拼接上i的值
        for (int i = 0; i < 10; i++) {
            try {
                Message msg = new Message("TopicTest",// topic
                        "TagA",// tag
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)// body
                );

                //调用producer的send()方法发送消息
                //这里调用的是同步的方式，所以会有返回结果
                SendResult sendResult = producer.send(msg);

                //打印返回结果，可以看到消息发送的状态以及一些相关信息
                System.out.println(sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }

        //发送完消息之后，调用shutdown()方法关闭producer
        producer.shutdown();
    }
}
```

### 消费者

```java
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {

        //声明并初始化一个consumer
        //需要一个consumer group名字作为构造方法的参数，这里为consumer1
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer1");

        //同样也要设置NameServer地址
        consumer.setNamesrvAddr("ip:9876");

        //这里设置的是一个consumer的消费策略
        //CONSUME_FROM_LAST_OFFSET 默认策略，从该队列最尾开始消费，即跳过历史消息
        //CONSUME_FROM_FIRST_OFFSET 从队列最开始开始消费，即历史消息（还储存在broker的）全部消费一遍
        //CONSUME_FROM_TIMESTAMP 从某个时间点开始消费，和setConsumeTimestamp()配合使用，默认是半个小时以前
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        //设置consumer所订阅的Topic和Tag，*代表全部的Tag
        consumer.subscribe("TopicTest", "*");

        //设置一个Listener，主要进行消息的逻辑处理
        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                            ConsumeConcurrentlyContext context) {

                System.out.println(Thread.currentThread().getName() + " Receive New Messages: " + msgs);

                //返回消费状态
                //CONSUME_SUCCESS 消费成功
                //RECONSUME_LATER 消费失败，需要稍后重新消费
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        //调用start()方法启动consumer
        consumer.start();

        System.out.println("Consumer Started.");
    }
}
```

## 4. Spring Cloud Alibaba





## 5. 源码分析

