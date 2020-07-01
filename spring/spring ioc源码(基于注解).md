





[Spring IoC源码分析(注解版) ](https://zengbiaobiao.com/article/detail/28)

# 核心概念

## BeanDefinition

这个是一个接口，它是用来存储Bean定义的一些信息的，比如ClassName，Scope，init-methon，等等。它的实现有RootBeanDefination，AnnotatedGenericBeanDefinition等。
```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

   // 我们可以看到，默认只提供 sington 和 prototype 两种，
   // 还有 request, session, globalSession, application, websocket ，它们属于基于 web 的扩展。
   String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
   String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

   // 比较不重要，直接跳过吧
   int ROLE_APPLICATION = 0;
   int ROLE_SUPPORT = 1;
   int ROLE_INFRASTRUCTURE = 2;

   // 设置父 Bean，这里涉及到 bean 继承，不是 java 继承。请参见附录的详细介绍
   // 一句话就是：继承父 Bean 的配置信息而已
   void setParentName(String parentName);

   // 获取父 Bean
   String getParentName();

   // 设置 Bean 的类名称，将来是要通过反射来生成实例的
   void setBeanClassName(String beanClassName);

   // 获取 Bean 的类名称
   String getBeanClassName();


   // 设置 bean 的 scope
   void setScope(String scope);

   String getScope();

   // 设置是否懒加载
   void setLazyInit(boolean lazyInit);

   boolean isLazyInit();

   // 设置该 Bean 依赖的所有的 Bean，注意，这里的依赖不是指属性依赖(如 @Autowire 标记的)，
   // 是 depends-on="" 属性设置的值。
   void setDependsOn(String... dependsOn);

   // 返回该 Bean 的所有依赖
   String[] getDependsOn();

   // 设置该 Bean 是否可以注入到其他 Bean 中，只对根据类型注入有效，
   // 如果根据名称注入，即使这边设置了 false，也是可以的
   void setAutowireCandidate(boolean autowireCandidate);

   // 该 Bean 是否可以注入到其他 Bean 中
   boolean isAutowireCandidate();

   // 主要的。同一接口的多个实现，如果不指定名字的话，Spring 会优先选择设置 primary 为 true 的 bean
   void setPrimary(boolean primary);

   // 是否是 primary 的
   boolean isPrimary();

   // 如果该 Bean 采用工厂方法生成，指定工厂名称。对工厂不熟悉的读者，请参加附录
   // 一句话就是：有些实例不是用反射生成的，而是用工厂模式生成的
   void setFactoryBeanName(String factoryBeanName);
   // 获取工厂名称
   String getFactoryBeanName();
   // 指定工厂类中的 工厂方法名称
   void setFactoryMethodName(String factoryMethodName);
   // 获取工厂类中的 工厂方法名称
   String getFactoryMethodName();

   // 构造器参数
   ConstructorArgumentValues getConstructorArgumentValues();

   // Bean 中的属性值，后面给 bean 注入属性值的时候会说到
   MutablePropertyValues getPropertyValues();

   // 是否 singleton
   boolean isSingleton();

   // 是否 prototype
   boolean isPrototype();

   // 如果这个 Bean 是被设置为 abstract，那么不能实例化，
   // 常用于作为 父bean 用于继承，其实也很少用......
   boolean isAbstract();

   int getRole();
   String getDescription();
   String getResourceDescription();
   BeanDefinition getOriginatingBeanDefinition();
}
```

### BeanDefinitionHolder

这是BeanDefination的包装类，用来存储BeanDefinition，beanName以及aliases等。

```java
public class BeanDefinitionHolder implements BeanMetadataElement {
    private final BeanDefinition beanDefinition;
    private final String beanName;
    @Nullable
    //别名
    private final String[] aliases;
}
```

## BeanFactory

BeanFactory就是Bean工厂啦，所有的Bean都由BeanFactory统一创建和管理，Spring提供了一个AbstractBeanFactory，它继承了BeanFactory接口，并且实现了大部分BeanFactory的功能。

AbstractBeanFactory有一个非常重要的方法叫getBean()

```java
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
        return doGetBean(name, requiredType, null, false);
    }
```

它是用来根据名字和类型获取Bean对象的。它获取对象的原理是，如果BeanFactory中存在Bean，则从缓存中取Bean，否则创建并返回该Bean，并且将该Bean添加到缓存中(这里指Singleton类型的Bean)。所以确切的讲，Bean是在什么时候创建的，要看什么时候调用getBean方法获取这个Bean，根据Bean定义的不同，getBean方法会在不同时候被调用。

Spring中Bean大致可以分为两类，应用类Bean(被应用程序使用)以及功能性Bean(被Spring使用)。

Spring提供了一些功能性接口如BeanFactoryPostProcessor和BeanPostProcessor，实现了这些接口的Bean可以成为功能性Bean。

如果Bean实现了BeanFactoryPostProcessor，则在refresh方法中调用invokeBeanFactoryPostProcessors方法时创建Bean.

如果实现了BeanPostProcessor方法，则在registerBeanPostProcessors方法中创建Bean。其它的Bean应该属于应用类Bean，在默认的finishBeanFactoryInitialization方法中创建Bean，这在后边还会讲到。

## ApplicationContext

AnnotationConfigApplicationContext 

BeanFactory 

BeanDefinitionRegistry

DefaultListableBeanFactory

这几个类的关系也是很重要的，我们来捋一捋。

AnnotationConfigApplicationContext这个类是基于注解的容器类，它实现了BeanFactory和BeanDefinitionRegistry两个接口，拥有Bean对象的管理和BeanDefinition注册功能。同时这个类拥有一个DefaultListableBeanFactory的对象。

DefaultListableBeanFactory也实现了BeanFactory和BeanDefinitionRegistry接口，拥有Bean对象管理和BeanDefinition注册功能。

AnnotationConfigApplicationContext是委托DefaultListableBeanFactory来实现对象管理和BeanDefinition注册的。这里使用的是代理模式。

# 启动容器

```java
public static void main(String[] args) {
        AbstractApplicationContext context = new AnnotationConfigApplicationContext(SpringConfig.class);
        context.registerShutdownHook();
        HelloService helloService = context.getBean(HelloService.class);
        helloService.sayHello();
    }
```

## 构造函数

```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
        this();
        register(annotatedClasses);
        refresh();
}

public AnnotationConfigApplicationContext() {
        //注册BeanDefinition
        this.reader = new AnnotatedBeanDefinitionReader(this);
    	//用于扫描包
        this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
### AnnotatedBeanDefinitionReader

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
        this.registry = registry;
        this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
        // 注册注解配置类后置处理器 ConfigurationClassPostProcessor
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
```

#### registerAnnotationConfigProcessors

```java
//AnnotationConfigUtils
public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
        // 调用了registerAnnotationConfigProcessors方法
        registerAnnotationConfigProcessors(registry, null);
    }

public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
            BeanDefinitionRegistry registry, @Nullable Object source) {

        DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);    

        Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

        if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
            // 看见没有，在这里注册了ConfigurationClassPostProcessor类。
            RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
            def.setSource(source);
            // 主要是调用了registerPostProcessor来进行注册的。
            beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
        }

        if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
            def.setSource(source);
            // 同时也注册了另一个重要类AutowiredAnnotationBeanPostProcessor        
            beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
        }
        // 其它代码就不看了。省略...

        return beanDefs;
    }

// 就是调用registerBeanDefinition注册一个BeanDefination
private static BeanDefinitionHolder registerPostProcessor(
            BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

        definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        registry.registerBeanDefinition(beanName, definition);
        return new BeanDefinitionHolder(definition, beanName);
    }
```

### register

```java
public void register(Class<?>... annotatedClasses) {
        Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
        this.reader.register(annotatedClasses);
}

public void registerBean(Class<?> annotatedClass) {
    doRegisterBean(annotatedClass, null, null, null);
}


<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, 	@Nullable String name,@Nullable Class<? extends Annotation>[] qualifiers, 	      	      BeanDefinitionCustomizer... definitionCustomizers) {

    // 创建BeanDefinition
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
    // 生成bean name
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);

    // ...省去了其它代码

    // 创建BeanDefinitionHolder，它是BeanDefiniton的一个包装类，包含BeanDefiniton对象和它的名字
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    // 最后调用registerBeanDefinition
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

#### registerBeanDefinition

```java
public static void registerBeanDefinition(
            BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
            throws BeanDefinitionStoreException {

        // Register bean definition under primary name.
        String beanName = definitionHolder.getBeanName();
        // 调用registry的registerBeanDefinition方法，registry是我们传进来的AnnotationConfigApplicationContext，因为它实现了BeanDefinitionRegistry，所以我们进入AnnotationConfigApplicationContext的registerBeanDefinition方法。
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

        // Register aliases for bean name, if any.
        String[] aliases = definitionHolder.getAliases();
        if (aliases != null) {
            for (String alias : aliases) {
                registry.registerAlias(beanName, alias);
            }
        }
}

//父类GenericApplicationContext
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException {

        // 调用beanFactory的registerBeanDefinition。我们之前讲过，AnnotationConfigApplicationContext是委托DefaultListableBeanFactory来注册BeanDefinition和管理Bean的，所以调用了beanFactory的registerBeanDefinition
        this.beanFactory.registerBeanDefinition(beanName, beanDefinition);
    }
```

#### GenericApplicationContext

```java
public GenericApplicationContext() {
  // 前面也讲过，DefaultListableBeanFactory也实现了BeanDefinitionRegistry和BeanFactory接口的。
        this.beanFactory = new DefaultListableBeanFactory();
    }
```

#### DefaultListableBeanFactory

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException {


        //... 省去重复注册代码逻辑以及其它一些逻辑判断，

        //最终执行以下代码，可以看出，最终是将beanDefinition放到beanDefinitionMap中。所有的beanDefinition都会放到这个map中，以beanName为主键。如果一个beanDefinition有多个别名，那个它会被注册多次。
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
```



# refresh

refresh其实是在AbstractApplicationContext类中实现的。AbstractApplicationContext实现了BeanDefinitionRegistry以及BeanFactory的大部分功能，所以其子类如AnnotationConfigApplicationContext或ClassPathXmlApplicationContext只需要实现部分相关功能(如加载resource)即可。

```java
//AbstractApplicationContext
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
            // 设置一些属性值，如开始日期等，不是重点
            prepareRefresh();

            // 获取beanFactory，对于AnnotationConfigApplicationContext而言，其实在其父类GenericApplicationContext的构造函数中就已经创建好了DefaultListableBeanFactory
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // 配置一些必要的对象，如ClassLoader，也不是重点。
            prepareBeanFactory(beanFactory);

            try {
                // BeanFactory后置处理器回调函数，让子类有机会在创建beanFactory之后，有机会对beanFactory做额外处理。模板方法模式的使用。
                postProcessBeanFactory(beanFactory);

                // 调用BeanFactoryPostProcessor的postProcessBeanFactory方法。
                // Spring提供了BeanFactoryPostProcessor接口，用于对BeanFactory进行扩展。如果一个Bean实现了BeanFactoryPostProcessor接口的话，在这里会调用其接口方法。
                // 这个方法和上一个方法的区别在于，上一个方法是调用ApplicationContext子类的postProcessBeanFactory方法，而这个方法是调用所有BeanFactoryPostProcessor实例的postProcessBeanFactory方法。这个方法做了很多事情，是我们要重点分析的对象。
                invokeBeanFactoryPostProcessors(beanFactory);

                // 只是注册BeanPostProcessor，还没调用后置处理器相关的方法。如果Bean实现了BeanPostProcessor接口的话，在这里注册其后置处理器。
                registerBeanPostProcessors(beanFactory);

                // 国际化的东西，不是重点。
                initMessageSource();

                // 我们现在不讨论事件机制，主要关注Bean的加载和创建过程，所以跳过。
                initApplicationEventMulticaster();

                // 好吧，又来一个回调方法，模板方法模式哟。
                onRefresh();

                // 不讨论事件机制，跳过。
                registerListeners();

                // 这又是一个重点方法。这个方法会实例化所有非lazy-init的singleton bean，并执行与Spring Bean生命周期相关的方法。如init-method，*aware接口方法等。
                finishBeanFactoryInitialization(beanFactory);

                // 与事件有关，跳过
                finishRefresh();
            }
        }
}
```

对于ioc容器而言，首先就是执行BeanFactory的后置处理器，最核心的就是ConfigurationClassPostProcessor，1.分析配置类：比如springboot中使用的处理 @ComponentScan，扫描package，如@Component，@Service，@Repository等注解的类，如果是@Configuration，则递归处理，继续分析配置类。将所有都添加了@Bean注解的方法添加到configClass中，这里没有注册，仅仅只是添加。

2.加载BeanDefinition：注册所有的@Bean方法上的类。

注册bean的后置处理器，比如AutowiredAnnotationBeanPostProcessor ，@Autowired就是通过这个bean的后置处理器实现的。

初始化bean，bean的生命周期。

## invokeBeanFactoryPostProcessors

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    
        // 委托给PostProcessorRegistrationDelegate执行，调用以下方法
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
    
        省略......
        }
}
```



### PostProcessorRegistrationDelegate

```java
public static void invokeBeanFactoryPostProcessors(
            ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
        if (beanFactory instanceof BeanDefinitionRegistry) {
            List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

            // 先处理BeanDefinitionRegistryPostProcessor，并调用其postProcessBeanDefinitionRegistry方法。
            // BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor的一个子接口，它允许注册更多的BeanDefinition。我们之前讲过，前面的register方法中，只注册了配置类(添加了@Configuration的类)，其它类的BeabnDifination还没有被注册进来
            String[] postProcessorNames =
                    beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                // 这里会得到一个ConfigurationClassPostProcessor类，这个类的作用很明显，就是要继续注册配置类中要求注册的BeanDefination，如在配置类上添加了@ComponentScan注解，那么会进行包扫描，指定包下的所有添加了@Component,@Service等注解的类全部注册进来。
                // 那么这个类是什么时候注册的呢，我们稍后会讲。
                if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            // 在这里开始调用BeanDefinitionRegistryPostProcessor的后置处理器postProcessBeanDefinitionRegistry方法。
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();

            // 省略大量重复代码，主要处理各种优先级的BeanFactoryPostProcessor对象。可以为BeanFactoryPostProcessor设置优先级，优先级高的对象会先被调用。

            // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
            invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
            invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
        }

        else {
            // Invoke factory processors registered with the context instance.
            invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);


        // 对所有的postProcessor进行分类，高优先级的和正常优先级的分别放在不同的列表中，高优先级的先调用其postProcessBeanFactory方法。        
        List<String> nonOrderedPostProcessorNames = new ArrayList<>();
        for (String ppName : postProcessorNames) {
            if (processedBeans.contains(ppName)) {
                // skip - already processed in first phase above
            }
            else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            }
            else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                orderedPostProcessorNames.add(ppName);
            }
            else {
                nonOrderedPostProcessorNames.add(ppName);
            }
        }


        // 此处省略重复调用代码，就是对高优先级的BeanFactoryPostProcessor先执行invokeBeanFactoryPostProcessors方法，然后是低优先级，再之后是无优先级的
        List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
        for (String postProcessorName : nonOrderedPostProcessorNames) {
            // 这里这行代码，它调用了beanFactory.getBean方法。我们知道只要调用了getBean方法，如果Bean没有被加载，Spring就会创建该Bean，并对其进行相关属性复制。所以对于实现了BeanFactoryPostProcessor的Bean，在这里就已经进行实例化了。因为它要拿到当前对象，才能执行其接口方法。
            nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        // 开始进行回调。
        invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);        
        beanFactory.clearMetadataCache();
    }
