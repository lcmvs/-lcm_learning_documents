

# RabbitMQ环境

## 环境搭建



## 管理界面

![这里写图片描述](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/20180805224641957.png)

![这里写图片描述](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/20180805224654453.png)

## 添加用户

![这里写图片描述](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/20180805224521190.png)

1、超级管理员(administrator)
可登陆管理控制台，可查看所有的信息，并且可以对用户，策略(policy)进行操作。
2、监控者(monitoring)
可登陆管理控制台，同时可以查看rabbitmq节点的相关信息(进程数，内存使用情况，磁盘使用情况等)
3、策略制定者(policymaker)
可登陆管理控制台, 同时可以对policy进行管理。但无法查看节点的相关信息(上图红框标识的部分)。
4、普通管理者(management)
仅可登陆管理控制台，无法看到节点信息，也无法对策略进行管理。
5、其他

## 创建Virtual Hosts

![这里写图片描述](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/20180805224535905.png)

为用户配置权限

![这里写图片描述](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/20180805224551538.png)



# 队列模式

## 简单队列

简单队列就是一个生产者，一个消费者、对应一个队列。

![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/1120165-20180701235656495-1545126593.png)

声明队列

```java
@Configuration
public class WorkQueueConfig {

    @Bean
    public Queue workQueue() {
        return QueueBuilder
                .durable(QueueProperty.WORK_QUEUE)
                .build();
    }

}
```

生产者

```java
@Component
public class WorkProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.WORK_QUEUE, msg);
    }

}
```

消费者

```java
@Component
public class WorkListener {

    @RabbitListener(queues = QueueProperty.WORK_QUEUE)
    public void workProcess(WorkMsg msg){
        System.out.println(msg);
    }

}
```

## work队列

一个生产者对应多个消费者，但是一个消息只能被一个消费者消费

![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/1120165-20180717220023680-448688747.png)

简单队列情况下增加一个消费者

```java
@Component
public class WorkListener2 {

    @RabbitListener(queues = QueueProperty.WORK_QUEUE)
    public void workProcess2(WorkMsg msg){
        System.out.println(msg);
    }

}
```

## 发布订阅模式

　　**一个消费者将消息首先发送到交换器，交换器绑定到多个队列，然后被监听该队列的消费者所接收并消费。**

![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/1120165-20180717230211764-1067848569.png)



创建队列，绑定交换器

```java
@Component
public class FanoutConfig {

    /**
     * 声明 pubsub队列
     * @return
     */
    @Bean
    public Queue pubsubQueue1() {
        return QueueBuilder.durable(QueueProperty.PUBSUB_QUEUE_1).build();
    }

    @Bean
    public Queue pubsubQueue2() {
        return QueueBuilder.durable(QueueProperty.PUBSUB_QUEUE_2).build();
    }

    @Bean
    public Queue pubsubQueue3() {
        return QueueBuilder.durable(QueueProperty.PUBSUB_QUEUE_3).build();
    }

    /**
     * 声明队列交换机
     * @return
     */
    @Bean
    public FanoutExchange fanoutExchange() {
        return ExchangeBuilder.fanoutExchange(QueueProperty.PUBSUB_EXCHANGE).durable(true).build();
    }

    /**
     * 队列绑定交换机
     * @return
     */
    @Bean
    public Binding pubsubQueue1Binding(Queue pubsubQueue1, FanoutExchange fanoutExchange) {
        return BindingBuilder
                .bind(pubsubQueue1)
                .to(fanoutExchange);
    }

    @Bean
    public Binding pubsubQueue2Binding(Queue pubsubQueue2, FanoutExchange fanoutExchange) {
        return BindingBuilder
                .bind(pubsubQueue2)
                .to(fanoutExchange);
    }

    @Bean
    public Binding pubsubQueue3Binding(Queue pubsubQueue3, FanoutExchange fanoutExchange) {
        return BindingBuilder
                .bind(pubsubQueue3)
                .to(fanoutExchange);
    }

}
```

生产者

```java
@Component
public class FanoutProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send1(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.PUBSUB_EXCHANGE,"1", msg);
    }

    public void send2(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.PUBSUB_EXCHANGE,"2", msg);
    }

    public void send3(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.PUBSUB_EXCHANGE,"", msg);
    }
}
```

消费者

```java
@Component
public class FanoutListener {

    @RabbitListener(queues = QueueProperty.PUBSUB_QUEUE_1)
    public void pubsubProcess1(WorkMsg msg){
        System.out.println(msg);
    }

    @RabbitListener(queues = QueueProperty.PUBSUB_QUEUE_2)
    public void pubsubProcess2(WorkMsg msg){
        System.out.println(msg);
    }

    @RabbitListener(queues = QueueProperty.PUBSUB_QUEUE_3)
    public void pubsubProcess3(WorkMsg msg){
        System.out.println(msg);
    }
}
```

