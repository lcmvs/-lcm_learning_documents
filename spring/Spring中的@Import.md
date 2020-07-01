# @Import注解

[相亲相爱的@Import和@EnableXXX](https://blog.csdn.net/qq_34436819/article/details/101052199)

在springboot中的@EnableAutoConfiguration中有一个@Import(EnableAutoConfigurationImportSelector.class)注解

@Import其实是一种向spring注入bean的注解，他有三种方式向spring中注入bean。

**测试工程：lcm-code**

## 注入方式

### 1.直接注入

```java
public class BeanA {
    public void test(){
        System.out.println("beana-test");
    }
}

@Configuration
@Import(BeanA.class)
public class BeanAImportConfig {
}


@RunWith(SpringRunner.class)
@SpringBootTest
public class LcmCodeApplicationTests {

    @Autowired
    BeanA beana;

    @Test
    public void contextLoads() {
        beana.test();
    }

}
```



### 2.通过ImportSelector接口注入

```java
public class BeanB {

    public void test(){
        System.out.println("beanB-test");
    }

}

public class BeanBA {

    public void test(){
        System.out.println("beanBA-test");
    }

}

public class BeanBImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"com.lcm.learn.lcmcode.importTest.selector.BeanB","com.lcm.learn.lcmcode.importTest.selector.BeanBA"};
    }

}

@Configuration
@Import(BeanBImportSelector.class)
public class BeanBConfig {
}

@RunWith(SpringRunner.class)
@SpringBootTest
public class LcmCodeApplicationTests {

    @Autowired
    BeanB beanB;

    @Autowired
    BeanBA beanBA;

    @Test
    public void testImportSelector() {
        beanB.test();
        beanBA.test();
    }

}
```



### 3.通过ImportBeanDefinitionRegistrar接口注入

```java
public class BeanC {
    public void test(){
        System.out.println("beanC-test");
    }
}

public class BeanCImportRigister implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(BeanC.class);
        AbstractBeanDefinition beanDefinition = beanDefinitionBuilder.getBeanDefinition();
        registry.registerBeanDefinition("beanC",beanDefinition);
    }

}

@Configuration
@Import(BeanCImportRigister.class)
public class BeanCConfig {
}

@RunWith(SpringRunner.class)
@SpringBootTest
public class LcmCodeApplicationTests {

    @Autowired
    BeanC beanC;

    @Test
    public void testImportRigister() {
        beanC.test();
    }

}
```



## 原理解析

在Spring容器自动过程中，会执行refresh()方法，refresh()方法中会调用postProcessBeanFactory()。

在postProcessBeanFactory()方法中又会执行所有BeanFactoryPostProcessor后置处理器。

那么首先就会就会执行到优先级最高的ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry()方法，而在该方法中调用了processConfigBeanDefinitions()方法。




## @Enable系列注解

在实际工作当中，我们经常会碰到带有@Enable前缀的注解，通常我们称之为开启某某功能。

例如@EnableAsync（开启@Async注解实现异步执行的功能）、@EnableScheduling（开启@Scheduling注解实现定时任务的功能）、@EnableTransactionManagement（开启事物管理的功能）等注解。

尤其是现在绝大部分项目中都要SpringBoot框架搭建，接入了SpringCloud等微服务，遇见@Enable系列的注解更是家常便饭，例如：@EnableDiscoveryClient、@EnableFeignClients。既然这么常见，那就很有必要知道@Enable系列注解的原理了。



在SpringBoot中，通常会需要整合第三方jar包，通常我们的做法是先引入一个starter，然后在配置类或者启动类上加一个@EnableXXX，这样就和第三方jar包整合完毕了。阅读完本文，现在应该都知道这个原理了吧。



## @Order`和`@AutoConfigureOrder

本篇主要介绍了网上对`@Order`和`@AutoConfigureOrder`常见的错误使用姿势，并给出了正确的使用 case。

下面用简单的几句话介绍一下正确的姿势

- `@Order`注解不能指定 bean 的加载顺序，它适用于 AOP 的优先级，以及将多个 Bean 注入到集合时，这些 bean 在集合中的顺序
- `@AutoConfigureOrder`指定外部依赖的 AutoConfig 的加载顺序（即定义在`/META-INF/spring.factories`文件中的配置 bean 优先级)，在当前工程中使用这个注解并没有什么鸟用
- 同样的 `@AutoConfigureBefore`和 `@AutoConfigureAfter`这两个注解的适用范围和`@AutoConfigureOrder`一样



## 为什么要用？

答案很明显，灵活。例如第三方jar包，可能在某些项目中并不需要使用该jar包中的某些功能，如果我们直接在类的代码上加上@Component注解，这样在和Spring整合时，首先要确保Spring的ComponentScan注解能扫描到第三方jar包中类所在的包，其次，这样Spring容器启动后，不管用不用这个功能，都会在容器中添加这个Bean，这样就不太合适，所以灵活度不够。而用@EnableXXX注解，就能达到想用就用，不想用就关闭的目的，而且还不需要确保Spring扫描到这个第三方jar包的包名。



## 自定义starter

### spring.factories

springboot再进行refresh操作时会扫描到spring.factories这个文件，将里面的配置类自动加载出来。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=cn.gdmcmc.iovs.authbasecommon.AuthBaseCommonApplication
```

### 配置类

```java
//让这个配置类在RedisAutoConfiguration之前装载
@AutoConfigureBefore(RedisAutoConfiguration.class)
//扫描package下的所有类
@ComponentScan
@MapperScan(basePackages = {"cn.gdmcmc.iovs.authbasecommon.**.mapper","cn.gdmcmc.iovs.authbasecommon.**.dao"})
@Configuration
public class AuthBaseCommonApplication {

}
```

### starter下面的配置类

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

        redisTemplate.setKeySerializer(stringRedisSerializer);
        redisTemplate.setValueSerializer(stringRedisSerializer);
        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        redisTemplate.setHashValueSerializer(stringRedisSerializer);
        redisTemplate.setConnectionFactory(redisConnectionFactory);

        log.info("Redis配置完成......");
        return redisTemplate;
    }

    @Bean
    public ValueOperations<String, String> valueOperations(RedisTemplate<String, String> redisTemplate) {
        return redisTemplate.opsForValue();
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