```



```java
// 接着PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors的代码分析，会调用以下方法
private static void invokeBeanDefinitionRegistryPostProcessors(
            Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

        for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
            // 这里的postProcessor就是ConfigurationClassPostProcessor
            postProcessor.postProcessBeanDefinitionRegistry(registry);
        }
    }
```

## registerBeanPostProcessors

registerBeanPostProcessors方法负责初始化实现了BeanPostProcessor接口的Bean，并将其注册到BeanFactory中。

后置处理器BeanPostProcessor定义了两个方法postProcessBeforeInitialization和postProcessAfterInitialization。Spring会添加所有的BeanPostProcessor到BeanFactory的beanPostProcessors列表中。

BeanPostProcessor可以理解成辅助类，在所有其它类型的Bean(应用Bean)初始化过程中，Spring会分别在它们初始化前后调用BeanPostProcessor实例的postProcessBeforeInitialization和postProcessAfterInitialization。

前面讲的BeanFactoryPostProcessor，和BeanPostProcessor是一样的。在BeanFactory创建之后，Spring会调用所有BeanFactoryPostProcessor实例的postProcessBeanFactory方法，这样为用户提供了在BeanFactory创建之后，对Bean进行扩展的机会。

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        // 调用PostProcessorRegistrationDelegate的registerBeanPostProcessors方法
        PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

### registerBeanPostProcessors

```java
public static void registerBeanPostProcessors(
            ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
        // 获取所有的postProcessor names
        String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);


        List<String> nonOrderedPostProcessorNames = new ArrayList<>();
        for (String ppName : postProcessorNames) {
            // 放到nonOrderedPostProcessorNames列表中，这里省略了其它代码，都是处理优先级的。BesnPostProcessor也可以设置优先级的，优先级高的会先被调用。                        
            nonOrderedPostProcessorNames.add(ppName);
        }

        // Now, register all regular BeanPostProcessors.
        List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
        for (String ppName : nonOrderedPostProcessorNames) {
            // 实例化BeanPostProcessor实例
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            nonOrderedPostProcessors.add(pp);
        }

        // 注册BestProcessor
        registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
}

