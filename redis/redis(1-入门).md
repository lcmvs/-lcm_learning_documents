



## 客户端选择

Lettuce 和 Jedis 的定位都是Redis的client，所以他们当然可以直接连接redis server。

### jedis

Jedis在实现上是直接连接的redis server，如果在多线程环境下是非线程安全的，这个时候只有使用连接池，为每个Jedis实例增加物理连接。

### lettuce

Lettuce的连接是基于Netty的，连接实例（StatefulRedisConnection）可以在多个线程间并发访问，应为StatefulRedisConnection是线程安全的，所以一个连接实例（StatefulRedisConnection）就可以满足多线程环境下的并发访问，当然这个也是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。



## 添加依赖

目前java操作redis的客户端有`jedis`跟`Lettuce`。在`springboot1.x`系列中，其中使用的是`jedis`,但是到了`springboot2.x`其中使用的是`Lettuce`。

spring boot 2.x默认使用lettuce作为redis客户端。如果需要使用jedis，需要去除lettuce的依赖，如果两个依赖都存在，优先选择lettuce。

### lettuce

springboot中的spring-boot-starter-data-redis已经包含了lettuce的依赖。，所有不需要额外添加依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```



### jedis

使用jedis需要去除lettuce的依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```



## 配置大全

```java
@ConfigurationProperties(prefix = "spring.redis")
public class RedisProperties {

   /**
    * Database index used by the connection factory.
    */
   private int database = 0;

   /**
    * Connection URL. Overrides host, port, and password. User is ignored. Example:
    * redis://user:password@example.com:6379
    */
   private String url;

   /**
    * Redis server host.
    */
   private String host = "localhost";

   /**
    * Login password of the redis server.
    */
   private String password;

   /**
    * Redis server port.
    */
   private int port = 6379;

   /**
    * Whether to enable SSL support.
    */
   private boolean ssl;

   /**
    * Connection timeout.
    */
   private Duration timeout;

   private Sentinel sentinel;

   private Cluster cluster;

   private final Jedis jedis = new Jedis();

   private final Lettuce lettuce = new Lettuce();

   /**
    * Pool properties.
    */
   public static class Pool {

      /**
       * Maximum number of "idle" connections in the pool. Use a negative value to
       * indicate an unlimited number of idle connections.
       */
      private int maxIdle = 8;

      /**
       * Target for the minimum number of idle connections to maintain in the pool. This
       * setting only has an effect if it is positive.
       */
      private int minIdle = 0;

      /**
       * Maximum number of connections that can be allocated by the pool at a given
       * time. Use a negative value for no limit.
       */
      private int maxActive = 8;

      /**
       * Maximum amount of time a connection allocation should block before throwing an
       * exception when the pool is exhausted. Use a negative value to block
       * indefinitely.
       */
      private Duration maxWait = Duration.ofMillis(-1);

   }

   /**
    * Cluster properties.
    */
   public static class Cluster {

      /**
       * Comma-separated list of "host:port" pairs to bootstrap from. This represents an
       * "initial" list of cluster nodes and is required to have at least one entry.
       */
      private List<String> nodes;

      /**
       * Maximum number of redirects to follow when executing commands across the
       * cluster.
       */
      private Integer maxRedirects;

      public List<String> getNodes() {
         return this.nodes;
      }

      public void setNodes(List<String> nodes) {
         this.nodes = nodes;
      }

      public Integer getMaxRedirects() {
         return this.maxRedirects;
      }

      public void setMaxRedirects(Integer maxRedirects) {
         this.maxRedirects = maxRedirects;
      }

   }

   /**
    * Redis sentinel properties.
    */
   public static class Sentinel {

      /**
       * Name of the Redis server.
       */
      private String master;

      /**
       * Comma-separated list of "host:port" pairs.
       */
      private List<String> nodes;

      public String getMaster() {
         return this.master;
      }

      public void setMaster(String master) {
         this.master = master;
      }

      public List<String> getNodes() {
         return this.nodes;
      }

      public void setNodes(List<String> nodes) {
         this.nodes = nodes;
      }

   }

   /**
    * Jedis client properties.
    */
   public static class Jedis {

      private Pool pool;

   }

   /**
    * Lettuce client properties.
    */
   public static class Lettuce {

      private Duration shutdownTimeout = Duration.ofMillis(100);

      private Pool pool;

   }

}
```



### 单机配置示例

```yaml
spring:    
    redis:
      database: 0
      host: 10.172.0.201
      port: 6379
      # 连接超时时长（毫秒）
      timeout: 6000
      jedis:
        pool:
          # 连接池最大连接数（使用负值表示没有限制）
          max-active: 16
          # 连接池最大阻塞等待时间,单位毫秒（使用负值表示没有限制）
          max-wait: 10000
          # 连接池中的最大空闲连接
          max-idle: 10
          # 连接池中的最小空闲连接
          min-idle: 5
```



