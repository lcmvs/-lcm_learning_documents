# springboot
### [SpringBoot 2.0启动剖析之@SpringBootApplication](https://www.itmsbx.com/article/12)
> ==包含三个注解：==
- @SpringBootConfiguration
- @EnableAutoConfiguration
- @ComponentScan

**实现**：
```
底层真正实现是从classpath中搜寻所有的META-INF/spring.factories配置文件，
并将其中org.springframework.boot.autoconfigure.EnableutoConfiguration对应的配置项，
通过反射实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，
然后汇总为一个，并加载到IoC容器。
```

### [ SpringBoot 2.0启动剖析之SpringApplication.run](https://www.itmsbx.com/article/13)
[补充Spring Boot 启动流程详解](http://www.importnew.com/24875.html)
#####  1） SpringApplication实例化

```java


    推断应用类型是Standard还是Web 可能会出现三种结果REACTIVE 、NONE、SERVLET ：

     this.webApplicationType = this.deduceWebApplicationType();//该行代码位于构造器中

    查找并加载所有可用初始化器，设置到initializers属性中

     this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));//spring.factories处读取

    找出所有的应用程序监听器，设置到listeners属性中。

     this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));//该行代码位于构造器中

    推断并设置main方法的定义类，意思就是找出运行的主类。

     this.mainApplicationClass = this.deduceMainApplicationClass();//该行代码位于构造器中



以下代码摘自：org.springframework.boot.SpringApplication
 
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}
 
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<String>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}

在该方法中，首先通过调用SpringFactoriesLoader.loadFactoryNames(type, classLoader)
来获取所有Spring Factories的名字，
然后调用createSpringFactoriesInstances方法根据读取到的名字创建对象。
最后会将创建好的对象列表排序并返回。
SpringBoot的autoconfigure依赖包中的META-INF/spring.factories配置文件。
```

#####  2） SpringApplication.run方法调用
```java
public ConfigurableApplicationContext run(String... args) {
        //计时工具
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();

        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();

        // step1  设置java.awt.headless系统属性为true - 没有图形化界面
        this.configureHeadlessProperty();

        //step2  初始化监听器
        SpringApplicationRunListeners listeners = this.getRunListeners(args);

        //step3  启动已准备好的监听器，发布应用启动事件
        listeners.starting();

        Collection exceptionReporters;
        try {

            //step4  根据SpringApplicationRunListeners以及参数来装配环境，如java -jar xx.jar --server.port=8000
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            this.configureIgnoreBeanInfo(environment);

            //step5  打印Banner 就是启动Spring Boot的时候打印在console上的ASCII艺术字体
            Banner printedBanner = this.printBanner(environment);

            //step6  创建Spring ApplicationContext()上下文
            context = this.createApplicationContext();

            //step7  准备异常报告器
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, 
             new Class[]{ConfigurableApplicationContext.class}, context);

            //step8  装配Context，上下文前置处理
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);

            //step9  Spring上下文刷新处理
            this.refreshContext(context);

            //step10  Spring上下文后置结束处理 
            this.afterRefresh(context, applicationArguments);

            // 停止计时器监控
            stopWatch.stop();

            //输出日志记录执行主类名、时间信息
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            //step11  发布应用上下文启动完成事件
            listeners.started(context);

            //step12  执行所有 Runner 运行器
            this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
            this.handleRunFailure(context, var10, exceptionReporters, listeners);
            throw new IllegalStateException(var10);
        }

        try {
            //step13  发布应用上下文就绪事件
            listeners.running(context);

            //step14  返回应用上下文
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }
```