private static void registerBeanPostProcessors(
            ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {

        // 把所有的BeanPostProcessor添加到beanFactory中
        for (BeanPostProcessor postProcessor : postProcessors) {
            beanFactory.addBeanPostProcessor(postProcessor);
        }
}
```

#### addBeanPostProcessor

```java
//DefaultListableBeanFactory
public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {        
    // Remove from old position, if any
    this.beanPostProcessors.remove(beanPostProcessor);        
    // 将beanPostProcessor添加到beanPostProcessors列表中
    this.beanPostProcessors.add(beanPostProcessor);
}
```

## finishBeanFactoryInitialization

初始化Bean对象

refresh方法中我们最后要讲的一个方法是finishBeanFactoryInitialization，它负责根据BeanDefinition创建Bean对象，并执行一些与Bean生命周期相关的回调函数。我们直接看源码：

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        // 省略其它处理代码，直接调用beanFactory的preInstantiateSingletons.
        beanFactory.preInstantiateSingletons();
}
```

### preInstantiateSingletons

```java
//DefaultListableBeanFactory
// 这里要注意，只初始化所有singletons的Bean,关于prototype类型的bean，是每次调用getBean都会创建的，不会在容器启动的时候初始化。
public void preInstantiateSingletons() throws BeansException {        
        List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);        
        for (String beanName : beanNames) {
            // 这里省略了FactoryBean对象处理逻辑，如果是isEglarBean，也会调用getBean方法。所以简单理解为，不管是哪种sigleton bean，都会调用getBean方法
            getBean(beanName);
        }

        // 省略SmartInitializingSingleton的处理，有兴趣的可以去了解一下这个类的作用...        
    }

public Object getBean(String name) throws BeansException {
        // getBean其实调用了doGetBean方法。
        return doGetBean(name, null, null, false);
    }
```

### doGetBean

```java
final String beanName = transformedBeanName(name);
        Object bean;

        // 取单例Bean，这里简单介绍一下DefaultListableBeanFactory的另一种集成关系，它继承了DefaultSingletonBeanRegistry类。这个类允许单例管理，简单讲就是其中定义了一个HashMap<String,Object>，维护了bean名字和Bean对象的关系，并且保证每一个Bean名字只创建一个Bean对象。
        Object sharedInstance = getSingleton(beanName);
        if (sharedInstance != null && args == null) {
            // 如果缓存(上面说的HashMap)中存在bean，则取这个bean
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }

        else {
        // BeanFactory是可以继承的，如果当前BeanFactory中找不到Bean定义，就从父BeanFactory中去找。
            BeanFactory parentBeanFactory = getParentBeanFactory();
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // Not found -> check parent.
                String nameToLookup = originalBeanName(name);
                // 省略了参数处理的过程。直接从父BeanFactory去找。
                return (T) parentBeanFactory.getBean(nameToLookup);
            }            

            try {
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                // 这里先处理Bean依赖，如果当前Bean依赖其它Bean，则先注册并实例化依赖的Bean.
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        // 先注册依赖Bean
                        registerDependentBean(dep, beanName);
                        // 再实例化依赖Bean
                        getBean(dep);
                    }
                }

                // Create bean instance.
                if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, () -> {
                        // 创建单例Bean
                        return createBean(beanName, mbd, args);

                    });
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }

                else if (mbd.isPrototype()) {
                    // Prototyp类型的Bean，是每次都会创建的。
                    Object prototypeInstance = null;
                    try {
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }

                else {
                    // 处理其它类型的Bean,如一些扩展的类型Session，Request等等，代码省略了..
                }
            }
            catch (BeansException ex) {
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            }
        }

        // 类型转换的代码也省略了...
        return (T) bean;
```

### createBean

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args){    
        // 其它代码都省略了，就是调用了doCreateBean方法
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        return beanInstance;
}

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
            throws BeanCreationException {

        // 创建BeanWrapper.
        BeanWrapper instanceWrapper = null;        
        if (instanceWrapper == null) {
            // 调用createBeanInstance创建Bean实例，这个方法我们不再仔细往下阅读了，大致的思路是根据RootBeanDefinition找到其类型classType，然后再获取其构造函数，然后根据构造函数使用反射动态创建出一个bean实例，再对这个实例进行一下包装，返回BeanWrapper对象。当需要对bean增强的时候，创建bean也可能使用CGLib动态创建，我们这里只讲最简单的情况。
            instanceWrapper = createBeanInstance(beanName, mbd, args);
        }
        final Object bean = instanceWrapper.getWrappedInstance();        


        Object exposedObject = bean;
        try {
            // 根据BeanDefinition对bean属性进行赋值。
            populateBean(beanName, mbd, instanceWrapper);
         // 初始化Bean,在这里执行Bean生命周期的回调函数，如设置beanName，调用BeanPostProcessor等。
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
        catch (Throwable ex) {            
        }

        return exposedObject;
}
```

上面的两个重要方法分别是createBeanInstance和initializeBean。其中createBeanInstance所做的事情大致是根据RootBeanDefinition找到其类型classType，然后再获取其构造函数，然后根据构造函数使用反射动态创建出一个bean实例，再对这个实例进行一下包装，返回BeanWrapper对象。因为这个类的业务很简单，大部分代码都是在处理一些与反射有关的东西，所以我们不再进行分析了。我们把重点放在initializeBean这个方法上

#### initializeBean

初始化bean，bean的生命周期主要就在这个类

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
        // 执行*Aware接口的方法，Spring提供了大量的*Aware接口，用来给Bean设置值，如可以向Bean中注入ApplicationContext，ClassLoader等。    
        invokeAwareMethods(beanName, bean);

        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            //如果Bean实现了BeanPostProcessor接口，则执行postProcessBeforeInitialization方法。
            wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
        }

        try {
            // 执行bean自定义初始化相关方法。@PostConstruct、InitializingBean、init-method
            invokeInitMethods(beanName, wrappedBean, mbd);
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                    (mbd != null ? mbd.getResourceDescription() : null),
                    beanName, "Invocation of init method failed", ex);
        }
        if (mbd == null || !mbd.isSynthetic()) {
            // 如果Bean实现了BeanPostProcessor接口，则执行postProcessAfterInitialization方法。
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
    }
```

# 后置处理器

## BeanFactory后置处理器

BeanFactoryPostProcessor接口，它叫BeanFactory后置处理器。如果你的类实现了这个接口，那么在BeanFactory创建完成之后，Spring会调用这个类的postProcessBeanFactory方法。

这个类为用户提供了扩展BeanFactory的机会。

## Bean后置处理器

BeanPostProcessor也是Spring提供的另一个有用接口，所有的BeanPostProcessor会被添加到Spring容器列表中。

在应用Bean被实例化前后，Spring会一次调用BeanPostProcessor列表中的接口方法。后边会有具体分析。

所有实现了BeanFactoryPostProcessor和BeanPostProcessor接口的Bean，理论上是提供给Spring使用的，我们叫他功能性Bean。而其它Bean，如UserService，叫应用型Bean。

比如 AutowiredAnnotationBeanPostProcessor ，实现@Autowired的关键。

# ConfigurationClassPostProcessor

[Spring源码研究之@Configuration](https://blog.csdn.net/lqzkcx3/article/details/78409305)

ConfigurationClassPostProcessor也是一个BeanFactory后置处理器。

这个类是AnnotationConfigApplicationContext构造函数中，帮我们注册进来的。

ApplicationContext的中refresh中调用invokeBeanFactoryPostProcessors时，就会使用这个后置处理器。

分析配置类：比如springboot中使用的处理 @ComponentScan，扫描package，如@Component，@Service，@Repository等注解的类，如果是@Configuration，则递归处理，继续分析配置类。将所有都添加了@Bean注解的方法添加到configClass中，这里没有注册，仅仅只是添加。

加载BeanDefinition：注册所有的@Bean方法上的类。

其作用是用于注册用户自定义的类。

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        // 开始注册Config类中定义的BeanDefinition
        processConfigBeanDefinitions(registry);
}

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
        String[] candidateNames = registry.getBeanDefinitionNames();

        // 将配置类放在configCandidates列表中。代码省略...

        // 创建配置类的分析器，对配置类进行分析。配置类就是我们添加了@Configuration注解的类，可以有多个的。我写的demo中只有一个，叫SpringConfig。
        ConfigurationClassParser parser = new ConfigurationClassParser(
                this.metadataReaderFactory, this.problemReporter, this.environment,
                this.resourceLoader, this.componentScanBeanNameGenerator, registry);

        Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
        do {
            // 开始分析配置类
            parser.parse(candidates);
            parser.validate();

            // Read the model and create bean definitions based on its content
            if (this.reader == null) {
                this.reader = new ConfigurationClassBeanDefinitionReader(
                        registry, this.sourceExtractor, this.resourceLoader, this.environment,
                        this.importBeanNameGenerator, parser.getImportRegistry());
            }
            // 加载BeanDefinition
            this.reader.loadBeanDefinitions(configClasses);
        }
        while (!candidates.isEmpty());
    }
