# Spring AOP

spring aop就是面向切面编程，aop可以通过动态代理或者aspectj实现。

aspectj实现aop需要另外一个编译攻击ajc。

spring只是使用了aspectj的注解，没有使用aspectj的编译期和织入器。

### 1.注解使用

@Aspect
@Component
public class DemoAspect {

```java
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
```

}

##### 1.1 通知类型

**前置通知[Before advice]**：在连接点前面执行，前置通知不会影响连接点的执行，除非此处抛出异常。 
**环绕通知[Around advice]**：环绕通知围绕在连接点前后，比如一个方法调用的前后。这是最强大的通知类型，能在方法调用前后自定义一些操作。环绕通知还需要负责决定是继续处理join point(调用ProceedingJoinPoint的proceed方法)还是中断执行。 
**正常返回通知[After returning advice]**：在连接点正常执行完成后执行，如果连接点抛出异常，则不会执行。 
**异常返回通知[After throwing advice]**：在连接点抛出异常后执行。 
**返回通知[After (finally) advice]**：在连接点执行完成后执行，不管是正常执行完成，还是抛出异常，都会执行返回通知中的内容。 

[Spring AOP 五大通知类型](https://www.cnblogs.com/chuijingjing/p/9806651.html)


##### 1.2 execution表达式

[Spring AOP -- execution表达式](https://my.oschina.net/u/1251536/blog/1631705)

[一个很不错的AspectJ的Execution表达式说明](https://blog.csdn.net/kq721/article/details/70255855)


### 2.源码解析

[【Spring源码分析】AOP源码解析（上篇）](https://www.cnblogs.com/xrq730/p/6753160.html) 	

[【Spring源码分析】AOP源码解析（下篇）](https://www.cnblogs.com/xrq730/p/6757608.html)

[Spring AOP 实现原理与 CGLIB 应用](https://www.ibm.com/developerworks/cn/java/j-lo-springaopcglib/)	

### 3.其他注意

##### 3.1 springboot中的aop

springboot 中 aop默认使用cglib实现，可以通过配置让spring自己判断，当有接口时用jdk实现，无接口使用cglib。

```java
spring:
  aop:
    #默认true,使用cglib动态代理
    proxy-target-class: false      
```

##### 3.2 springboot controller层统一异常处理

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