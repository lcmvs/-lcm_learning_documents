

为什么webflux没能够普及。

1. reactive 编程难度较高，编码时你需要时刻注意防止自己的程序阻塞，不然会导致非常严重的后果
2. 技术还支持不够完善比如 JDBC 目前各种数据库的驱动对于异步编程本身支持较弱，要支持 reactive 编程可能还需要一个很长的过程
3. 如果你使用 reactive 编程像 spring 事务管理以及 aop 那一套模式全都得重新换
4. 传统的编程方式已经可以满足很多公司，及业务场景的需求，可能对于 reactive 编程需求并不会那么高

[Spring 5 WebFlux和JDBC：阻止还是不阻止](<https://dzone.com/articles/spring-5-webflux-and-jdbc-to-block-or-not-to-block>)