## 路由模式

　生产者将消息发送到direct交换器，在绑定队列和交换器的时候有一个路由key，生产者发送的消息会指定一个路由key，那么消息只会发送到相应key相同的队列，接着监听该队列的消费者消费消息。

　　**也就是让消费者有选择性的接收消息。**

![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/1120165-20180718075638848-561394407.png)

```java
@Configuration
public class DirectConfig {

    /**
     * 声明 direct队列
     * @return
     */
    @Bean
    public Queue directQueue1() {
        return QueueBuilder.durable(QueueProperty.DIRECT_QUEUE_1).build();
    }

    @Bean
    public Queue directQueue2() {
        return QueueBuilder.durable(QueueProperty.DIRECT_QUEUE_2).build();
    }

    /**
     * 声明队列交换机
     * @return
     */
    @Bean
    public DirectExchange directExchange() {
        return ExchangeBuilder.directExchange(QueueProperty.DIRECT_EXCHANGE).durable(true).build();
    }

    /**
     * 队列绑定交换机
     * @return
     */
    @Bean
    public Binding directQueue1Binding(Queue directQueue1, DirectExchange directExchange) {
        return BindingBuilder
                .bind(directQueue1)
                .to(directExchange)
                .with("1");
    }

    @Bean
    public Binding directQueue2Binding(Queue directQueue2, DirectExchange directExchange) {
        return BindingBuilder
                .bind(directQueue2)
                .to(directExchange)
                .with("2");
    }

}
```

生产者

```java
@Component
public class DirectProcuder {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.DIRECT_QUEUE_1, msg);
    }

    public void send1(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.DIRECT_EXCHANGE,"1", msg);
    }

    public void send2(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.DIRECT_EXCHANGE,"2", msg);
    }

}
```

消费者

```java
@Component
public class DirectListener {

    @RabbitListener(queues = QueueProperty.DIRECT_QUEUE_1)
    public void directProcess1(WorkMsg msg){
        System.out.println(msg);
    }

    @RabbitListener(queues = QueueProperty.DIRECT_QUEUE_2)
    public void directProcess2(WorkMsg msg){
        System.out.println(msg);
    }

}
```

## topic模式

上面的路由模式是根据路由key进行完整的匹配（完全相等才发送消息），这里的通配符模式通俗的来讲就是模糊匹配。也就是说topic模式是路由模式的升级加强版。

![img](../../%E5%85%AC%E5%BC%80%E6%96%87%E6%A1%A3/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97/assets/1120165-20180718081131530-359095794.png)

```java
@Configuration
public class TopicConfig {

    /**
     * 声明 direct队列
     * @return
     */
    @Bean
    public Queue topicQueue1() {
        return QueueBuilder.durable(QueueProperty.TOPIC_QUEUE_1).build();
    }

    @Bean
    public Queue topicQueue2() {
        return QueueBuilder.durable(QueueProperty.TOPIC_QUEUE_2).build();
    }

    @Bean
    public Queue topicQueue3() {
        return QueueBuilder.durable(QueueProperty.TOPIC_QUEUE_3).build();
    }

    /**
     * 声明队列交换机
     * @return
     */
    @Bean
    public TopicExchange topicExchange() {
        return ExchangeBuilder.topicExchange(QueueProperty.TOPIC_EXCHANGE).durable(true).build();
    }

    /**
     * 队列绑定交换机
     * @return
     */
    @Bean
    public Binding topicQueue1Binding(Queue topicQueue1, TopicExchange topicExchange) {
        return BindingBuilder
                .bind(topicQueue1)
                .to(topicExchange)
                .with("lcm.topic1");
    }

    @Bean
    public Binding topicQueue2Binding(Queue topicQueue2, TopicExchange topicExchange) {
        return BindingBuilder
                .bind(topicQueue2)
                .to(topicExchange)
                .with("lcm.topic2");
    }

    @Bean
    public Binding topicQueue3Binding(Queue topicQueue3, TopicExchange topicExchange) {
        return BindingBuilder
                .bind(topicQueue3)
                .to(topicExchange)
                .with("lcm.*");
    }
}
```

生产者

```java
@Component
public class TopicProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.TOPIC_QUEUE_1, msg);
    }

    public void send1(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.TOPIC_EXCHANGE,"lcm.topic1", msg);
    }

    public void send2(WorkMsg msg) {
        rabbitTemplate.convertAndSend(QueueProperty.TOPIC_EXCHANGE,"lcm.topic2", msg);
    }


}
```

消费者

