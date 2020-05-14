

# 生命周期

![img](assets/16c171f5a82a7a7c)



# initialization 和 destroy

## 接口实现

这两个接口都只包含一个方法。通过实现InitializingBean接口的afterPropertiesSet()方法可以在Bean属性值设置好之后做一些操作，实现DisposableBean接口的destroy()方法可以在销毁Bean之前做一些操作。

这种方法比较简单，但是不建议使用。因为这样**会将Bean的实现和Spring框架耦合**在一起。

```java
public class GiraffeService implements InitializingBean,DisposableBean {
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("执行InitializingBean接口的afterPropertiesSet方法");

    }

    @Override
    public void destroy() throws Exception {
        System.out.println("执行DisposableBean接口的destroy方法");
    }
}
```

## 配置实现

Spring允许我们创建自己的init方法和destroy方法，只要在Bean的配置文件中指定init-method和destroy-method的值就可以在Bean初始化时和销毁之前执行一些操作。

需要注意的是自定义的init-method和post-method方法可以抛异常但是不能有参数。

这种方式比较推荐，因为可以自己创建方法，无需将Bean的实现直接依赖于spring的框架。

```java
public class GiraffeService {
    //通过<bean>的destroy-method属性指定的销毁方法
    public void destroyMethod() throws Exception {
        System.out.println("执行配置的destroy-method");
    }

    //通过<bean>的init-method属性指定的初始化方法
    public void initMethod() throws Exception {
        System.out.println("执行配置的init-method");
    }

}
```

```xml
<bean name="giraffeService" class="com.giraffe.spring.service.GiraffeService" init-method="initMethod" destroy-method="destroyMethod">
</bean>
```



## 注解实现

除了配置的方式，Spring也支持用`@PostConstruct`和 `@PreDestroy`注解来指定init和destroy方法。这两个注解均在`javax.annotation`包中。

```java
public class GiraffeService {
    
    @PostConstruct
    public void initPostConstruct(){
        System.out.println("执行PostConstruct注解标注的方法");
    }

    @PreDestroy
    public void preDestroy(){
        System.out.println("执行preDestroy注解标注的方法");
    }

}
```
```xml
<bean class="org.springframework.context.annotation.CommonAnnotationBeanPostProcessor" />
```

# Aware 接口

[Spring4.x高级话题(一):Spring Aware](http://blog.longjiazuo.com/archives/1324)

`Spring`的依赖注入的最大亮点就是你所有的`Bean`对`Spring`容器的存在是没有意识的。即你可以将你的容器替换成别的容器，例如`Goggle Guice`,这时`Bean`之间的耦合度很低。

但是在实际的项目中，我们不可避免的要用到`Spring`容器本身的功能资源，这时候`Bean`必须要意识到`Spring`容器的存在，才能调用`Spring`所提供的资源，这就是所谓的`Spring Aware`。其实`Spring Aware`本来就是`Spring`设计用来框架内部使用的，若使用了`Spring Aware`，你的`Bean`将会和`Spring`框架耦合。

| BeanNameAware                  | 获得到容器中Bean的名称                              |
| ------------------------------ | --------------------------------------------------- |
| BeanFactoryAware               | 获得当前bean factory，这样可以调用容器的服务        |
| ApplicationContextAware*       | 获得当前application context，这样可以调用容器的服务 |
| MessageSourceAware             | 获得message source这样可以获得文本信息              |
| ApplicationEventPublisherAware | 应用事件发布器，可以发布事件                        |
| ResourceLoaderAware            | 获得资源加载器，可以获得外部资源文件                |

`Spring Aware`的目的是为了让`Bean`获得`Spring`容器的服务。因为`ApplicationContext`接口集成了`MessageSource`接口，`ApplicationEventPublisherAware`接口和`ResourceLoaderAware`接口，所以`Bean`继承`ApplicationContextAware`可以获得`Spring`容器的所有服务，但原则上我们还是用到什么接口就实现什么接口。

## 使用示例

```java
@Service
//实现BeanNameAware,ResourceLoaderAware接口，获得Bean名称和资源加载的服务。
public class AwareService implements BeanNameAware,ResourceLoaderAware{

    private String beanName;
    private ResourceLoader loader;

    @Override
    //实现ResourceLoaderAware需要重写setResourceLoader方法。
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.loader = resourceLoader;
    }

    @Override
    //实现BeanNameAware需要重写setBeanName方法。
    public void setBeanName(String name) {
        this.beanName = name;
    }

    public void outputResult(){
        System.out.println("Bean的名称为：" + beanName);

        Resource resource = 
                loader.getResource("classpath:org/light4j/sping4/senior/aware/test.txt");
        try{

            System.out.println("ResourceLoader加载的文件内容为: " + IOUtils.toString(resource.getInputStream()));

           }catch(IOException e){
            e.printStackTrace();
           }
    }
}
```

# BeanPostProcessor

BeanPostProcessor 接口，大家也应该有印象，里面只有两个方法：

```java
@Component
public class SpringLifeCycleProcessor implements BeanPostProcessor {

    @Override
    //初始化之前调用
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if ("annotationBean".equals(beanName)){
            LOGGER.info("SpringLifeCycleProcessor start beanName={}",beanName);
        }
        return bean;
    }

    @Override
    //bean 初始化完成调用
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if ("annotationBean".equals(beanName)){
            LOGGER.info("SpringLifeCycleProcessor end beanName={}",beanName);
        }
        return bean;
    }
}
```





