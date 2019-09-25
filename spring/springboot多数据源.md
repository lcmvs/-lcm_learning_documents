# springboot多数据源

开发一个项目的时候，可能会遇到需要对多个数据库进行操作的需求，这时候就得在项目中配置多个数据源了。

### 1.配置yaml属性文件，多数据源属性

##### 1.1 配置数据库的相应属性，用于config类获取相关属性。

```yml
#primary 主数据库
db1:
  mapper: classpath:mapper/primary/*.xml
  url: jdbc:mysql://10.172.0.163:3306/iovs?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&allowMultiQueries=true&useSSL=false&serverTimezone=GMT%2B8&tinyInt1isBit=true
  driverClassName: com.mysql.cj.jdbc.Driver
  username: mcmysql
  password: 123456
  hikari:
    idleTimeout: 30000
    maximumPoolSize: 10

db2:
  mapper: classpath:mapper/db2/*.xml
  url: jdbc:mysql://rm-wz9usshd072esja7m5o.mysql.rds.aliyuncs.com:3306/evms?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&allowMultiQueries=true&useSSL=false&serverTimezone=GMT%2B8&tinyInt1isBit=true
  driverClassName: com.mysql.cj.jdbc.Driver
  username: evms
  password: evms!1918
  hikari:
    idleTimeout: 30000
    maximumPoolSize: 10
```

##### 1.2 创建属性实体类获取db属性

```pom
<dependency>
    <groupId> org.springframework.boot </groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

```java
@Data
@Component
@ConfigurationProperties(prefix="db1")
public class PrimaryProperties {

    private String mapper;
    private String url;
    private String driverClassName;
    private String username;
    private String password;
    private Map<String,String> hikari;

}
```

### 2.编写数据库config配置类

```java
/*
com.wcwc.iovs.multipledatasource.module.dao.primary为该数据源扫描package
*/
@Configuration
@MapperScan(basePackages = "com.wcwc.iovs.multipledatasource.module.dao.primary", sqlSessionFactoryRef = "primarySqlSessionFactory")
public class PrimaryDataBaseConfig {

    @Autowired
    PrimaryProperties primaryProperties;

    @Primary
    @Bean(name = "primaryDataSource")
    public DataSource primaryDataSource(){
        HikariDataSource hikariDataSource=new HikariDataSource();
        hikariDataSource.setJdbcUrl(primaryProperties.getUrl());
        hikariDataSource.setDriverClassName(primaryProperties.getDriverClassName());
        hikariDataSource.setUsername(primaryProperties.getUsername());
        hikariDataSource.setPassword(primaryProperties.getPassword());
        hikariDataSource.setIdleTimeout(Long.parseLong(primaryProperties.getHikari().get("idleTimeout")));
        hikariDataSource.setMaximumPoolSize(Integer.parseInt(primaryProperties.getHikari().get("maximumPoolSize")));
        return hikariDataSource;
    }

    /**
     * 创建该数据源的事务管理
     * @return
     * @throws SQLException
     */
    @Primary
    @Bean(name = "primaryTransactionManager")
    public DataSourceTransactionManager primaryTransactionManager(){
        return new DataSourceTransactionManager(primaryDataSource());
    }


    /**
     * 创建Mybatis的连接会话工厂实例
     * @param dataSource
     * @return
     * @throws Exception
     */
    @Primary
    @Bean(name = "primarySqlSessionFactory")
    public SqlSessionFactory primarySqlSessionFactory(@Qualifier("primaryDataSource") DataSource dataSource) throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        // 设置数据源bean
        sessionFactory.setDataSource(dataSource);
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()
                // 设置mapper文件路径
                .getResources(primaryProperties.getMapper()));

        return sessionFactory.getObject();
    }

}
```

### 3.相应package和mapper下编写dao接口和映射文件

##### 3.1 包com.wcwc.iovs.multipledatasource.module.dao.primary下：

```java
package com.wcwc.iovs.multipledatasource.module.dao.primary;

import org.springframework.stereotype.Component;

import java.util.Map;

/**
 * @description:
 * @author: lcm
 * @create: 2019-04-25 11:36
 **/
@Component
public interface PrimaryDao {

    Map select();

}

```
##### 3.2 映射文件classpath:mapper/primary/*.xml下：

```xml

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.wcwc.iovs.multipledatasource.module.dao.primary.PrimaryDao">

    <select id="select" resultType="java.util.Map">
    select * from area limit 1
    </select>


</mapper>
```