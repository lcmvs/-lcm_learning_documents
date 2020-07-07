# 开始使用

## 服务提供者

### 依赖

```xml
<!-- Dubbo Spring Boot Starter -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.7</version>
</dependency>

<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.12.0</version>
</dependency>
```

### 属性配置

```yml
server:
  port: 20701

spring:
  profiles:
    active: lcm
  application:
    name: boot-dubbo-provider


dubbo:
  protocol:
    name: dubbo
    port: 20801
  application:
    name: ${spring.application.name}
  registry:
    address: zookeeper://xxxx:xx
  scan:
    base-packages: com.lcm.test.dubbo.bootdubboprovider.service
```

### 被调用接口

```java
public interface HelloService {

    String hello(String name);

}
```

### 被调用实现

```java
@DubboService
public class HelloServiceImpl implements HelloService {

    @Override
    public String hello(String name) {
        return "hi "+name;
    }

}
```

## 服务消费者

依赖同上

### 属性配置

```yml
server:
  port: 20702

spring:
  profiles:
    active: lcm
  application:
    name: boot-dubbo-consumer


dubbo:
  protocol:
    name: dubbo
    port: 20802
  application:
    name: ${spring.application.name}
```

### 远程调用

```java
@RestController
public class HelloController {

    @DubboReference
    HelloService helloService;

    @GetMapping("/hello")
    String hello(String name){
        return helloService.hello(name);
    }

}
```