# 简单使用

1.依赖配置

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.2</version>
</dependency>
```

2.映射自动扫描

```java
@MapperScan("com.lcm.test.mybatis.testmybatis.mapper")
@SpringBootApplication
public class TestMybatisApplication {

    public static void main(String[] args) {
        SpringApplication.run(TestMybatisApplication.class, args);
    }

}
```

3.配置文件

```yml
spring:
  application:
    name: test-mybatis
  # 数据库配置
  datasource:
    type: com.zaxxer.hikari.HikariDataSource
    url: jdbc:mysql://127.0.0.1:3306/security?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&allowMultiQueries=true&useSSL=false&tinyInt1isBit=false&serverTimezone=GMT%2B8&zeroDateTimeBehavior=convertToNull
    driverClassName: com.mysql.cj.jdbc.Driver
    username: root
    password: root
    # hikari数据库连接池配置
    hikari:
      idleTimeout: 30000
      maximumPoolSize: 10

# mybatis配置
mybatis:
  # 给实体类配置别名
  type-aliases-package: com.lcm.test.mybatis.testmybatis.entity
  # 加载mybatis核心配置文件
  # configuration 和 configLocation 不能同时存在,这里使用configuration
  #  config-location: classpath:mybatis/mybatis-config.xml
  mapper-locations: classpath:mapper/**/*Mapper.xml
  configuration:
    # 开发环境控制台打印sql语句
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
    # 开启驼峰规则自动映射字段属性值；如字段为user_name的可以映射到userName属性中
    map-underscore-to-camel-case: true
    # 设置sql执行超时时间,以秒为单位的全局sql超时时间设置,当超出了设置的超时时间时,会抛出SQLTimeoutException
    default-statement-timeout: 30
    # 解决查询返回结果含null没有对应字段值问题
    call-setters-on-nulls: true
```

4.实体类

```java
package com.lcm.test.mybatis.testmybatis.entity;

import lombok.Data;

import java.util.Date;

@Data
public class SysUser {

    private Long userId;

    private Long deptId;

    private String userName;

    ......

}
```

5.映射类

```java
package com.lcm.test.mybatis.testmybatis.mapper;

import com.lcm.test.mybatis.testmybatis.entity.SysUser;

public interface SysUserMapper {

    SysUser select(Long id);

}
```

6.映射配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.lcm.test.mybatis.testmybatis.mapper.SysUserMapper">


    <select id="select" resultType="com.lcm.test.mybatis.testmybatis.entity.SysUser" >
        select * from sys_user where user_id=#{id}
    </select>

</mapper>
```

7.测试文件

```java
package com.lcm.test.mybatis.testmybatis;

import com.lcm.test.mybatis.testmybatis.mapper.SysUserMapper;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@Slf4j
@SpringBootTest
class TestMybatisApplicationTests {

    @Autowired
    SysUserMapper sysUserMapper;

    @Test
    void contextLoads() {
        log.info(sysUserMapper.select(2L).toString());
    }

}
```



# 配置文件