```

## 分析配置类

### parse

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
        for (BeanDefinitionHolder holder : configCandidates) {
            // 调用链往下看
            parse(bd.getBeanClassName(), holder.getBeanName());
        }
        this.deferredImportSelectorHandler.process();
    }

protected final void parse(Class<?> clazz, String beanName) throws IOException {
        // 继续往下看
        processConfigurationClass(new ConfigurationClass(clazz, beanName));
    }

protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {    

        // Recursively process the configuration class and its superclass hierarchy.
        SourceClass sourceClass = asSourceClass(configClass);
        do {
            // 最终调到了这个方法，处理配置类
            sourceClass = doProcessConfigurationClass(configClass, sourceClass);
        }
        while (sourceClass != null);

        this.configurationClasses.put(configClass, configClass);
    }
```

#### doProcessConfigurationClass

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
            throws IOException {

        // 这两个方法简单过，就是处理配置类里边的内部类和@PropertySource注解
        processMemberClasses(configClass, sourceClass);
        processPropertySource(propertySource);        

        // 处理 @ComponentScan 注解类，重点，先获取加了@ComponentScan注解的配置类
        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
                sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        if (!componentScans.isEmpty() &&
                !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
            for (AnnotationAttributes componentScan : componentScans) {
// 开始扫描指定package的Bean，那些加了@Component，@Service，@Repository等注解的类，都会加进来。
                Set<BeanDefinitionHolder> scannedBeanDefinitions =
                        this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
                // Check the set of scanned definitions for any further config classes and parse recursively if needed
                for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                    // 如果扫描到了配置类，则递归处理，继续分析配置类。
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        // 这个方法我们就不分析了，递归哈
                        parse(bdCand.getBeanClassName(), holder.getBeanName());
                    }
                }
            }
        }

        // Process any @Import annotations
        processImports(configClass, sourceClass, getImports(sourceClass), true);        

        // Process individual @Bean methods
        // 将所有都添加了@Bean注解的方法添加到configClass中，这里没有注册，仅仅只是添加。后边有一个叫reader.loadBeanDefinitions的方法来处理它。
        Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
        for (MethodMetadata methodMetadata : beanMethods) {
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        }

        // Process default methods on interfaces
        processInterfaces(configClass, sourceClass);        

        // No superclass -> processing is complete
        return null;
    }
