# nacos学习

[Nacos](https://github.com/alibaba/Nacos) 是阿里巴巴开源的一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

关于 Nacos Spring Cloud 的详细文档请参看：[Nacos Config](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/wiki/Nacos-config) 和 [Nacos Discovery](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/wiki/Nacos-discovery)。

- 通过 Nacos Server 和 spring-cloud-starter-alibaba-nacos-config 实现配置的动态变更。
- 通过 Nacos Server 和 spring-cloud-starter-alibaba-nacos-discovery 实现服务的注册与发现。

## Nacos Server安装使用

1.直接下载：[Nacos Server 下载页](https://github.com/alibaba/nacos/releases)

2.启动 Server，进入解压后文件夹或编译打包好的文件夹，找到如下相对文件夹 nacos/bin，并对照操作系统实际情况之下如下命令。

1. Linux/Unix/Mac 操作系统，执行命令 `sh startup.sh -m standalone`
2. Windows 操作系统，执行命令 `cmd startup.cmd`

## Nacos作为注册中心

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

## Nacos作为配置中心

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

## ACM

Spring Cloud AliCloud ACM 是阿里云提供的商业版应用配置管理(Application Configuration Management) 产品 在 Spring Cloud 应用侧的客户端实现，且目前完全免费。

外网无法访问，只有阿里云的云服务器上才能直接访问。只有配置中心功能，没有服务管理功能。