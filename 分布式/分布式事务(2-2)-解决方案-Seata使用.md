# 单机部署

打开 [Seata 下载页面](https://github.com/seata/seata/releases)，选择想要的 Seata 版本。

```Bash
# 创建目录
$ mkdir -p /Users/yunai/Seata
$ cd /Users/yunai/Seata

# 下载
$ wget https://github.com/seata/seata/releases/download/v1.1.0/seata-server-1.1.0.tar.gz

# 解压
$ tar -zxvf seata-server-1.1.0.tar.gz

# 查看目录
$ cd seata
$ ls -ls
24 -rw-r--r--    1 yunai  staff  11365 May 13  2019 LICENSE
 0 drwxr-xr-x    4 yunai  staff    128 Apr  2 07:46 bin # 执行脚本
 0 drwxr-xr-x    9 yunai  staff    288 Feb 19 23:49 conf # 配置文件
 0 drwxr-xr-x  138 yunai  staff   4416 Apr  2 07:46 lib #  seata-*.jar + 依赖库 
```

执行 `nohup sh bin/seata-server.sh &` 命令，启动 TC Server 在后台。在 `nohup.out` 文件中，我们看到如下日志，说明启动成功：

```Java
# 使用 File 存储器
2020-04-02 08:36:01.302 INFO [main]io.seata.common.loader.EnhancedServiceLoader.loadFile:247 -load TransactionStoreManager[FILE] extension by class[io.seata.server.store.file.FileTransactionStoreManager]
2020-04-02 08:36:01.302 INFO [main]io.seata.common.loader.EnhancedServiceLoader.loadFile:247 -load SessionManager[FILE] extension by class[io.seata.server.session.file.FileBasedSessionManager]
# 启动成功
2020-04-02 08:36:01.597 INFO [main]io.seata.core.rpc.netty.RpcServerBootstrap.start:155 -Serve
```

- 默认配置下，Seata TC Server 启动在 **8091** 端点。

因为我们使用 file 模式，所以可以看到用于持久化的本地文件 `root.data`。操作命令如下：

# 集群部署





# 属性

## 公共部分

| key                     | desc                           | remark                                                       |
| ----------------------- | ------------------------------ | ------------------------------------------------------------ |
| transport.serialization | client和server通信编解码方式   | seata（ByteBuf）、protobuf、kryo、hession，默认seata         |
| transport.compressor    | client和server通信数据压缩方式 | none、gzip，默认none                                         |
| transport.heartbeat     | client和server通信心跳检测开关 | 默认true开启                                                 |
| registry.type           | 注册中心类型                   | 默认file，支持file 、nacos 、eureka、redis、zk、consul、etcd3、sofa、custom |
| config.type             | 配置中心类型                   | 默认file，支持file、nacos 、apollo、zk、consul、etcd3、custom |

## server端

| key                                       | desc                                             | remark                                                       |
| ----------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| server.undo.logSaveDays                   | undo保留天数                                     | 默认7天,log_status=1（附录3）和未正常清理的undo              |
| server.undo.logDeletePeriod               | undo清理线程间隔时间                             | 默认86400000，单位毫秒                                       |
| server.maxCommitRetryTimeout              | 二阶段提交重试超时时长                           | 单位ms,s,m,h,d,对应毫秒,秒,分,小时,天,默认毫秒。默认值-1表示无限重试。公式: timeout>=now-globalTransactionBeginTime,true表示超时则不再重试 |
| server.maxRollbackRetryTimeout            | 二阶段回滚重试超时时长                           | 同commit                                                     |
| server.recovery.committingRetryPeriod     | 二阶段提交未完成状态全局事务重试提交线程间隔时间 | 默认1000，单位毫秒                                           |
| server.recovery.asynCommittingRetryPeriod | 二阶段异步提交状态重试提交线程间隔时间           | 默认1000，单位毫秒                                           |
| server.recovery.rollbackingRetryPeriod    | 二阶段回滚状态重试回滚线程间隔时间               | 默认1000，单位毫秒                                           |
| server.recovery.timeoutRetryPeriod        | 超时状态检测重试线程间隔时间                     | 默认1000，单位毫秒，检测出超时将全局事务置入回滚会话管理器   |
| store.mode                                | 事务会话信息存储方式                             | file本地文件(不支持HA)，db数据库(支持HA)                     |
| store.file.dir                            | file模式文件存储文件夹名                         | 默认sessionStore                                             |
| store.db.datasource                       | db模式数据源类型                                 | 默认dbcp                                                     |
| store.db.dbType                           | db模式数据库类型                                 | 默认mysql                                                    |
| store.db.driverClassName                  | db模式数据库驱动                                 | 默认com.mysql.jdbc.Driver                                    |
| store.db.url                              | db模式数据库url                                  | 默认jdbc:mysql://127.0.0.1:3306/seata                        |
| store.db.user                             | db模式数据库账户                                 | 默认mysql                                                    |
| store.db.password                         | db模式数据库账户密码                             | 默认mysql                                                    |
| store.db.minConn                          | db模式数据库初始连接数                           | 默认1                                                        |
| store.db.maxConn                          | db模式数据库最大连接数                           | 默认3                                                        |
| store.db.maxWait                          | db模式获取连接时最大等待时间                     | 默认5000，单位毫秒                                           |
| store.db.globalTable                      | db模式全局事务表名                               | 默认global_table                                             |
| store.db.branchTable                      | db模式分支事务表名                               | 默认branch_table                                             |
| store.db.lockTable                        | db模式全局锁表名                                 | 默认lock_table                                               |
| store.db.queryLimit                       | db模式查询全局事务一次的最大条数                 | 默认100                                                      |
| metrics.enabled                           | 是否启用Metrics                                  | 默认false关闭，在False状态下，所有与Metrics相关的组件将不会被初始化，使得性能损耗最低 |
| metrics.registryType                      | 指标注册器类型                                   | Metrics使用的指标注册器类型，默认为内置的compact（简易）实现，这个实现中的Meter仅使用有限内存计数，性能高足够满足大多数场景；目前只能设置一个指标注册器实现 |
| metrics.exporterList                      | 指标结果Measurement数据输出器列表                | 默认prometheus，多个输出器使用英文逗号分割，例如"prometheus,jmx"，目前仅实现了对接prometheus的输出器 |
| metrics.exporterPrometheusPort            | prometheus输出器Client端口号                     | 默认9898                                                     |

## client端

| key                                                | desc                                        | remark                                                       |
| -------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------ |
| seata.enabled                                      | 是否开启spring-boot自动装配                 | true、false,(SSBS)专有配置，默认true（附录4）                |
| seata.enableAutoDataSourceProxy=true               | 是否开启数据源自动代理                      | true、false,seata-spring-boot-starter(SSBS)专有配置,SSBS默认会开启数据源自动代理,可通过该配置项关闭. |
| seata.useJdkProxy=false                            | 是否使用JDK代理作为数据源自动代理的实现方式 | true、false,(SSBS)专有配置,默认false,采用CGLIB作为数据源自动代理的实现方式 |
| transport.enableClientBatchSendRequest             | 客户端事务消息请求是否批量合并发送          | 默认true，false单条发送                                      |
| client.log.exceptionRate                           | 日志异常输出概率                            | 默认100，目前用于undo回滚失败时异常堆栈输出，百分之一的概率输出，回滚失败基本是脏数据，无需输出堆栈占用硬盘空间 |
| service.vgroupMapping.my_test_tx_group             | 事务群组（附录1）                           | my_test_tx_group为分组，配置项值为TC集群名                   |
| service.default.grouplist                          | TC服务列表（附录2）                         | 仅注册中心为file时使用                                       |
| service.disableGlobalTransaction                   | 全局事务开关                                | 默认false。false为开启，true为关闭                           |
| service.enableDegrade                              | 降级开关（待实现）                          | 默认false。业务侧根据连续错误数自动降级不走seata事务         |
| client.rm.reportSuccessEnable                      | 是否上报一阶段成功                          | true、false，从1.1.0版本开始,默认false.true用于保持分支事务生命周期记录完整，false可提高不少性能 |
| client.rm.asynCommitBufferLimit                    | 异步提交缓存队列长度                        | 默认10000。 二阶段提交成功，RM异步清理undo队列               |
| client.rm.lock.retryInterval                       | 校验或占用全局锁重试间隔                    | 默认10，单位毫秒                                             |
| client.rm.lock.retryTimes                          | 校验或占用全局锁重试次数                    | 默认30                                                       |
| client.rm.lock.retryPolicyBranchRollbackOnConflict | 分支事务与其它全局回滚事务冲突时锁策略      | 默认true，优先释放本地锁让回滚成功                           |
| client.rm.reportRetryCount                         | 一阶段结果上报TC重试次数                    | 默认5次                                                      |
| client.rm.tableMetaCheckEnable                     | 自动刷新缓存中的表结构                      | 默认false                                                    |
| client.tm.commitRetryCount                         | 一阶段全局提交结果上报TC重试次数            | 默认1次，建议大于1                                           |
| client.tm.rollbackRetryCount                       | 一阶段全局回滚结果上报TC重试次数            | 默认1次，建议大于1                                           |
| client.undo.dataValidation                         | 二阶段回滚镜像校验                          | 默认true开启，false关闭                                      |
| client.undo.logSerialization                       | undo序列化方式                              | 默认jackson                                                  |
| client.undo.logTable                               | 自定义undo表名                              | 默认undo_log                                                 |



# 事务分组

事务分组是seata的资源逻辑，类似于服务实例。在file.conf中的my_test_tx_group就是一个事务分组。

首先程序中配置了事务分组（GlobalTransactionScanner 构造方法的txServiceGroup参数），程序会通过用户配置的配置中心去寻找service.vgroupMapping .事务分组配置项，取得配置项的值就是TC集群的名称。拿到集群名称程序通过一定的前后缀+集群名称去构造服务名，各配置中心的服务名实现不同。拿到服务名去相应的注册中心去拉取相应服务名的服务列表，获得后端真实的TC服务列表。

这里多了一层获取事务分组到映射集群的配置。这样设计后，事务分组可以作为资源的逻辑隔离单位，出现某集群故障时可以快速failover，只切换对应分组，可以把故障缩减到服务级别，但前提也是你有足够server集群。

seata注册、配置中心分为两类，内置file、第三方注册（配置）中心如nacos等等，注册中心和配置中心之间没有约束，可各自使用不同类型。

## file注册中心和配置中心

### Server端

```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "file"                ---------------> 使用file作为注册中心
}
config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"                ---------------> 使用file作为配置中心
  file {
    name = "file.conf"
  }
}
```

- file、db模式启动server，见文章上方节点：启动Server

### Client端

```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "file"                ---------------> 使用file作为注册中心
}
config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"                ---------------> 使用file作为配置中心
  file {
    name = "file.conf"         ---------------> 配置参数存储文件
  }
}
spring.cloud.alibaba.seata.tx-service-group=my_test_tx_group ---------------> 事务分组配置
file.conf: 
    service {
      vgroupMapping.my_test_tx_group = "default"
      default.grouplist = "127.0.0.1:8091"
    }
```

- 读取配置

> 通过FileConfiguration本地加载file.conf的配置参数

- 获取事务分组

> spring配置，springboot可配置在yml、properties中，服务启动时加载配置，对应的值"my_test_tx_group"即为一个事务分组名，若不配置，默认获取属性spring.application.name的值+"-fescar-service-group"

- 查找TC集群名

> 拿到事务分组名"my_test_tx_group"拼接成"service.vgroupMapping.my_test_tx_group"查找TC集群名clusterName为"default"

- 查询TC服务

> 拼接"service."+clusterName+".grouplist"找到真实TC服务地址127.0.0.1:8091

## nacos注册中心和配置中心

### Server端

```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"                ---------------> 使用nacos作为注册中心
  nacos {
    serverAddr = "localhost"    ---------------> nacos注册中心所在ip
    namespace = ""              ---------------> nacos命名空间id，""为nacos保留public空间控件，用户勿配置namespace = "public"
    cluster = "default"         ---------------> seata-server在nacos的集群名
  }
}
config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"                ---------------> 使用nacos作为配置中心
  nacos {
    serverAddr = "localhost"
    namespace = ""
    cluster = "default"
  }
}
```

- 脚本

> script-->[config-center下的3个文件nacos-config.py](http://xn--config-center3nacos-config-9s15b3yaj3py01r5bse.py)、[nacos-config.sh](http://nacos-config.sh)、config.txt
>  txt为参数明细（包含Server和Client），sh为linux脚本，windows可下载git来操作，py为python脚本。

- 导入配置

> 用命令执行脚本导入seata配置参数至nacos，在nacos控制台查看配置确认是否成功

- 注册TC

> 启动seata-server注册至nacos，查看nacos控制台服务列表确认是否成功

### Client端

```
spring.cloud.alibaba.seata.tx-service-group=my_test_tx_group ---------------> 事务分组配置
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"                ---------------> 从nacos获取TC服务
  nacos {
    serverAddr = "localhost"
    namespace = ""
  }
}
config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"                ---------------> 使用nacos作为配置中心
  nacos {
    serverAddr = "localhost"
    namespace = ""
  }
}
```

- 读取配置

> 通过NacosConfiguration远程读取seata配置参数

- 获取事务分组

> springboot可配置在yml、properties中，服务启动时加载配置，对应的值"my_test_tx_group"即为一个事务分组名，若不配置，默认获取属性spring.application.name的值+"-fescar-service-group"

- 查找TC集群名

> 拿到事务分组名"my_test_tx_group"拼接成"service.vgroupMapping.my_test_tx_group"从配置中心查找到TC集群名clusterName为"default"

- 查找TC服务

> 根据serverAddr和namespace以及clusterName在注册中心找到真实TC服务列表

注：serverAddr和namespace与Server端一致，clusterName与Server端cluster一致



# springboot简单demo

## 添加依赖

```xml
<!-- 实现对 dynamic-datasource 的自动化配置 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>


<!-- 实现对 Seata 的自动化配置 -->
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>${seata.version}</version>
</dependency>
```

### 多数据源自动配置

[Mybatis-Plus-多数据源](https://mp.baomidou.com/guide/dynamic-datasource.html)

#### 简介

dynamic-datasource-spring-boot-starter 是一个基于springboot的快速集成多数据源的启动器。

其支持 **Jdk 1.7+,    SpringBoot 1.4.x  1.5.x   2.0.x**。

**示例项目** 可参考项目下的samples目录。

#### 特性

1. 数据源分组，适用于多种场景 纯粹多库  读写分离  一主多从  混合模式。
2. 内置敏感参数加密和启动初始化表结构schema数据库database。
3. 提供对Druid，Mybatis-Plus，P6sy，Jndi的快速集成。
4. 简化Druid和HikariCp配置，提供全局参数配置。
5. 提供自定义数据源来源接口(默认使用yml或properties配置)。
6. 提供项目启动后增减数据源方案。
7. 提供Mybatis环境下的  **纯读写分离** 方案。
8. 使用spel动态参数解析数据源，如从session，header或参数中获取数据源。（多租户架构神器）
9. 提供多层数据源嵌套切换。（ServiceA >>>  ServiceB >>> ServiceC，每个Service都是不同的数据源）
10. 提供 **不使用注解**  而   **使用 正则 或 spel**    来切换数据源方案（实验性功能）。
11. **基于seata的分布式事务支持。**

#### 约定

1. 本框架只做 **切换数据源** 这件核心的事情，并**不限制你的具体操作**，切换了数据源可以做任何CRUD。
2. 配置文件所有以下划线 `_` 分割的数据源 **首部** 即为组的名称，相同组名称的数据源会放在一个组下。
3. 切换数据源可以是组名，也可以是具体数据源名称。组名则切换时采用负载均衡算法切换。
4. 默认的数据源名称为  **master** ，你可以通过 `spring.datasource.dynamic.primary` 修改。
5. 方法上的注解优先于类上注解。

#### 使用方法

1. 引入dynamic-datasource-spring-boot-starter。

```xml
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
  <version>${version}</version>
</dependency>
```

1. 配置数据源。

```yaml
spring:
  datasource:
    dynamic:
      primary: master #设置默认的数据源或者数据源组,默认值即为master
      strict: false #设置严格模式,默认false不启动. 启动后在未匹配到指定数据源时候回抛出异常,不启动会使用默认数据源.
      datasource:
        master:
          url: jdbc:mysql://xx.xx.xx.xx:3306/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave_1:
          url: jdbc:mysql://xx.xx.xx.xx:3307/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave_2:
          url: ENC(xxxxx) # 内置加密,使用请查看详细文档
          username: ENC(xxxxx)
          password: ENC(xxxxx)
          driver-class-name: com.mysql.jdbc.Driver
          schema: db/schema.sql # 配置则生效,自动初始化表结构
          data: db/data.sql # 配置则生效,自动初始化数据
          continue-on-error: true # 默认true,初始化失败是否继续
          separator: ";" # sql默认分号分隔符
          
       #......省略
       #以上会配置一个默认库master，一个组slave下有两个子库slave_1,slave_2
```

```yml
 多主多从                 纯粹多库（记得设置primary）           混合配置
spring:                          spring:                          spring:
  datasource:                      datasource:                      datasource:
    dynamic:                         dynamic:                         dynamic:
      datasource:                      datasource:                      datasource:
        master_1:                        mysql:                           master:
        master_2:                        oracle:                          slave_1:
        slave_1:                         sqlserver:                       slave_2:
        slave_2:                         postgresql:                      oracle_1:
        slave_3:                         h2:                              oracle_2:
```


1. 使用  **@DS**  切换数据源。

**@DS** 可以注解在方法上和类上，**同时存在方法注解优先于类上注解**。

强烈建议只注解在service实现上。

|     注解      |                   结果                   |
| :-----------: | :--------------------------------------: |
|    没有@DS    |                默认数据源                |
| @DS("dsName") | dsName可以为组名也可以为具体某个库的名称 |

```java
@Service
@DS("slave")
public class UserServiceImpl implements UserService {

  @Autowired
  private JdbcTemplate jdbcTemplate;

  public List<Map<String, Object>> selectAll() {
    return  jdbcTemplate.queryForList("select * from user");
  }
  
  @Override
  @DS("slave_1")
  public List<Map<String, Object>> selectByCondition() {
    return  jdbcTemplate.queryForList("select * from user where age >10");
  }
}
```

## 属性文件

```yml
server:
  port: 8081 # 端口
  servlet:
    context-path: /seata-test

spring:
  application:
    name: seata-test  # 应用名
  datasource:
    # dynamic-datasource-spring-boot-starter 动态数据源的配配项，对应 DynamicDataSourceProperties 类
    dynamic:
      primary: order-ds # 设置默认的数据源或者数据源组，默认值即为 master
      datasource:
        # 订单 order 数据源配置
        order-ds:
          url: jdbc:mysql://127.0.0.1:3306/seata_order?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&allowMultiQueries=true&useSSL=false&tinyInt1isBit=false&serverTimezone=GMT%2B8
          driver-class-name: com.mysql.jdbc.Driver
          username: root
          password: root
        # 账户 pay 数据源配置
        amount-ds:
          url: jdbc:mysql://127.0.0.1:3306/seata_amount?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&allowMultiQueries=true&useSSL=false&tinyInt1isBit=false&serverTimezone=GMT%2B8
          driver-class-name: com.mysql.jdbc.Driver
          username: root
          password: root
        # 库存 storage 数据源配置
        storage-ds:
          url: jdbc:mysql://127.0.0.1:3306/seata_storage?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&allowMultiQueries=true&useSSL=false&tinyInt1isBit=false&serverTimezone=GMT%2B8
          driver-class-name: com.mysql.jdbc.Driver
          username: root
          password: root
      seata: true # 是否启动对 Seata 的集成

# Seata 配置项，对应 SeataProperties 类
seata:
  application-id: ${spring.application.name} # Seata 应用编号，默认为 ${spring.application.name}
  tx-service-group: ${spring.application.name}-group # Seata 事务组编号，用于 TC 集群名
  # 服务配置项，对应 ServiceProperties 类
  service:
    vgroup-mapping:
      seata-test-group: default # TC 集群（必须与seata-server保持一致）
    enable-degrade: false # 降级开关
    disable-global-transaction: false # 禁用全局事务（默认false）
    grouplist:
      default: 127.0.0.1:8091
```

## 添加回滚日志

为每个数据库添加回滚日志表。

```sql
CREATE TABLE `undo_log` (
	`id` BIGINT(20) NOT NULL AUTO_INCREMENT,
	`branch_id` BIGINT(20) NOT NULL,
	`xid` VARCHAR(100) NOT NULL,
	`context` VARCHAR(128) NOT NULL,
	`rollback_info` LONGBLOB NOT NULL,
	`log_status` INT(11) NOT NULL,
	`log_created` DATETIME NOT NULL,
	`log_modified` DATETIME NOT NULL,
	PRIMARY KEY (`id`),
	UNIQUE INDEX `ux_undo_log` (`xid`, `branch_id`)
)
COLLATE='utf8_general_ci'
ENGINE=InnoDB
AUTO_INCREMENT=11
```
## 简单使用

### dao

```java
@Mapper
@Repository
public interface AccountDao {

    /**
     * 获取账户余额
     *
     * @param userId 用户 ID
     * @return 账户余额
     */
    @Select("SELECT balance FROM account WHERE id = #{userId}")
    Integer getBalance(@Param("userId") Long userId);

    /**
     * 扣减余额
     *
     * @param price 需要扣减的数目
     * @return 影响记录行数
     */
    @Update("UPDATE account SET balance = balance - #{price} WHERE id = 1 AND balance >= ${price}")
    int reduceBalance(@Param("price") Integer price);

}

@Mapper
@Repository
public interface OrderDao {

    /**
     * 插入订单记录
     *
     * @param order 订单
     * @return 影响记录数量
     */
    @Insert("INSERT INTO orders (user_id, product_id, pay_amount) VALUES (#{userId}, #{productId}, #{payAmount})")
    @Options(useGeneratedKeys = true, keyColumn = "id", keyProperty = "id")
    int saveOrder(OrderDO order);

}

@Mapper
@Repository
public interface ProductDao {

    /**
     * 获取库存
     *
     * @param productId 商品编号
     * @return 库存
     */
    @Select("SELECT stock FROM product WHERE id = #{productId}")
    Integer getStock(@Param("productId") Long productId);

    /**
     * 扣减库存
     *
     * @param productId 商品编号
     * @param amount    扣减数量
     * @return 影响记录行数
     */
    @Update("UPDATE product SET stock = stock - #{amount} WHERE id = #{productId} AND stock >= #{amount}")
    int reduceStock(@Param("productId") Long productId, @Param("amount") Integer amount);

}
```

### service

#### 全局事务

```java
@Service
public class ProductServiceImpl implements ProductService {

    private Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private ProductDao productDao;

    @Override
    @DS(value = "storage-ds")
    @GlobalTransactional(rollbackFor = Exception.class)
    @Transactional(rollbackFor = Exception.class)
    public void reduceStock(Long productId, Integer amount) throws Exception {
        logger.info("[reduceStock] 当前 XID: {}", RootContext.getXID());

        // 检查库存
        checkStock(productId, amount);

        logger.info("[reduceStock] 开始扣减 {} 库存", productId);
        // 扣减库存
        int updateCount = productDao.reduceStock(productId, amount);
        // 扣除成功
        if (updateCount == 0) {
            logger.warn("[reduceStock] 扣除 {} 库存失败", productId);
            throw new Exception("库存不足");
        }
        // 扣除失败
        logger.info("[reduceStock] 扣除 {} 库存成功", productId);
    }

    private void checkStock(Long productId, Integer requiredAmount) throws Exception {
        logger.info("[checkStock] 检查 {} 库存", productId);
        Integer stock = productDao.getStock(productId);
        if (stock < requiredAmount) {
            logger.warn("[checkStock] {} 库存不足，当前库存: {}", productId, stock);
            throw new Exception("库存不足");
        }
    }

}
```

#### 本地事务-余额扣除

```java
@Service
public class AccountServiceImpl implements AccountService {

    private Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private AccountDao accountDao;

    @Override
    @DS(value = "amount-ds")
    @Transactional(rollbackFor = Exception.class)
    public void reduceBalance(Long userId, Integer price) throws Exception {
        logger.info("[reduceBalance] 当前 XID: {}", RootContext.getXID());

        // 检查余额
        checkBalance(userId, price);

        logger.info("[reduceBalance] 开始扣减用户 {} 余额", userId);
        // 扣除余额
        int updateCount = accountDao.reduceBalance(price);
        // 扣除成功
        if (updateCount == 0) {
            logger.warn("[reduceBalance] 扣除用户 {} 余额失败", userId);
            throw new Exception("余额不足");
        }
        logger.info("[reduceBalance] 扣除用户 {} 余额成功", userId);
    }

    private void checkBalance(Long userId, Integer price) throws Exception {
        logger.info("[checkBalance] 检查用户 {} 余额", userId);
        Integer balance = accountDao.getBalance(userId);
        if (balance < price) {
            logger.warn("[checkBalance] 用户 {} 余额不足，当前余额:{}", userId, balance);
            throw new Exception("余额不足");
        }
    }

}
```

#### 本地事务-生成订单

```java
@Service
public class OrderServiceImpl implements OrderService {

    private Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private OrderDao orderDao;

    @Autowired
    private AccountService accountService;

    @Autowired
    private ProductService productService;

    @Override
    @DS(value = "order-ds")
    @Transactional(rollbackFor = Exception.class)
    public Integer createOrder(Long userId, Long productId, Integer price) throws Exception {
        Integer amount = 1; // 购买数量，暂时设置为 1。

        logger.info("[createOrder] 当前 XID: {}", RootContext.getXID());

        // 扣减库存
        productService.reduceStock(productId, amount);

        // 扣减余额
        accountService.reduceBalance(userId, price);

        // 保存订单
        OrderDO order = new OrderDO().setUserId(userId).setProductId(productId).setPayAmount(amount * price);
        orderDao.saveOrder(order);
        logger.info("[createOrder] 保存订单: {}", order.getId());

        // 返回订单编号
        return order.getId();
    }

}
```

### controller

```java
@RestController
@RequestMapping("/order")
public class OrderController {

    private Logger logger = LoggerFactory.getLogger(OrderController.class);

    @Autowired
    private OrderService orderService;

    @PostMapping("/create")
    public Integer createOrder(@RequestParam("userId") Long userId,
                               @RequestParam("productId") Long productId,
                               @RequestParam("price") Integer price) throws Exception {
        logger.info("[createOrder] 收到下单请求,用户:{}, 商品:{}, 价格:{}", userId, productId, price);
        return orderService.createOrder(userId, productId, price);
    }

}
```