```

#### componentScanParser.parse

```java
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
        // 创建scanner
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
                componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
        Set<String> basePackages = new LinkedHashSet<>();
        // 添加所有的package，代码省略...
        // 开始扫描
        return scanner.doScan(StringUtils.toStringArray(basePackages));
    }
```

#### scanner.doScan

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
        for (String basePackage : basePackages) {
            // 找出当前包下面的所有符合条件的类，这个方法我们就不分析了。
            Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
            for (BeanDefinition candidate : candidates) {
                // 处理一些其他情况，也不说了...
                if (candidate instanceof AbstractBeanDefinition) {
                    postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
                }
                if (candidate instanceof AnnotatedBeanDefinition) {
                    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
                }
                if (checkCandidate(beanName, candidate)) {
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    definitionHolder =
                            AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                    beanDefinitions.add(definitionHolder);
                    // 最后注册BeanDefinition，这样就将包下的所有BeanDefintion都注册完成了
                    registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }
        return beanDefinitions;
    }
```

## 注册BeanDefinition

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
    TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
    for (ConfigurationClass configClass : configurationModel) {
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
    }
}

private void loadBeanDefinitionsForConfigurationClass(
            ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
        // 处理Imported的配置类
        if (configClass.isImported()) {
            registerBeanDefinitionForImportedConfigurationClass(configClass);
        }
        for (BeanMethod beanMethod : configClass.getBeanMethods()) {
            // 注册所有的@Bean方法上的类，重点看一下这个方法
            loadBeanDefinitionsForBeanMethod(beanMethod);
        }

        // 处理其它注解，这里就不分析了。
        loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());           loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
    }
```

### loadBeanDefinitionsForBeanMethod

```java
private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
        ConfigurationClass configClass = beanMethod.getConfigurationClass();        
        // 一些别名处理的代码都省略了...

        //  创建BeanDefinition
        ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);

        // 处理initMethod和destroyMethod
        String initMethodName = bean.getString("initMethod");
        if (StringUtils.hasText(initMethodName)) {
            beanDef.setInitMethodName(initMethodName);
        }

        String destroyMethodName = bean.getString("destroyMethod");
        beanDef.setDestroyMethodName(destroyMethodName);        

        // Replace the original bean definition with the target one, if necessary
        BeanDefinition beanDefToRegister = beanDef;
        // 最后注册这个BeanDefinition
        this.registry.registerBeanDefinition(beanName, beanDefToRegister);
    }
```

## 