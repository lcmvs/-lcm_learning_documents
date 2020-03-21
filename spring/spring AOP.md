#  Spring AOP

- 它基于动态代理来实现。默认地，如果使用接口的，用 JDK 提供的动态代理实现，如果没有接口，使用 CGLIB 实现。大家一定要明白背后的意思，包括什么时候会不用 JDK 提供的动态代理，而用 CGLIB 实现。
- Spring 3.2 以后，spring-core 直接就把 CGLIB 和 ASM 的源码包括进来了，这也是为什么我们不需要显式引入这两个依赖
- Spring 的 IOC 容器和 AOP 都很重要，Spring AOP 需要依赖于 IOC 容器来管理。
- 如果你是 web 开发者，有些时候，你可能需要的是一个 Filter 或一个 Interceptor，而不一定是 AOP。
- Spring AOP 只能作用于 Spring 容器中的 Bean，它是使用纯粹的 Java 代码实现的，只能作用于 bean 的方法。
- Spring 提供了 AspectJ 的支持，后面我们会单独介绍怎么使用，一般来说我们用**纯的** Spring AOP 就够了。
- 很多人会对比 Spring AOP 和 AspectJ 的性能，Spring AOP 是基于代理实现的，在容器启动的时候需要生成代理实例，在方法调用上也会增加栈的深度，使得 Spring AOP 的性能不如 AspectJ 那么好。

## 基础概念

1.通知(Advice) 通知定义了在切入点代码执行时间点附近需要做的工作。 Spring支持五种类型的通知： Before(前) org.apringframework.aop.MethodBeforeAdvice After-returning(返回后) org.springframework.aop.AfterReturningAdvice After-throwing(抛出后) org.springframework.aop.ThrowsAdvice Arround(周围) org.aopaliance.intercept.MethodInterceptor Introduction(引入) org.springframework.aop.IntroductionInterceptor

2.连接点(Joinpoint)
程序能够应用通知的一个“时机”，这些“时机”就是连接点，例如方法调用时、异常抛出时、方法返回后等等。

3.切入点(Pointcut)
通知定义了切面要发生的“故事”，连接点定义了“故事”发生的时机，那么切入点就定义了“故事”发生的地点，例如某个类或方法的名称，Spring中允许我们方便的用正则表达式来指定。

4.切面(Aspect)
通知、连接点、切入点共同组成了切面：时间、地点和要发生的“故事”。