```java
@Component
public class TopicListener {

    @RabbitListener(queues = QueueProperty.TOPIC_QUEUE_1)
    public void topicProcess1(WorkMsg msg){
        System.out.println(msg);
    }

    @RabbitListener(queues = QueueProperty.TOPIC_QUEUE_2)
    public void topicProcess2(WorkMsg msg){
        System.out.println(msg);
    }

    @RabbitListener(queues = QueueProperty.TOPIC_QUEUE_3)
    public void topicProcess3(WorkMsg msg){
        System.out.println(msg);
    }
}
```



# 交换器

direct

如果路由键完全匹配的话，消息才会被投放到相应的队列。

fanout

当发送一条消息到fanout交换器上时，它会把消息投放到所有附加在此交换器上的队列。

topic

设置模糊的绑定方式，“*”操作符将“.”视为分隔符，匹配单个字符；“#”操作符没有分块的概念，它将任意“.”均视为关键字的匹配部分，能够匹配多个字符。

headers

header 交换器和 direct 交换器功能基本上上一样，但是性能比较差，基本不会使用。



## 死信队列

声明死信队列、死信交换机、死信队列绑定死信交换机

```java
@Configuration
public class DeadConfig {

    /**
     * 声明work死信队列
     * @return
     */
    @Bean
    public Queue deadWorkQueue() {
        return QueueBuilder.durable(QueueProperty.DEAD_WORK_QUEUE).build();
    }

    /**
     * 声明死信队列交换机
     * @return
     */
    @Bean
    public DirectExchange deadLetterExchange() {
        return ExchangeBuilder.directExchange(QueueProperty.DEAD_EXCHANGE).durable(true).build();
    }

    /**
     * 死信队列绑定交换机
     * @return
     */
    @Bean
    public Binding deadLetterBinding(Queue deadWorkQueue, DirectExchange deadLetterExchange) {
        return BindingBuilder
                .bind(deadWorkQueue)
                .to(deadLetterExchange)
                .with(QueueProperty.DEAD_WORK_QUEUE);
    }

}
```

为某个队列绑定死信队列

```java
@Configuration
public class WorkQueueConfig {

    @Bean
    public Queue workQueue() {
        return QueueBuilder
                .durable(QueueProperty.WORK_QUEUE)
                .withArgument("x-dead-letter-exchange", QueueProperty.DEAD_EXCHANGE)
                .withArgument("x-dead-letter-routing-key", QueueProperty.DEAD_WORK_QUEUE)
                .build();
    }

}
```

消费死信队列的消息

```java
@Component
public class DeadListener {

    @RabbitListener(queues = QueueProperty.DEAD_WORK_QUEUE)
    public void workDeadProcess(WorkMsg msg){
        System.out.println(msg);
    }

}
```

# 批量处理

## 批量发送

