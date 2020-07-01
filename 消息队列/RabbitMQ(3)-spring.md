[Spring AMQP中文文档](http://liuxing.info/ 2017/06/30/Spring AMQP中文文档/)

[Spring AMQP英文文档](https://docs.spring.io/spring-amqp/docs/2.1.2.RELEASE/reference/html/)

[spring整合rabbitmq](https://blog.csdn.net/weixin_38380858/category_8466321.html)



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



# 概念

## AmqpTemplate

### 生产消息

```java
void send(Message message) throws AmqpException;

void send(String routingKey, Message message) throws AmqpException;

void send(String exchange, String routingKey, Message message) throws AmqpException;
```
方法三使用示例如下：
```java
amqpTemplate.send("marketData.topic", "quotes.nasdaq.FOO",
                  new Message("12.34".getBytes(), someProperties));
```

如果您计划使用该模板实例将大部分或全部 time 发送到同一个交换，则可以在模板上设置“exchange”property。在这种情况下，可以使用上面列出的第二种方法。以下 example 在功能上与前一个相同：

```java
amqpTemplate.setExchange("marketData.topic");
amqpTemplate.send("quotes.nasdaq.FOO", new Message("12.34".getBytes(), someProperties));
```

如果在模板上设置了“exchange”和“routingKey”properties，则可以使用仅接受`Message`的方法：

```java
amqpTemplate.setExchange("marketData.topic");
amqpTemplate.setRoutingKey("quotes.nasdaq.FOO");
amqpTemplate.send(new Message("12.34".getBytes(), someProperties));
```

## Message(消息体)

MessageProperties接口定义了几个常见的属性，如messageId，timestamp，contentType等等。
这些属性也可以通过调用setHeader(String key，Object value)方法来扩展用户定义的头属性。

```java
public class Message {

    private final MessageProperties messageProperties;

    private final byte[] body;

    public Message(byte[] body, MessageProperties messageProperties) {
        this.body = body;
        this.messageProperties = messageProperties;
    }

    public byte[] getBody() {
        return this.body;
    }

    public MessageProperties getMessageProperties() {
        return this.messageProperties;
    }
}
```

#### MessageBuilder

```java
Message message = MessageBuilder.withBody("foo".getBytes())
    .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
    .setMessageId("123")
    .setHeader("bar", "baz")
    .build();


```

提供了五种静态方法来创建初始消息构建器：

```java
public static MessageBuilder withBody(byte[] body) 

public static MessageBuilder withClonedBody(byte[] body) 

public static MessageBuilder withBody(byte[] body, int from, int to) 

public static MessageBuilder fromMessage(Message message) 

public static MessageBuilder fromClonedMessage(Message message)
```

> 构建器创建的消息将具有一个主体，该主体是参数的直接参考。

> 构建器创建的消息将具有一个主体，该主体是包含参数中字节副本的新 array。

> 构建器创建的消息将具有一个主体，该主体是一个新的 array，包含参数中的字节范围。有关详细信息，请参阅`Arrays.copyOfRange()`。

> 构建器创建的消息将具有一个主体，该主体是参数主体的直接参考。参数的 properties 被复制到新的`MessageProperties` object。

> 构建器创建的消息将具有一个主体，该主体是包含参数主体副本的新 array。参数的 properties 被复制到新的`MessageProperties` object。

#### MessageProperties
```java
MessageProperties props = MessagePropertiesBuilder.newInstance()
    .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
    .setMessageId("123")
    .setHeader("bar", "baz")
    .build();
Message message = MessageBuilder.withBody("foo".getBytes())
    .andProperties(props)
    .build();
```



```java
public static MessagePropertiesBuilder newInstance() 

public static MessagePropertiesBuilder fromProperties(MessageProperties properties) 

public static MessagePropertiesBuilder fromClonedProperties(MessageProperties properties)
```

> 使用默认值初始化新消息 properties object。

> 使用提供的 properties object 初始化构建器，并将 return。

> 参数的 properties 被复制到一个新的`MessageProperties` object。
> 使用`AmqpTemplate`的`RabbitTemplate` implementation，每个`send()`方法都有一个重载的 version，需要额外的`CorrelationData` object。启用发布者确认后，将在[第 3.1.4 节，“AmqpTemplate”](https://www.docs4dev.com/docs/zh/spring-amqp/2.1.2.RELEASE/reference/_reference.html#amqp-template)中描述的回调中返回此 object。这允许发送者将确认(ack 或 nack)与发送的消息相关联。



### 消息转换器

`MessageConverter`通过将提供的 object 转换为`Message`主体的 byte数组然后添加任何提供的`MessageProperties`来负责创造每个`Message`

```java
void convertAndSend(Object message) throws AmqpException;

void convertAndSend(String routingKey, Object message) throws AmqpException;

void convertAndSend(String exchange, String routingKey, Object message)
    throws AmqpException;

void convertAndSend(Object message, MessagePostProcessor messagePostProcessor)
    throws AmqpException;

void convertAndSend(String routingKey, Object message,
    MessagePostProcessor messagePostProcessor) throws AmqpException;

void convertAndSend(String exchange, String routingKey, Object message,
    MessagePostProcessor messagePostProcessor) throws AmqpException;
```



## exchange(交换机)

```java
public interface Exchange {

    String getName();

    String getExchangeType();

    boolean isDurable();

    boolean isAutoDelete();

    Map<String, Object> getArguments();

}
```

如您所见，Exchange还具有由ExchangeTypes中定义的常量表示的类型。
基本类型有：Direct, Topic, Fanout, 和 Headers。
在核心包中，您将找到每种类型的Exchange接口的实现。
这些Exchange类型的行为在如何处理与队列绑定方面有所不同。
例如，Direct exchange允许队列被固定的routing key(通常是队列的名称)绑定。
Topic exchange支持绑定与路由模式，可能包括`*`和`#`通配符

## Queue(队列)

```java
public class Queue  {

    private final String name;

    private volatile boolean durable;

    private volatile boolean exclusive;

    private volatile boolean autoDelete;

    private volatile Map<String, Object> arguments;

    /**
     * The queue is durable, non-exclusive and non auto-delete.
     *
     * @param name the name of the queue.
     */
    public Queue(String name) {
        this(name, true, false, false);
    }

    // Getters and Setters omitted for brevity

}
```

## Binding(绑定)

鉴于生产者发送到Exchange并且消费者从队列接收到消息，将队列连接到exchange的绑定对于通过消息传递连接这些生产者和消费者至关重要。
在Spring AMQP中，我们定义一个Binding类来表示这些连接。
我们来看看将队列绑定到交换机的基本选项。

您可以使用固定的routing key将队列绑定到DirectExchange。

```java
new Binding(someQueue, someDirectExchange, "foo.bar")
```

您可以使用路由模式将队列绑定到TopicExchange。

```java
new Binding(someQueue, someTopicExchange, "foo.*")
```

您可以使用无routing key将Queue绑定到FanoutExchange。

```java
new Binding(someQueue, someFanoutExchange)
```

我们还提供了一个BindingBuilder进行链式风格的构建。

```java
Binding b = BindingBuilder.bind(someQueue).to(someTopicExchange).with("foo.*");
```



# CachingConnectionFactory

用于管理与RabbitMQ代理的连接的中心组件是`ConnectionFactory`接口。 `ConnectionFactory`实现的责任是提供一个`org.springframework.amqp.rabbit.connection.Connection`的实例，它是`com.rabbitmq.client.Connection`的包装器。我们提供的唯一具体实现是`CachingConnectionFactory`，默认情况下，它建立可以由应用程序共享的单个连接代理。连接是共享的，因为与AMQP通信的“工作单位”实际上是一个“通道”(在某些方面，这与JMS中的连接和会话之间的关系类似)。您可以想像，连接实例提供了一个`createChannel`方法。 `CachingConnectionFactory`实现支持对这些通道的缓存，并且基于它们是否是事务来维护单独的通道高速缓存。创建`CachingConnectionFactory`实例时，可以通过构造函数提供主机名。还应提供用户名和密码属性。如果要配置通道缓存的大小(默认值为25)，您也可以在此处调用`setChannelCacheSize()`方法。


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

# cache
spring.rabbitmq.cache.channel.size: 缓存中保持的channel数量
spring.rabbitmq.cache.channel.checkout-timeout: 当缓存数量被设置时，从缓存中获取一个channel的超时时间，单位毫秒；如果为0，则总是创建一个新channel
spring.rabbitmq.cache.connection.size: 缓存的连接数，只有是CONNECTION模式时生效
spring.rabbitmq.cache.connection.mode: 连接工厂缓存模式：CHANNEL 和 CONNECTION

# ssl
spring.rabbitmq.ssl.enabled: 是否支持ssl
spring.rabbitmq.ssl.key-store: 指定持有SSL certificate的key store的路径
spring.rabbitmq.ssl.key-store-password: 指定访问key store的密码
spring.rabbitmq.ssl.trust-store: 指定持有SSL certificates的Trust store
spring.rabbitmq.ssl.trust-store-password: 指定访问trust store的密码
spring.rabbitmq.ssl.algorithm: ssl使用的算法，例如，TLSv1.1
```

# RabbitTemplate
```properties
# template
spring.rabbitmq.template.mandatory: 启用强制信息；默认false
spring.rabbitmq.template.receive-timeout: receive() 操作的超时时间
spring.rabbitmq.template.reply-timeout: sendAndReceive() 操作的超时时间
spring.rabbitmq.template.retry.enabled: 发送重试是否可用
spring.rabbitmq.template.retry.max-attempts: 最大重试次数
spring.rabbitmq.template.retry.initial-interval: 第一次和第二次尝试发布或传递消息之间的间隔
spring.rabbitmq.template.retry.multiplier: 应用于上一重试间隔的乘数
spring.rabbitmq.template.retry.max-interval: 最大重试时间间隔
```
## 简单使用

```java
ApplicationContext context = 
    new AnnotationConfigApplicationContext(RabbitConfiguration.class);
AmqpTemplate template = context.getBean(AmqpTemplate.class);
template.convertAndSend("myqueue", "foo");
String foo = (String) template.receiveAndConvert("myqueue");
........
    
@Configuration
public class RabbitConfiguration {

    @Bean
    public ConnectionFactory connectionFactory() {
        return new CachingConnectionFactory("localhost");
    }
    
    @Bean
    public AmqpAdmin amqpAdmin() {
        return new RabbitAdmin(connectionFactory());
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate() {
        return new RabbitTemplate(connectionFactory());
    }
    
    @Bean
    public Queue myQueue() {
       return new Queue("myqueue");
    }
}

```

# RabbitMessagingTemplate

从版本1.4开始，构建在`RabbitTemplate`之上的`RabbitMessagingTemplate`提供了与Spring Framework消息抽象(即`org.springframework.messaging.Message`)的集成。这允许您使用`spring-messaging` `Message<?>`抽象发送和接收消息。这种抽象是由Spring Integration和Spring的STOMP支持的其他Spring项目使用的。有两个消息转换器涉及;一个用于在Spring消息传递`Message<?>`和Spring AMQP的`Message`抽象之间进行转换，另一个用于在Spring AMQP的Message抽象与底层RabbitMQ客户端库所需的格式之间进行转换。默认情况下，消息有效载荷由提供的`RabbitTemplate`的消息转换器转换。或者，您可以使用其他有效载荷转换器注入自定义`MessagingMessageConverter`：

```java
MessagingMessageConverter amqpMessageConverter = new MessagingMessageConverter();
amqpMessageConverter.setPayloadConverter(myPayloadConverter);
rabbitMessagingTempalte.setAmqpMessageConverter(amqpMessageConverter);
```



# BatchingRabbitTemplate

批量发送确认

```java
`public interface BatchingStrategy {	MessageBatch addToBatch(String exchange, String routingKey, Message message);	Date nextRelease();	Collection<MessageBatch> releaseBatches();}`
```



# AsyncRabbitTemplate

异步发送确认

版本1.6引入了`AsyncRabbitTemplate`。这与`AmqpTemplate`中的`sendAndReceive`(和`convertSendAndReceive`)类似，但是它们返回一个`ListenableFuture`。

您可以稍后通过调用`get()`来同步检索结果，也可以注册一个将结果异步调用的回调。

```java
@Autowired
private AsyncRabbitTemplate template;

...

public void doSomeWorkAndGetResultLater() {

    ...

    ListenableFuture<String> future = this.template.convertSendAndReceive("foo");

    // do some more work

    String reply = null;
    try {
        reply = future.get();
    }
    catch (ExecutionException e) {
        ...
    }

    ...

}

public void doSomeWorkAndGetResultAsync() {

    ...

    RabbitConverterFuture<String> future = this.template.convertSendAndReceive("foo");
    future.addCallback(new ListenableFutureCallback<String>() {

        @Override
        public void onSuccess(String result) {
            ...
        }

        @Override
        public void onFailure(Throwable ex) {
            ...
        }

    });

    ...

}
```



# 消费消息

## 轮询消费-拉

```java
Message receive() throws AmqpException;

Message receive(String queueName) throws AmqpException;

Message receive(long timeoutMillis) throws AmqpException;

Message receive(String queueName, long timeoutMillis) throws AmqpException;
```

### 序列化

就像发送消息的情况一样，`AmqpTemplate`有一些方便的方法来接收 POJO 而不是`Message`实例，而 implementations 将提供一种方法来自定义用于创建`Object`返回的`MessageConverter`

```java
Object receiveAndConvert() throws AmqpException;

Object receiveAndConvert(String queueName) throws AmqpException;

Message receiveAndConvert(long timeoutMillis) throws AmqpException;

Message receiveAndConvert(String queueName, long timeoutMillis) throws AmqpException;
```

### 回复消息

从 version 2.0 开始，这些方法的变体采用额外的`ParameterizedTypeReference`参数来转换复杂类型。必须使用`SmartMessageConverter`配置模板;有关详细信息，请参阅[“使用 RabbitTemplate 从消息转换”一节](https://www.docs4dev.com/docs/zh/spring-amqp/2.1.2.RELEASE/reference/_reference.html#json-complex)。

与方法类似，从 version 1.3 开始，`AmqpTemplate`有几个方便的`receiveAndReply`方法，用于同步接收，处理和回复消息：

```java
<R, S> boolean receiveAndReply(ReceiveAndReplyCallback<R, S> callback)
	   throws AmqpException;

<R, S> boolean receiveAndReply(String queueName, ReceiveAndReplyCallback<R, S> callback)
 	throws AmqpException;

<R, S> boolean receiveAndReply(ReceiveAndReplyCallback<R, S> callback,
	String replyExchange, String replyRoutingKey) throws AmqpException;

<R, S> boolean receiveAndReply(String queueName, ReceiveAndReplyCallback<R, S> callback,
	String replyExchange, String replyRoutingKey) throws AmqpException;

<R, S> boolean receiveAndReply(ReceiveAndReplyCallback<R, S> callback,
 	ReplyToAddressCallback<S> replyToAddressCallback) throws AmqpException;

<R, S> boolean receiveAndReply(String queueName, ReceiveAndReplyCallback<R, S> callback,
			ReplyToAddressCallback<S> replyToAddressCallback) throws AmqpException;
```



## 异步监听消费-推
```properties
# listener
spring.rabbitmq.listener.type: simple(默认)或者direct
# simple
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

#direct
spring.rabbitmq.listener.direct.acknowledge-mode= # 确认容器的模式
spring.rabbitmq.listener.direct.auto-startup=true # 是否在启动时自动启动容器
spring.rabbitmq.listener.direct.consumers-per-queue= # 每个队列的消费者数量
spring.rabbitmq.listener.direct.default-requeue-rejected= # 默认情况下，拒绝交付是否重新排队
spring.rabbitmq.listener.direct.idle-event-interval= # 应该多久发布一次空闲容器事件
spring.rabbitmq.listener.direct.missing-queues-fatal=false # 如果容器声明的队列在代理上不可用，则是否失败
spring.rabbitmq.listener.direct.prefetch= # 每个消费者可能未完成的最大未确认消息数
spring.rabbitmq.listener.direct.retry.enabled=false # 是否启用发布重试
spring.rabbitmq.listener.direct.retry.initial-interval=1000ms # 第一次和第二次尝试传递消息之间的持续时间
spring.rabbitmq.listener.direct.retry.max-attempts=3 # 传递消息的最大尝试次数
spring.rabbitmq.listener.direct.retry.max-interval=10000ms # 最长尝试次数
spring.rabbitmq.listener.direct.retry.multiplier=1 # 乘数应用于先前的重试间隔
spring.rabbitmq.listener.direct.retry.stateless=true # 重试是无国籍还是有状态
```
推送

Spring AMQP 还通过使用`@RabbitListener` annotation 支持 annotated-listener endpoints，并提供了一个开放的基础结构来以编程方式注册 endpoints。这是设置异步 consumer 最方便的方法，有关详细信息，请参阅[名为“Annotation-driven Listener Endpoints”的部分](https://www.docs4dev.com/docs/zh/spring-amqp/2.1.2.RELEASE/reference/_reference.html#async-annotation-driven)。

### @RabbitListener

```java
@Component
public class MyService {

    @RabbitListener(queues = "myQueue")
    public void processOrder(String data) {
        ...
    }

}
```



### Container

```java
public abstract static class AmqpContainer {

   /**
    * 是否启动时自动启动容器
    */
   private boolean autoStartup = true;

   /**
    * 确认模式
    */
   private AcknowledgeMode acknowledgeMode;

   /**
    * 消费者最多持有未确认消息
    */
   private Integer prefetch;

   /**
    * 被拒绝的消息是否重新入队；默认是true（与参数acknowledge-mode有关系）
    */
   private Boolean defaultRequeueRejected;

   /**
    * 多久发布一次空闲容器事件，删除空闲消费者
    */
   private Duration idleEventInterval;

   /**
    * 重试机制
    */
   private final ListenerRetry retry = new ListenerRetry();
}

public static class ListenerRetry extends Retry {

    /**
	 * Whether retries are stateless or stateful.
	 */
    private boolean stateless = true;
}

public static class Retry {

    //是否开启重试
    private boolean enabled;

    //最大重试次数
    private int maxAttempts = 3;

    //第一个重试间隔时间
    private Duration initialInterval = Duration.ofMillis(1000);

    //应用于前一个重试间隔的乘数
    private double multiplier = 1.0;

    //最大重试间隔时间
    private Duration maxInterval = Duration.ofMillis(10000);
}
```



#### AcknowledgeMode

**AcknowledgeMode.NONE**

- `rabbitmq server`默认推送的所有消息都已经消费成功，会不断地向消费端推送消息。
- 因为`rabbitmq server`认为推送的消息已被成功消费，所以推送出去的消息不会暂存在`server`端。

**AcknowledgeMode.MANUAL**模式需要人为地获取到channel之后调用方法向server发送ack（或消费失败时的nack）信息。

**AcknowledgeMode.AUTO**模式下，由spring-rabbit依据消息处理逻辑是否抛出异常自动发送ack（无异常）或nack（异常）到server端。

#### SimpleRabbitListenerContainer



```java
public static class SimpleContainer extends AmqpContainer {

   /**
    * 最小并发线程
    */
   private Integer concurrency;

   /**
    * 最大并发线程
    */
   private Integer maxConcurrency;

   /**
    * 批量消息大小
    */
   private Integer batchSize;

   /**
    * 当在broker上找不到队列处理
    * true:consumer将进行3次重试以连接到队列（每5秒间隔），并在尝试失败时停止容器。
    * false:3次重试以后将继续进行恢复
    */
   private boolean missingQueuesFatal = true;
}
```

##### 实现

```java
public class SimpleMessageListenerContainer extends AbstractMessageListenerContainer {

   private int batchSize = 1;

   //并发消费者
   private Set<BlockingQueueConsumer> consumers;

   private final ActiveObjectCounter<BlockingQueueConsumer> cancellationLock = new ActiveObjectCounter<>();

   private Integer declarationRetries;

   private Long retryDeclarationInterval;

   private TransactionTemplate transactionTemplate;

   private long consumerStartTimeout = DEFAULT_CONSUMER_START_TIMEOUT;

   private volatile int concurrentConsumers = 1;

   private volatile Integer maxConcurrentConsumers;

}

//AbstractMessageListenerContainer
//默认使用的线程工厂，大概就是给每个消费者分配一个线程
private Executor taskExecutor = new SimpleAsyncTaskExecutor();
```

#### DirectMessageListenerContainer



```java
public static class DirectContainer extends AmqpContainer {

   /**
    * 每个队列的消费者数量
    */
   private Integer consumersPerQueue;

   /**
    * Whether to fail if the queues declared by the container are not available on
    * the broker.
    */
   private boolean missingQueuesFatal = false;
    
}
```



#### 选择Container

2.0版本增加DirectMessageListenerContainer，以前的版本只有SimpleRabbitListenerContainer。

smlc使用一个内部队列和一个专有线程，如果一个Container配置监听了多个队列，这一个线程需要处理所有的队列。并发是由`concurrentConsumers` 和控制。当消息到达rabbitmq客户端，客户端线程将它们加入队列再传递给消费者。

DMLC直接通过rabbitmq客户端线程唤醒listener，因此它的结构比SMLC更简单。但是它们有各自的优势:

SMLC独有特点:

- txSize 可以通过这个属性控制事务中传递的Message的数量，并且增加或减少ack的数量，但是这样做可能会导致消息发送失败后重复发送的次数增加。
- maxConcurrentConsumers DMLC中不支持自动缩放，只能够已编码的方式修改consumersPerQueue的属性

DMLC优势：

- 相比于SMLC，DMLC更方便的可以再运行时动态的添加或者删除队列
- 避免了rabbitmq客户端线程和消费者线程之间的上下文切换
- 消费者之间共享线程，而SMLC则是为每个消费者提供专用线程