5.引入(Introduction)
引入允许我们向现有的类添加新的方法和属性(Spring提供了一个方法注入的功能）。

6.目标(Target)
即被通知的对象，如果没有AOP，那么通知的逻辑就要写在目标对象中，有了AOP之后它可以只关注自己要做的事，解耦合！

7.代理(proxy)
应用通知的对象，详细内容参见设计模式里面的动态代理模式。

8.织入(Weaving)
把切面应用到目标对象来创建新的代理对象的过程，织入一般发生在如下几个时机:
(1)编译时：当一个类文件被编译时进行织入，这需要特殊的编译器才可以做的到，例如AspectJ的织入编译器；
(2)类加载时：使用特殊的ClassLoader在目标类被加载到程序之前增强类的字节代码；
(3)运行时：切面在运行的某个时刻被织入,SpringAOP就是以这种方式织入切面的，原理应该是使用了JDK的动态代理技术。

## 基于注解

```java
@Aspect
@Component
public class DemoAspect {
    @Pointcut("execution(public void com.lcm.springstudy.aopdemo.aop.*.*(..))")
    public void demoPointCut(){}

    @Before("demoPointCut()")
    public void demoBefore(){
        System.out.println("前置。。。");
    }

    @After("demoPointCut()")
    public void demoAfter(){
        System.out.println("后置。。。");
    }
}

//@Before("") 前置通知
//@Around("") 环绕通知
//@AfterReturning("") 正常返回通知
//@AfterThrowing("") 异常返回通知
//@After("") finall通知
```
### 通知类型

[Spring AOP 五大通知类型](https://www.cnblogs.com/chuijingjing/p/9806651.html)

**前置通知[Before advice]**：在连接点前面执行，前置通知不会影响连接点的执行，除非此处抛出异常。 
**环绕通知[Around advice]**：环绕通知围绕在连接点前后，比如一个方法调用的前后。这是最强大的通知类型，能在方法调用前后自定义一些操作。环绕通知还需要负责决定是继续处理join point(调用ProceedingJoinPoint的proceed方法)还是中断执行。 
**正常返回通知[After returning advice]**：在连接点正常执行完成后执行，如果连接点抛出异常，则不会执行。 
**异常返回通知[After throwing advice]**：在连接点抛出异常后执行。 
**返回通知[After (finally) advice]**：在连接点执行完成后执行，不管是正常执行完成，还是抛出异常，都会执行返回通知中的内容。 

### 连接点匹配

[一个很不错的AspectJ的Execution表达式说明](https://blog.csdn.net/kq721/article/details/70255855)

#### execution

[Spring AOP -- execution表达式](https://my.oschina.net/u/1251536/blog/1631705)

##### 表达式示例

```
execution(* com.sample.service.impl..*.*(..))
```

详述：

- execution()，表达式的主体
- 第一个“*”符号，表示返回值类型任意；
- com.sample.service.impl，AOP所切的服务的包名，即我们的业务部分
- 包名后面的“..”，表示当前包及子包
- 第二个“*”，表示类名，*即所有类
- .*(..)，表示任何方法名，括号表示参数，两个点表示任何参数类型

##### 语法格式

```
execution(<修饰符模式>？<返回类型模式><方法名模式>(<参数模式>)<异常模式>?)
```

除了**返回类型模式**、**方法名模式**和**参数模式**外，其它项都是可选的。

##### **方法名定义切点**

```
execution(public * *(..))
```

匹配所有目标类的public方法，第一个*代表方法返回值类型，第二个*代表方法名，而".."代表任意入参的方法；

```
execution(* *To(..))
```

匹配目标类所有以To为后缀的方法。第一个“*”代表任意方法返回类型，而“*To”代表任意以To结尾的方法名。

##### **类定义切点**

```
execution(* com.taotao.Waiter.*(..))
```

匹配Waiter接口的所有方法，第一个“*”代表任意返回类型，“com.taotao.Waiter.*”代表Waiter接口中的所有方法。

```
execution(* com.taotao.Waiter+.*(..))
```

匹配Waiter接口及其所有实现类的方法

##### **包名定义切点**

注意：在包名模式串中，"**.***"表示包下的所有类，而“**..***”表示包、子孙包下的所有类。

```
execution(* com.taotao.*(..))
```

匹配com.taotao包下所有类的所有方法

```
execution(* com.taotao..*(..))
```

匹配com.taotao包及其子孙包下所有类的所有方法，如com.taotao.user.dao,com.taotao.user.service等包下的所有类的所有方法。

```
execution(* com..*.*Dao.find*(..))
```

匹配以com开头的任何包名下后缀为Dao的类，并且方法名以find为前缀，如com.taotao.UserDao#findByUserId()、com.taotao.dao.ForumDao#findById()的方法都是匹配切点。

##### **方法入参定义切点**

切点表达式中方法入参部分比较复杂，可以使用“*”和“..”通配符，其中“*”表示任意类型的参数，而“..”表示任意类型参数且参数个数不限。

```
* joke(String, *)
```

匹配目标类中joke()方法，该方法第一个入参为String类型，第二个入参可以是任意类型

```
execution(* joke(String, int))
```

匹配类中的joke()方法，且第一个入参为String类型，第二个入参 为int类型

```
execution(* joke(String, ..))
```

匹配目标类中joke()方法，该方法第一个入参为String，后面可以有任意个且类型不限的参数

##### 常见的切点表达式

- **匹配方法签名**

```
// 匹配指定包中的所有方法
execution(* com.xys.service.*(..))

// 匹配当前包中的所有public方法
execution(public * UserService.*(..))

// 匹配指定包中的所有public方法，并且返回值是int类型的方法
execution(public int com.xys.service.*(..))

// 匹配指定包中的所有public方法，并且第一个参数是String，返回值是int类型的方法
execution(public int com.xys.service.*(String name, ..))
```

- **匹配类型签名**

```
// 匹配指定包中的所有方法，但不包括子包
within(com.xys.service.*)

// 匹配指定包中的 所有方法，包括子包
within(com.xys.service..*)

// 匹配当前包中的指定类中的方法
within(UserService)

// 匹配一个接口的所有实现类中的实现的方法
within(UserDao+)
```

- **匹配Bean名字**

```
// 匹配以指定名字结尾的bean中的所有方法
bean(*Service)
```

- **切点表达式组合**

```
// 匹配以Service或ServiceImpl结尾的bean
bean(*Service || *ServiceImpl)

// 匹配名字以Service结尾，并且在包com.xys.service中的Bean
bean(*Service) && within(com.xys.service.*)
```

#### within



#### annotation



#### bean



## 拦截器



## 过滤器



## 源码解析

[【Spring源码分析】AOP源码解析（上篇）](https://www.cnblogs.com/xrq730/p/6753160.html) 	

[【Spring源码分析】AOP源码解析（下篇）](https://www.cnblogs.com/xrq730/p/6757608.html)

[Spring AOP 实现原理与 CGLIB 应用](https://www.ibm.com/developerworks/cn/java/j-lo-springaopcglib/)	

## 其他注意

### springboot中的aop

springboot 中 aop默认使用cglib实现，可以通过配置让spring自己判断，当有接口时用jdk实现，无接口使用cglib。

```java
spring:
  aop:
    #默认true,使用cglib动态代理
    proxy-target-class: false      
```

### 统一异常处理

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * Description: 数据校检异常
     *
     * @param e
     * @return
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
        FieldError fieldError = e.getBindingResult().getFieldError();
        log.error(fieldError.getDefaultMessage(), e);
        return Result.error(400,fieldError.getDefaultMessage());
    }

    /**
     * Description: 其他异常
     *
     * @param e
     * @return
     */
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        log.error(e.getMessage(), e);
        return Result.error();
    }

}
```