实际上，RabbitMQ Broker 存储的是**一条**消息。又或者说，**RabbitMQ 并没有提供批量接收消息的 API 接口**。批量发送消息是 Spring-AMQP 的 [SimpleBatchingStrategy](https://github.com/spring-projects/spring-amqp/blob/master/spring-rabbit/src/main/java/org/springframework/amqp/rabbit/batch/SimpleBatchingStrategy.java) 所封装提供：

- 在 Producer 最终批量发送消息时，SimpleBatchingStrategy 会通过 [`#assembleMessage()`](https://github.com/spring-projects/spring-amqp/blob/master/spring-rabbit/src/main/java/org/springframework/amqp/rabbit/batch/SimpleBatchingStrategy.java#L141-L156) 方法，将批量发送的**多条**消息**组装**成一条“批量”消息，然后进行发送。
- 在 Consumer 拉取到消息时，会根据[`#canDebatch(MessageProperties properties)`](https://github.com/spring-projects/spring-amqp/blob/master/spring-rabbit/src/main/java/org/springframework/amqp/rabbit/batch/SimpleBatchingStrategy.java#L158-L163) 方法，判断该消息是否为一条“批量”消息？如果是，则调用[`# deBatch(Message message, Consumer fragmentConsumer)`](https://github.com/spring-projects/spring-amqp/blob/master/spring-rabbit/src/main/java/org/springframework/amqp/rabbit/batch/SimpleBatchingStrategy.java#L165-L194) 方法，将一条“批量”消息**拆开**，变成**多条**消息。

### BatchingRabbitTemplate

```java
@Configuration
public class BatchingRabbitTemplateConfig {

    @Autowired
    MessageConverter messageConverter;

    @Bean
    public BatchingRabbitTemplate batchRabbitTemplate(ConnectionFactory connectionFactory) {
        // 超过收集的消息数量的最大条数。
        int batchSize = 64;
        // 每次批量发送消息的最大内存,字节
        int bufferLimit = 1024;
        // 超过收集的时间的最大等待时长，单位：毫秒
        int timeout = 1000;
        // 创建 BatchingStrategy 对象，代表批量策略
        BatchingStrategy batchingStrategy = new SimpleBatchingStrategy(batchSize, bufferLimit, timeout);

        // 创建 TaskScheduler 对象，用于实现超时发送的定时器
        TaskScheduler taskScheduler = new ConcurrentTaskScheduler();

        // 创建 BatchingRabbitTemplate 对象
        BatchingRabbitTemplate batchingRabbitTemplate = new BatchingRabbitTemplate(connectionFactory, batchingStrategy, taskScheduler);

        batchingRabbitTemplate.setMessageConverter(messageConverter);

        return batchingRabbitTemplate;
    }

}
```

使用

```java
@Component
public class BatchingProducer {

    @Autowired
    BatchingRabbitTemplate batchingRabbitTemplate;

    public void send(WorkMsg msg){
        batchingRabbitTemplate.convertAndSend(QueueProperty.BATCH_QUEUE, msg);
    }

}
```

## 批量消费

```java
@Configuration
public class BatchSimpleRabbitListenerContainerFactory {

    @Bean(name = "myBatchSimpleRabbitListenerContainerFactory")
    public SimpleRabbitListenerContainerFactory consumerBatchContainerFactory(
            SimpleRabbitListenerContainerFactoryConfigurer configurer, ConnectionFactory connectionFactory) {
        // 创建 SimpleRabbitListenerContainerFactory 对象
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        configurer.configure(factory, connectionFactory);
        // 额外添加批量消费的属性
        factory.setBatchListener(true);
        factory.setBatchSize(10);
        factory.setReceiveTimeout(1000L);
        factory.setConsumerBatchEnabled(true);
        return factory;
    }

}
```

使用

```java
@Component
public class BatchingConsumer {

    @RabbitListener(queues = QueueProperty.BATCH_QUEUE , containerFactory = "myBatchSimpleRabbitListenerContainerFactory")
    public void batchListProcess(List<WorkMsg> list){
        log.info("成功消费{{}}条消息:{{}}",list.size(),list);
    }

}
```

# 确认机制

## 生产确认

在 RabbitMQ 中，**默认**情况下，Producer 发送消息的方法，只保证将消息写入到 TCP Socket 中成功，并不保证消息发送到 RabbitMQ Broker 成功，并且持久化消息到磁盘成功。

也就是说，我们上述的示例，Producer 在发送消息都不是绝对可靠，是存在丢失消息的可能性。

不过不用担心，在 RabbitMQ 中，Producer 采用 Confirm 模式，实现发送消息的确认机制，以保证消息发送的可靠性。实现原理如下：

- 首先，Producer 通过调用 [`Channel#confirmSelect()`](https://github.com/rabbitmq/rabbitmq-java-client/blob/master/src/main/java/com/rabbitmq/client/Channel.java#L1278-L1283) 方法，将 Channel 设置为 Confirm 模式。
- 然后，在该 Channel 发送的消息时，需要先通过 [`Channel#getNextPublishSeqNo()`](https://github.com/rabbitmq/rabbitmq-java-client/blob/master/src/main/java/com/rabbitmq/client/Channel.java#L1285-L1290) 方法，给发送的消息分配一个唯一的 ID 编号(`seqNo` 从 1 开始递增)，再发送该消息给 RabbitMQ Broker 。
- 之后，RabbitMQ Broker 在接收到该消息，并被路由到相应的队列之后，会发送一个包含消息的唯一编号(`deliveryTag`)的确认给 Producer 。

通过 `seqNo` 编号，将 Producer 发送消息的“请求”，和 RabbitMQ Broker 确认消息的“响应”串联在一起。

通过这样的方式，Producer 就可以知道消息是否成功发送到 RabbitMQ Broker 之中，保证消息发送的可靠性。不过要注意，整个执行的过程实际是**异步**，需要我们调用 [`Channel#waitForConfirms()`](https://github.com/rabbitmq/rabbitmq-java-client/blob/master/src/main/java/com/rabbitmq/client/Channel.java#L1293-L1329) 方法，**同步**阻塞等待 RabbitMQ Broker 确认消息的“响应”。

也因此，Producer 采用 Confirm 模式时，有三种编程方式：

- 【同步】普通 Confirm 模式：Producer 每发送一条消息后，调用 `Channel#waitForConfirms()` 方法，等待服务器端 Confirm 。

- 【同步】批量 Confirm 模式：Producer 每发送一批消息后，调用`Channel#waitForConfirms()` 方法，等待服务器端 Confirm 。

  > 本质上，和「普通 Confirm 模式」是一样的。

- 【异步】异步 Confirm 模式：Producer 提供一个回调方法，RabbitMQ Broker 在 Confirm 了一条或者多条消息后，Producer 会回调这个方法。

```java
`// CachingConnectionFactory#ConfirmType.javapublic enum ConfirmType {	/**	 * Use {@code RabbitTemplate#waitForConfirms()} (or {@code waitForConfirmsOrDie()}	 * within scoped operations.	 */	SIMPLE, // 使用同步的 Confirm 模式	/**	 * Use with {@code CorrelationData} to correlate confirmations with sent	 * messsages.	 */	CORRELATED, // 使用异步的 Confirm 模式	/**	 * Publisher confirms are disabled (default).	 */	NONE // 不使用 Confirm 模式}`
```

### 同步确认

在 RabbitTemplate 提供的 API 方法中，如果 Producer 要使用同步的 Confirm 模式，需要调用 `#invoke(action, acks, nacks)` 方法。代码如下：

```java
`// RabbitOperations.java// RabbitTemplate 实现了 RabbitOperations 接口/** * Invoke operations on the same channel. * If callbacks are needed, both callbacks must be supplied. * @param action the callback. * @param acks a confirm callback for acks. * @param nacks a confirm callback for nacks. * @param <T> the return type. * @return the result of the action method. * @since 2.1 */@Nullable<T> T invoke(OperationsCallback<T> action, @Nullable com.rabbitmq.client.ConfirmCallback acks,		@Nullable com.rabbitmq.client.ConfirmCallback nacks);`
```

- 因为 Confirm 模式需要基于**相同** Channel ，所以我们需要使用该方法。

- 在方法参数 `action` 中，我们可以自定义操作。

- 在方法参数 `acks` 中，定义接收到 RabbitMQ Broker 的成功“响应”时的成回调。

- 在方法参数 `nacks` 中，定义接收到 RabbitMQ Broker 的失败“响应”时的成回调。

  > - 当消息最终得到确认之后，生产者应用便可以通过回调方法来处理该确认消息。
  > - 如果 RabbitMQ 因为自身内部错误导致消息丢失，就会发送一条 nack 消息，生产者应用程序同样可以在回调方法中处理该 nack 消息。

使用

```java
@Component
public class SynConfirmProducer {

    @Autowired
    RabbitTemplate rabbitTemplate;

    public void synProducer(WorkMsg msg){
        rabbitTemplate.invoke(operations -> {
            // 同步发送消息
            operations.convertAndSend("error", "error", "test");
            // 等待确认 timeout 参数，如果传递 0 ，表示无限等待
            operations.waitForConfirms(0);
            return null;
        }, (deliveryTag, multiple) -> {
        }, (deliveryTag, multiple) -> {
            throw new RuntimeException("同步发送失败！");
        });
    }

}
```



### 异步确认

#### ConfirmCallback

```java
@Slf4j
@Component
public class MyConfirmCallback implements RabbitTemplate.ConfirmCallback {

    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        if (!ack) {
            log.error("消息发送异常!");
        } else {
            log.info("发送者爸爸已经收到确认，correlationData={} ,ack={}, cause={}", correlationData, ack, cause);
        }
    }
}
```

#### ReturnCallback

当 Producer 成功发送消息到 RabbitMQ Broker 时，但是在通过 Exchange 进行**匹配不到** Queue 时，Broker 会将该消息回退给 Producer 。

```java
@Slf4j
@Component
public class MyReturnCallback implements RabbitTemplate.ReturnCallback {

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("returnedMessage ===> replyCode={} ,replyText={} ,exchange={} ,routingKey={},message={}",
                replyCode, replyText, exchange, routingKey,message);
    }

}
```

使用

```java
@Slf4j
@Component
public class AsynConfirmProducer {

    @Autowired
    MyConfirmCallback myConfirmCallback;

    @Autowired
    MyReturnCallback myReturnCallback;

    @Autowired
    RabbitTemplate rabbitTemplate;

    public void synProducer(WorkMsg msg){
        rabbitTemplate.setMandatory(true);
        rabbitTemplate.setConfirmCallback(myConfirmCallback);
        rabbitTemplate.setReturnCallback(myReturnCallback);
        rabbitTemplate.convertAndSend(QueueProperty.DIRECT_EXCHANGE,"error", msg);
    }

}
```



## 消费确认

在 RabbitMQ 中，Consumer 有两种消息确认的方式：

- 方式一，自动确认。
- 方式二，手动确认。

对于**自动确认**的方式，RabbitMQ Broker 只要将消息写入到 TCP Socket 中成功，就认为该消息投递成功，而无需 Consumer **手动确认**。

对于**手动确认**的方式，RabbitMQ Broker 将消息发送给 Consumer 之后，由 Consumer **手动确认**之后，才任务消息投递成功。

实际场景下，因为自动确认存在可能**丢失消息**的情况，所以在对**可靠性**有要求的场景下，我们基本采用手动确认。当然，如果允许消息有一定的丢失，对**性能**有更高的产经下，我们可以考虑采用自动确认。

```java
// AcknowledgeMode.java

/**
 * No acks - {@code autoAck=true} in {@code Channel.basicConsume()}.
 */
NONE, // 对应 Consumer 的自动确认

/**
 * Manual acks - user must ack/nack via a channel aware listener.
 */
MANUAL, // 对应 Consumer 的手动确认，由开发者在消费逻辑中，手动进行确认。

/**
 * Auto - the container will issue the ack/nack based on whether
 * the listener returns normally, or throws an exception.
 * <p><em>Do not confuse with RabbitMQ {@code autoAck} which is
 * represented by {@link #NONE} here</em>.
 */
AUTO; // 对应 Consumer 的手动确认，在消费消息完成（包括正常返回、和抛出异常）后，由 Spring-AMQP 框架来“自动”进行确认。
```

# spring boot参数

```properties
# base
spring.rabbitmq.host: 服务Host
spring.rabbitmq.port: 服务端口
spring.rabbitmq.username: 登陆用户名
spring.rabbitmq.password: 登陆密码
spring.rabbitmq.virtual-host: 连接到rabbitMQ的vhost
spring.rabbitmq.addresses: 指定client连接到的server的地址，多个以逗号分隔(优先取addresses，然后再取host)
spring.rabbitmq.requested-heartbeat: 指定心跳超时，单位秒，0为不指定；默认60s
spring.rabbitmq.publisher-confirms: 是否启用【发布确认】
spring.rabbitmq.publisher-returns: 是否启用【发布返回】
spring.rabbitmq.connection-timeout: 连接超时，单位毫秒，0表示无穷大，不超时
spring.rabbitmq.parsed-addresses: 

# template
spring.rabbitmq.template.mandatory: 启用强制信息；默认false
spring.rabbitmq.template.receive-timeout: receive() 操作的超时时间
spring.rabbitmq.template.reply-timeout: sendAndReceive() 操作的超时时间
spring.rabbitmq.template.retry.enabled: 发送重试是否可用
spring.rabbitmq.template.retry.max-attempts: 最大重试次数
spring.rabbitmq.template.retry.initial-interval: 第一次和第二次尝试发布或传递消息之间的间隔
spring.rabbitmq.template.retry.multiplier: 应用于上一重试间隔的乘数
spring.rabbitmq.template.retry.max-interval: 最大重试时间间隔

# cache
spring.rabbitmq.cache.channel.size: 缓存中保持的channel数量
spring.rabbitmq.cache.channel.checkout-timeout: 当缓存数量被设置时，从缓存中获取一个channel的超时时间，单位毫秒；如果为0，则总是创建一个新channel
spring.rabbitmq.cache.connection.size: 缓存的连接数，只有是CONNECTION模式时生效
spring.rabbitmq.cache.connection.mode: 连接工厂缓存模式：CHANNEL 和 CONNECTION
	 
# listener
spring.rabbitmq.listener.simple.auto-startup: 是否启动时自动启动容器
spring.rabbitmq.listener.simple.acknowledge-mode: 表示消息确认方式，其有三种配置方式，分别是none、manual和auto；默认auto
spring.rabbitmq.listener.simple.concurrency: 最小的消费者数量
spring.rabbitmq.listener.simple.max-concurrency: 最大的消费者数量
spring.rabbitmq.listener.simple.prefetch: 指定一个请求能处理多少个消息，如果有事务的话，必须大于等于transaction数量.
spring.rabbitmq.listener.simple.transaction-size: 指定一个事务处理的消息数量，最好是小于等于prefetch的数量.
spring.rabbitmq.listener.simple.default-requeue-rejected: 决定被拒绝的消息是否重新入队；默认是true（与参数acknowledge-mode有关系）
spring.rabbitmq.listener.simple.idle-event-interval: 多少长时间发布空闲容器时间，单位毫秒

spring.rabbitmq.listener.simple.retry.enabled: 监听重试是否可用
spring.rabbitmq.listener.simple.retry.max-attempts: 最大重试次数
spring.rabbitmq.listener.simple.retry.initial-interval: 第一次和第二次尝试发布或传递消息之间的间隔
spring.rabbitmq.listener.simple.retry.multiplier: 应用于上一重试间隔的乘数
spring.rabbitmq.listener.simple.retry.max-interval: 最大重试时间间隔
spring.rabbitmq.listener.simple.retry.stateless: 重试是有状态or无状态

# ssl
spring.rabbitmq.ssl.enabled: 是否支持ssl
spring.rabbitmq.ssl.key-store: 指定持有SSL certificate的key store的路径
spring.rabbitmq.ssl.key-store-password: 指定访问key store的密码
spring.rabbitmq.ssl.trust-store: 指定持有SSL certificates的Trust store
spring.rabbitmq.ssl.trust-store-password: 指定访问trust store的密码
spring.rabbitmq.ssl.algorithm: ssl使用的算法，例如，TLSv1.1
```



# AMQP

AMQP是应用层协议，需要先打开一个tcp连接。建立tcp连接以后，应用程序会建立一个AMQP信道，AMQP命令就是通过信道发送的，每条信道都有一个唯一id。

## 命令概览

AMQP 0-9-1 协议中的命令远远不止上面所涉及的这些，为了让读者在遇到其他命令的时候能够迅速查阅相关信息，下面列举了 AMQP 0-9-1 协议主要的命令，包含名称、是否包含内容体 (Content Body) 、对应客户端中相应的方法及简要描述等四个维度进行说明。

| 名 称               | 是否包含内容体 | 对应客户端中的方法      | 简要描述                           |
| ------------------- | -------------- | ----------------------- | ---------------------------------- |
| Connection.start    | 否             | factory.newConnection   | 建立连接相关                       |
| Connection.Start-Ok | 否             | 同上                    | 同上                               |
| Connection.Tune     | 否             | 同上                    | 同上                               |
| Connection.Tune-Ok  | 否             | 同上                    | 同上                               |
| Connection.Open     | 否             | 同上                    | 同上                               |
| Connection.Open-Ok  | 否             | 同上                    | 同上                               |
| Connection.Close    | 否             | connection.close        | 关闭连接                           |
| Connection.Close-Ok | 否             | 同上                    | 同上                               |
| Channel.Open        | 否             | connection.openChannel  | 开启信道                           |
| Channel.Open-Ok     | 否             | 同上                    | 同上                               |
| Cbannel.Close       | 否             | channel.close           | 关闭信道                           |
| Channel.Close-Ok    | 否             | 同上                    | 同上                               |
| Exchange.Declare    | 否             | channel.exchangeDeclare | 声明交换器                         |
| Exchange.Declare-Ok | 否             | 同上                    | 同上                               |
| Exchange.Delete     | 否             | channel.exchangeDelete  | 删除交换器                         |
| Exchange.Delete-Ok  | 否             | 向上                    | 同上                               |
| Exchange.Bind       | 否             | channel.exchangeBind    | 交换器与交换器绑定                 |
| Exchange.Bind-Ok    | 否             | 同上                    | 同上                               |
| Exchange.Unbind     | 否             | channel.exchangeUnbind  | 交换器与交换器解绑                 |
| Exchange.Unbind-Ok  | 否             | 同上                    | 同上                               |
| Queue.Declare       | 否             | channel.queueDeclare    | 声明队列                           |
| Queue.Declare-Ok    | 否             | 同上                    | 同上                               |
| Queue.Bind          | 否             | channel.queueBind       | 队列与交换器绑定                   |
| Queue.Bind-Ok       | 否             | 同上                    | 同上                               |
| Queue.Purge         | 否             | channel.queuePurge      | 清除队列中的内容                   |
| Queue.Purge-Ok      | 否             | 同上                    | 同上                               |
| Queue.Delete        | 否             | channel.queueDelete     | 删除队列                           |
| Queue.Delete-Ok     | 否             | 同上                    | 向上                               |
| Queue.Unbind        | 否             | channel.queueUnbind     | 队列与交换器解绑                   |
| Queue.Unbind-Ok     | 否             | 同上                    | 同上                               |
| Basic.Qos           | 否             | channel.basicQos        | 设置未被确认消费的个数             |
| Basic.Qos-Ok        | 否             | 同上                    | 同上                               |
| Basic.Consume       | 否             | channel.basicConsume    | 消费消息(推模式)                   |
| BasiιConsurne-Ok    | 否             | 同上                    | 同上                               |
| Basic.Cancel        | 否             | channel.basicCancel     | 取消                               |
| Basic.Cancel-Ok     | 否             | 同上                    | 同上                               |
| Basic.Publish       | 是             | channel.basicPublish    | 发送消息                           |
| Basic.Return        | 是             | 无                      | 未能成功路由的消息返回             |
| Basic.Deliver       | 是             | 无                      | Broker 推送消息                    |
| Basic.Get           | 否             | channel.basicGet        | 消费消息(拉模式〉                  |
| Basic.Get-Ok        | 是             | 同上                    | 同上                               |
| Basic.Ack           | 否             | channel.basicAck        | 确认                               |
| Basic.Reject        | 否             | channel.basicReject     | 拒绝(单条拒绝)                     |
| Basic.Recover       | 否             | channel.basicRecover    | 请求 Broker 重新发送未被确认的消息 |
| Basic.Recover-Ok    | 否             | 向上                    | 同上                               |
| Basic.Nack          | 否             | channel.basicNack       | 拒绝(可批量拒绝〉                  |
| Tx.Select           | 否             | channel.txSelect        | 开启事务                           |
| TX.Select-Ok        | 否             | 同上                    | 同上                               |
| Tx.Cornmit          | 否             | channel.txCommit        | 事务提交                           |
| TX.Commit-Ok        | 否             | 同上                    | 同上                               |
| Tx.Rollback         | 否             | channel.txRollback      | 事务回滚                           |
| TX.Rollback-Ok      | 否             | 同上                    | 同上                               |
| Confirrn Select     | 否             | channel.confinnSelect   | 开启发送端确认模式                 |
| Confinn.Select-Ok   | 否             | 同上                    | 同上                               |

## 生产者消费者

　　在 RabbitMQ 的通信过程中，有两个主要的角色：生产者和消费者。类比于邮件通信的发送方和接收方。

　　这里首先我们要明确 RabbtiMQ  服务器是不能够产生数据的，正如同其名字——消息中间件，是一个用来传递消息的中间商。生产者产生创建消息，然后发布到代理服务器（RabbitMQ），而消费者则从代理服务器获取消息（不是直接找生产者要消息），而且在实际应用中，生产者和消费者也是可以角色互相转换的，所以当我们应用程序连接到  RabbitMQ 服务器时，必须要明确我是生产者呢还是消费者。

## 队列

AMQP消息路由必须有3部分：交换机，队列，绑定，生产者把消息发布到交换机上，消息最终到达队列，并被消费者接收消费。绑定决定了消息如何从路由器到特定的队列，消息到达队列中（存储）并等待消费，需要注意对于没有路由到队列的消息会被丢弃，消费者通过以下2种方式从特定的队列中**接受消息**：

1.通过AMQP的**basic.consume**命令订阅 。这样会将信道置为接受模式，直到取消对对列的订阅，订阅消息后，消费者在消费或者拒绝最近接受的那条消息之后，就能从队列中自动接收下一条消息；如果消费者处理队列消息并且需要在消息一到达队列时就自动接收就是用这个命令。

2.如果只是想获取队列中的单条消息而不是持续订阅的话，可以使用AMQP的**basic.get**命令，意思就是接受队列的下条消息，如果还需要获取下个继续调用这个命令，不建议放在循环中调用来替代consume命令，性能会有影响

**消息投递：**

1对1：一个生产者一个消费者，消息会直接投递给消费者。

1对0：如果队列没有消费者，消息会在队列中先存起来，等待消费者订阅。

1对多：多个消费者时，队列会以轮询的方式发送给消费者。

**消息确认：**消费者接收每一天消息都必须确认。

1.消费者必须通过basic.ack命令显示的向RabbitMq发送一个确认。

2.或者在订阅队列的时候就将auto_ack设置为true。

**消息未确认处理**：

1.消费者从rabbitMQ断开连接。这会导致rabbitMQ自动重新把消息入队，并发送给另外一个消费者。。优点是支持所有版本，缺点是性能差。

2.如果是2.0.0或者以上的版本，可以使用basic.reject命令。（requeue参数为true则发送给其他消费者，为false则从队列中移除）。

## 信道

生产者产生了消息，然后发布到 RabbitMQ 服务器，发布之前肯定要先连接上服务器，也就是要在应用程序和rabbitmq 服务器之间建立一条 TCP 连接，一旦连接建立，应用程序就可以创建一条 AMQP 信道。

　　信道是建立在“真实的”TCP 连接内的虚拟连接，AMQP 命令都是通过信道发送出去的，每条信道都会被指派一个唯一的ID（AMQP库会帮你记住ID的），不论是发布消息、订阅队列或者接收消息，这些动作都是通过信道来完成的。

可能有人会问，为什么不直接通过 TCP 连接来发送AMQP命令呢？

　　这里原因是效率问题，因为对于操作系统来说，每次建立和销毁 TCP 会话是非常昂贵的开销，而实际系统中，比如电商双十一，每秒钟高峰期成千上万条连接，一般来说操作系统建立TCP连接是有数量限制的，那么这就会遇到瓶颈。

　　引入信道的概念，我们可以在一条 TCP 连接上创建 N多个信道，这样既能发送命令，也能够保证每条信道的私密性，我们可以将其想象为光纤电缆。

## 虚拟主机

为了在一个单独的代理上实现多个隔离的环境（用户、用户组、交换机、队列 等），AMQP 提供了一个虚拟主机（virtual hosts - vhosts）的概念。这跟 Web servers 虚拟主机概念非常相似，这为 AMQP 实体提供了完全隔离的环境。当连接被建立的时候，AMQP 客户端来指定使用哪个虚拟主机。



## 持久化

默认情况下，重启rabbitmq，队列和交换机都会消失。每个队列和交换机的durable属性，默认都是false。

想要消息从rabbit崩溃中恢复，需要达到三个条件：

1.消息的投递模式选项为2（持久）

2.持久化的交换器

3.持久化的队列。

持久化日志：



性能：

写入磁盘要比写入内存慢的多，而且会极大的减少rabbitMQ的每秒可处理的消息总数。

集群环境工作的不好