### 哨兵配置示例
```yaml
spring:
  redis:
    timeout: 6000ms
    lettuce:
      pool:
        max-active: 8
        max-wait: -1ms
        max-idle: 8
        min-idle: 0
    #sentinel哨兵配置
    sentinel:
      master: mymaster
      nodes: 127.0.0.1:26379,127.0.0.1:26380,127.0.0.1:26381
```



### 集群配置示例

```yaml
spring:
  redis:
    password:
    cluster:
      nodes: 10.1.96.10:8001,10.1.96.10:8002,10.1.96.11:8003,10.1.96.11:8004,10.1.96.12:8005,10.1.96.12:8006
      max-redirects: 3
    lettuce:
      pool:
        max-idle: 16
        max-active: 32
        min-idle: 8
```



## RedisTemplate序列化

### 序列化方式性能比较

#### 解析性能

![img](assets/f615deb9.png)

#### 序列化之空间开销

![img](assets/37cf0654.png)


### OxmSerializer

 以xml格式存储（**但还是String类型~**），解析起来也比较复杂，效率也比较低。因此几乎没有人再使用此方式了

### JdkSerializationRedisSerializer

 从源码里可以看出，这是RestTemplate类**默认的**序列化方式。若你没有自定义，那就是它了。

```java
public class RedisTemplate<K, V> extends RedisAccessor implements RedisOperations<K, V>, BeanClassLoaderAware {
    public void afterPropertiesSet() {

       super.afterPropertiesSet();

       boolean defaultUsed = false;

       if (defaultSerializer == null) {

          defaultSerializer = new JdkSerializationRedisSerializer(
                classLoader != null ? classLoader : this.getClass().getClassLoader());
       }
    }
}
```

 优点是反序列化时不需要提供（传入）类型信息(class)。

1.它要求存储的对象都必须实现`java.io.Serializable`接口，比较笨重 

2.他存储的为二进制数据，这对开发者是不友好的  

3.因为他存储的为**二进制**。但是有时候，我们的Redis会在一个项目的多个project中共用，这样如果同一个可以缓存的对象在不同的project中要使用两个不同的key来分别缓存，**既麻烦，又浪费**。

4.还有序列化后的结果非常庞大，**是JSON格式的5倍左右，这样就会消耗redis服务器的大量内存。**序列化速度也是json的更快。



###  StringRedisSerializer

 也是StringRedisTemplate默认的序列化方式，key和value都会采用此方式进行序列化，是被推荐使用的，对开发者友好，轻量级，效率也比较高。  

###  GenericToStringSerializer

他需要调用者给传一个对象到字符串互转的Converter（相当于转换为字符串的操作交给转换器去做），个人觉得使用起来其比较麻烦，还不如直接用字符串呢。所以不太推荐使用



### Jackson2JsonRedisSerializer

从名字可以看出来，这是把一个对象以Json的形式存储，效率高且对调用者友好

**优点是速度快，序列化后的字符串短小精悍**，不需要实现Serializable接口。

但缺点也非常致命：那就是此类的构造函数中有一个类型参数，**必须提供要序列化对象的类型信息(.class对象)**。 通过查看源代码，发现其在反序列化过程中用到了类型信息（必须根据此类型信息完成反序列化）。

```java
@Test
public void test(){
    redisTemplate.setKeySerializer(new StringRedisSerializer());
    redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(List.class));
    ValueOperations<String, List<Person>> valueOperations = redisTemplate.opsForValue();
    valueOperations.set("aaa", Arrays.asList(new Person("fsx", 24), new Person("fff", 30)));

    List<Person> p = valueOperations.get("aaa");
    System.out.println(p); //[{name=fsx, age=24}, {name=fff, age=30}]

    List<Person> aaa = (List<Person>) redisTemplate.opsForValue().get("aaa");
    System.out.println(aaa); //[{name=fsx, age=24}, {name=fff, age=30}]
}
```

### GenericJackson2JsonRedisSerializer

优点：

简单，不需要像Jackson2JsonRedisSerializer，还要一个序列化对象的类型，它会将类型信息存入json当中。

缺点：

1.类型信息浪费内存空间；

2.而且Integer和Long直接的类型转换会有bug。

