个人学习，文档记录

# 近期学习计划

spring源码
netty源码
算法
springcloud

熔断限流

tomcat源码
zookeeper
Kafka
分布式

# JDK

## 集合

ArrayList、LinkedList、ArrayDeque 实现细节？各自优劣？

TreeMap数据结构？红黑树时间复杂度？

PriorityQueue堆排序？

### HashMap

扩容resize()实现细节？

### ConcurrentHashMap

Hashtable和Collections.synchronizedMap实现？

put和remove实现？

get实现？

扩容实现？

size()实现？

## 多线程

### Thread

线程和进程区别？

sleep和wait区别？

线程状态？

什么是内存可见性？

### synchronized

jdk1.6优化？偏向锁、轻量级锁、重量级锁？

对象头结构？

### lock

加锁，释放锁实现细节？可重入性实现？

乐观锁和悲观锁？

公平锁和非公平锁？

node结构?waitStatus？

condition条件队列

tryLock(timeout)实现细节？

### 共享锁



### ThreadLocal

弱引用？自动回收？内存泄漏？

### 线程池

线程池参数以及意义？
定时任务？
任务拒绝策略？
tomcat线程池的不同点？



# JVM

## 运行时数据区域

堆和直接内存？

直接内存如何垃圾回收？

## 类加载



## 垃圾回收

如何判断对象是否垃圾回收？四种引用类型？

gc root是什么？

垃圾回收算法？

垃圾回收器？

## jvm参数



## jvm工具



# spring

## spring ioc

ioc容器整个执行流程？(基于注解) 源码分析？

BeanFactory和ApplicationContext

BeanFactory后置处理器：BeanFactoryPostProcessor？ConfigurationClassPostProcessor

Bean后置处理器：BeanPostProcessor？


@Configuration中的@Bean和@Import？


@Bean和@Component区别？

bean生命周期？

## spring aop

aop是什么？有什么好处？应用场景？

aop源码？

## spring mvc

springmvc工作流程？

如何使用xml替换json？

springmvc和springwebflux

## spring boot

springboot注解的意义？自动配置是如何实现的？

# spring cloud

## 注册中心

cap理论？

nacos、eureka和zookeeper区别？

fegin，rebbon，负载均衡？

## 熔断限流

sentinel

## 网关

spring cloud gateway和zuul区别？



# mybatis

jdbc流程？

resultType和resultMap区别？

动态代理，jdk和cglib？

源码分析？

mybatis缓存？

# netty

为什么使用netty？为什么性能好？

## NIO

bio、nio、aio？

channel

buffer

selector

## 线程模型

单线程、线程池、两个线程池

## channel

## ByteBuf

## PooledByteBuf

### 对象池

#### Recycler

异步回收对象、同步回收对象、同线程获取对象、异线程获取对象

#### FastThreadLocal

为什么快？如何使用？

和jdk的ThreadLocal比较？

### 内存池

PooledByteBufAllocator 初始化默认值

[PoolThreadCache](https://www.jianshu.com/p/9177b7dabd37) 线程私有缓存

[PoolArena](https://www.jianshu.com/p/86fbacdb68bd)

[PoolChunkList](https://www.jianshu.com/p/2b8375df2d1a) 使用率流转回收

[PoolChunk](https://www.jianshu.com/p/70181af2972a) 伙伴分配算法

[PoolSubpage](https://www.jianshu.com/p/7afd3a801b15) 位图算法



# redis

redis数据结构类型？内部原理？

redis为什么快？

哨兵？集群方案？cap理论？

持久化方案？

缓存雪崩、击穿、穿透？

spring中配置使用？客户端工具？

# MySQL

MySQL三层架构？

MySQL存储引擎？myisam和innodb

事务的特性？acid

隔离级别？锁？快照读和当前读？间隙锁？

表锁和行锁？意向锁？

隐式锁定和显示锁定？

索引？聚簇索引？二级索引？覆盖索引？回表？

sql优化？explain？

spring中的回滚规则？隔离级别？传播行为？



# http

http1.0、http1.1、http2

https

okhttp线程池、连接池



# 设计模式

单例模式
简单工厂模式
工厂方法模式
抽象工厂模式
建造者模式
原型模式
模板方法模式
代理模式
享元模式
装饰器模式
适配器模式
迭代器模式
观察者模式
责任链模式

# 分布式

cap理论？分布式一致性问题？

2pc，3pc

分布式事务一致性解决方案：tcc

## zookeeper

zookeeper原理？

api？客户端工具？

一致性协议：2pc，3pc，

zab协议：消息广播，崩溃恢复

## Kafka

为什么使用Kafka？

什么是Producer、Consumer、Broker、Topic、Partition？

Kafka为什么高性能？顺序io？

生产者可靠性保证？生产丢失

消费者可靠性？重复消费

负载均衡？

Kafka 如何保证消息的消费顺序？

# 系统设计

## 秒杀系统设计

# 数据结构与算法

## 数据结构

数组
链表
栈和队列
map和set
树，二叉树，二叉排序树，平衡二叉树、红黑树

## 算法

排序算法
回溯算法
贪心算法
动态规划



