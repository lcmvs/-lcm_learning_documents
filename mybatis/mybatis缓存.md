# mybatis缓存

[SpringBoot 下 mybatis 的缓存](https://www.cnblogs.com/zhuwbox/p/10460765.html)

[聊聊MyBatis缓存机制](https://tech.meituan.com/2018/01/19/mybatis-cache.html)

## spring boot中使用

```yaml
mybatis:
  configuration:
    #一级缓存 默认session，使用事务可能导致脏数据，建议statement
    local-cache-scope: statement
    #二级缓存 默认true，建议关闭，范围namespace，也就是一个mapper
    cache-enabled: false
```



```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MybatisSqlSessionTest {

    @Autowired
    SysUserDao sysUserDao;

    @Test
    @Transactional(rollbackFor = Exception.class,isolation = Isolation.READ_COMMITTED)
    public void test() {
        System.out.println(sysUserDao.getUserByKey("1"));
        System.out.println(sysUserDao.getUserByKey("1"));
        System.out.println(sysUserDao.getUserByKey("1"));
    }

    @Test
    public void test2() {
        System.out.println(sysUserDao.getUserByKey("1"));
        System.out.println(sysUserDao.getUserByKey("1"));
        System.out.println(sysUserDao.getUserByKey("1"));
    }

}
```
使用了spring的声明式事务，同一个事务内的sql执行使用的是同一个sqlSession，测试mybatis的一级缓存旧生效了，同一条查询执行了3次，但只是第一次查了数据库，后面全都使用了一级缓存。
```properties
Creating a new SqlSession
Registering transaction synchronization for SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@65600fb3]
JDBC Connection [HikariProxyConnection@1787450654 wrapping com.mysql.cj.jdbc.ConnectionImpl@3cc9632d] will be managed by Spring
==>  Preparing: select user_id,username,real_name,mobile_num from sys_user where username like concat('%',?,'%') or real_name like concat('%',?,'%') limit 10 
==> Parameters: 1(String), 1(String)
<==    Columns: user_id, username, real_name, mobile_num
<==        Row: 111, lighter613, 刘朗, 18202714635
<==      Total: 1
Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@65600fb3]
[{realName=刘朗, mobileNum=18202714635, userId=111, username=lighter613}]
Fetched SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@65600fb3] from current transaction
Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@65600fb3]
[{realName=刘朗, mobileNum=18202714635, userId=111, username=lighter613}]
Fetched SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@65600fb3] from current transaction
Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@65600fb3]
[{realName=刘朗, mobileNum=18202714635, userId=111, username=lighter613}]
Transaction synchronization deregistering SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@65600fb3]
Transaction synchronization closing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@65600fb3]
```
没有使用spring的声明式事务，每条sql的执行，使用的是不同的sqlSession，所以即便是相同的查询，也不会被一级缓存所影响。
```properties
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@5aea8994] was not registered for synchronization because synchronization is not active
JDBC Connection [HikariProxyConnection@547610892 wrapping com.mysql.cj.jdbc.ConnectionImpl@3cc9632d] will not be managed by Spring
==>  Preparing: select user_id,username,real_name,mobile_num from sys_user where username like concat('%',?,'%') or real_name like concat('%',?,'%') limit 10 
==> Parameters: 1(String), 1(String)
<==    Columns: user_id, username, real_name, mobile_num
<==        Row: 111, lighter613, 刘朗, 18202714635
<==      Total: 1
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@5aea8994]
[{realName=刘朗, mobileNum=18202714635, userId=111, username=lighter613}]
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@5002fde9] was not registered for synchronization because synchronization is not active
JDBC Connection [HikariProxyConnection@1872158052 wrapping com.mysql.cj.jdbc.ConnectionImpl@3cc9632d] will not be managed by Spring
==>  Preparing: select user_id,username,real_name,mobile_num from sys_user where username like concat('%',?,'%') or real_name like concat('%',?,'%') limit 10 
==> Parameters: 1(String), 1(String)
<==    Columns: user_id, username, real_name, mobile_num
<==        Row: 111, lighter613, 刘朗, 18202714635
<==      Total: 1
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@5002fde9]
[{realName=刘朗, mobileNum=18202714635, userId=111, username=lighter613}]
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@409732fb] was not registered for synchronization because synchronization is not active
JDBC Connection [HikariProxyConnection@121463477 wrapping com.mysql.cj.jdbc.ConnectionImpl@3cc9632d] will not be managed by Spring
==>  Preparing: select user_id,username,real_name,mobile_num from sys_user where username like concat('%',?,'%') or real_name like concat('%',?,'%') limit 10 
==> Parameters: 1(String), 1(String)
<==    Columns: user_id, username, real_name, mobile_num
<==        Row: 111, lighter613, 刘朗, 18202714635
<==      Total: 1
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@409732fb]
[{realName=刘朗, mobileNum=18202714635, userId=111, username=lighter613}]
```



## 总结

1、MyBatis一级缓存的生命周期和**SqlSession**一致。

2、MyBatis一级缓存内部设计简单，只是一个没有容量限定的HashMap，在缓存的功能性上有所欠缺。

3、MyBatis的一级缓存最大范围是SqlSession内部，有多个SqlSession或者分布式的环境下，数据库写操作会引起**脏数据**，虽然没有关闭的配置，但是设定缓存级别为**Statement**，也等同于没有使用缓存了。

4、MyBatis的二级缓存相对于一级缓存来说，实现了SqlSession之间缓存数据的共享，同时粒度更加的细，能够到**namespace**级别，也就是一个mapper.xml，通过Cache接口实现类不同的组合，对Cache的可控性也更强。

5、MyBatis在多表查询时，极大可能会出现脏数据，有设计上的缺陷，安全使用二级缓存的条件比较苛刻。

6、在分布式环境下，由于默认的MyBatis Cache实现都是基于本地的，分布式环境下必然会出现读取到脏数据，需要使用集中式缓存将MyBatis的Cache接口实现，有一定的开发成本，直接使用Redis、Memcached等分布式缓存可能成本更低，安全性也更高。

7、个人建议MyBatis缓存特性在生产环境中进行关闭，单纯作为一个ORM框架使用可能更为合适。