# jvm调试工具



## jps

[jvm 性能调优工具之 jps](https://www.jianshu.com/p/d39b2e208e72)

jps 命令类似与 linux 的 ps 命令，但是它只列出系统中所有的 Java 应用程序。 通过 jps 命令可以方便地查看 Java 进程的启动类、传入参数和 Java 虚拟机参数等信息。

如果在 linux 中想查看 java 的进程，一般我们都需要 ps -ef | grep java 来获取进程 ID。
 如果只想获取 Java 程序的进程，可以直接使用 jps 命令来直接查看。

`jps -l`

参数 -l 可以输出主函数的完整路径（类的全路径）



## jmap

`jmap -histo:live 765468|findstr okhttp3.internal.connection`

查看pid为765468的进程，包含关键字okhttp3.internal.connection的对象的内存信息

命令jmap是一个多功能的命令。它可以生成 java 程序的 dump 文件， 也可以查看堆内对象示例的统计信息、查看 ClassLoader 的信息以及 finalizer 队列。

[jvm 性能调优工具之 jmap](https://www.jianshu.com/p/a4ad53179df3)

[如何计算Java对象所占内存的大小](https://www.jianshu.com/p/9d729c9c94c4)