解决方案：[spring-data-redis java.lang.ClassCastException](https://stackoverflow.com/questions/34238153/spring-data-redis-java-lang-classcastexception)

```java
public void contextLoads() {
    redisTemplate.setKeySerializer(new StringRedisSerializer());
    redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
    ValueOperations<String, Long> valueOperations = redisTemplate.opsForValue();
    valueOperations.set("bbb", 1L);
    redisTemplate.opsForValue().set("ccc", Arrays.asList(new Person("fsx", 24), new Person("fff", 30)));

    //Long b = valueOperations.get("bbb"); 
    //转换异常 java.lang.Integer cannot be cast to java.lang.Long
    Number number = valueOperations.get("bbb");
    System.out.println(number.longValue());

    List<Person> c = (List<Person>)redisTemplate.opsForValue().get("ccc");
    System.out.println(c);
}
```

```json
[
  "java.util.Arrays$ArrayList",
  [
    {
      "@class": "cn.gdmcmc.iovs.Person",
      "name": "fsx",
      "age": 24
    },
    {
      "@class": "cn.gdmcmc.iovs.Person",
      "name": "fff",
      "age": 30
    }
  ]
]
```
int和long的json字符串是一样的。
```json
1
```
### FastJsonRedisSerializer

由于Redis的流行，很多第三方组件都提供了对应的序列化器。比较著名的有阿里巴巴的FastJsonRedisSerializer

还好阿里默认已经帮我们实现了基于fastjson的序列化方式，我们都不用自己动手了。  FastJsonRedisSerializer和GenericFastJsonRedisSerializer 和上面一样讲述的一样，FastJsonRedisSerializer需要指定反序列化类型，而GenericFastJsonRedisSerializer则比较通用。

### GenericFastJsonRedisSerializer

GenericFastJsonRedisSerializer和GenericJackson2JsonRedisSerializer类似，只是使用了阿里巴巴的fastjson，作为json序列化工具，而且还解决了它的Integer和Long直接的bug，因为它使用了下面的方式保存long类型。


```json
1L
```

保存数组类型，相对来说也省了一点空间。
```json
[
  {
    "@type": "cn.gdmcmc.iovs.Person",
    "age": 24,
    "name": "fsx"
  },
  {
    "@type": "cn.gdmcmc.iovs.Person",
    "age": 30,
    "name": "fff"
  }
]
```

### KryoRedisSerializer

#### 添加依赖

```xml
<dependency>
    <groupId>com.esotericsoftware</groupId>
    <artifactId>kryo</artifactId>
    <version>4.0.0</version>
</dependency>

<dependency>
    <groupId>de.javakaffee</groupId>
    <artifactId>kryo-serializers</artifactId>
    <version>0.41</version>
</dependency>
```

#### 自定义序列化工具

```java
//线程安全
public class KryoRedisSerializer<T> implements RedisSerializer<T> {

    private ThreadLocal<Kryo> kryos = ThreadLocal.withInitial(() -> new Kryo());

    @Override
    public byte[] serialize(T t) throws SerializationException {
        if(t==null) return null;

        byte[] buffer = new byte[2048];
        Output output = new Output(buffer);
        kryos.get().writeClassAndObject(output, t);
        return output.toBytes();
    }

    @Override
    public T deserialize(byte[] bytes) throws SerializationException {

        if(bytes==null||bytes.length==0) return null;

        Input input = new Input(bytes);
        T t = (T) kryos.get().readClassAndObject(input);
        return t;
    }

}
```



## RedisTemplate使用



### 自动配置

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

   @Bean
   @ConditionalOnMissingBean(name = "redisTemplate")
   public RedisTemplate<Object, Object> redisTemplate(
         RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
      RedisTemplate<Object, Object> template = new RedisTemplate<>();
      template.setConnectionFactory(redisConnectionFactory);
      return template;
   }

   @Bean
   @ConditionalOnMissingBean
   public StringRedisTemplate stringRedisTemplate(
         RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
      StringRedisTemplate template = new StringRedisTemplate();
      template.setConnectionFactory(redisConnectionFactory);
      return template;
   }

}
```



### 自定义配置
```java
@Slf4j
@Configuration
public class RedisConfig {

    @Autowired
    private RedisConnectionFactory redisConnectionFactory;

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();

        redisTemplate.setKeySerializer(RedisSerializer.string());
        redisTemplate.setValueSerializer(RedisSerializer.string());
        redisTemplate.setHashKeySerializer(RedisSerializer.string());
        redisTemplate.setHashValueSerializer(RedisSerializer.string());
        redisTemplate.setConnectionFactory(redisConnectionFactory);

        log.info("Redis配置完成......");
        return redisTemplate;
    }
    
    @Bean
    public HashOperations<String, String, Object> hashOperations(RedisTemplate<String, Object> redisTemplate) {
        return redisTemplate.opsForHash();
    }

    @Bean
    public ListOperations<String, Object> listOperations(RedisTemplate<String, Object> redisTemplate) {
        return redisTemplate.opsForList();
    }

    @Bean
    public SetOperations<String, Object> setOperations(RedisTemplate<String, Object> redisTemplate) {
        return redisTemplate.opsForSet();
    }

    @Bean
    public ZSetOperations<String, Object> zSetOperations(RedisTemplate<String, Object> redisTemplate) {
        return redisTemplate.opsForZSet();
    }
}
```