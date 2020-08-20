[高性能网络编程(五)：一文读懂高性能网络编程中的I/O模型](http://www.52im.net/thread-1935-1-1.html)

# 高性能io模型

**首先看看一个典型互联网服务端处理网络请求的典型过程：**

![高性能网络编程(五)：一文读懂高性能网络编程中的I/O模型_1.jpeg](http://www.52im.net/data/attachment/forum/201809/05/211858pgsyanbk1yffennv.jpeg)

由上图可以看到，主要处理步骤包括：

- 1）获取请求数据，客户端与服务器建立连接发出请求，服务器接受请求（1-3）；
- 2）构建响应，当服务器接收完请求，并在用户空间处理客户端的请求，直到构建响应完成（4）；
- 3）返回数据，服务器将已构建好的响应再通过内核空间的网络 I/O 发还给客户端（5-7）。

设计服务端并发模型时，主要有如下两个关键点：

- 1）服务器如何管理连接，获取输入数据；
- 2）服务器如何处理请求。

以上两个关键点最终都与操作系统的 I/O 模型以及线程(进程)模型相关。



## I/O 模型的基本认识

介绍操作系统的 I/O 模型之前，先了解一下几个概念：

### 阻塞/非阻塞

- 1）阻塞调用与非阻塞调用；
- 2）阻塞调用是指调用结果返回之前，当前线程会被挂起，调用线程只有在得到结果之后才会返回；
- 3）非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程。

两者的最大区别在于被调用方在收到请求到返回结果之前的这段时间内，调用方是否一直在等待。

**阻塞**是指调用方一直在等待而且别的事情什么都不做；**非阻塞**是指调用方先去忙别的事情。

### 同步/异步

同步处理是指被调用方得到最终结果之后才返回给调用方；

异步处理是指被调用方先返回应答，然后再计算调用结果，计算完最终结果后再通知并返回给调用方。

阻塞、非阻塞和同步、异步的区别（阻塞、非阻塞和同步、异步其实针对的对象是不一样的）：

- 1）阻塞、非阻塞的讨论对象是调用者；
- 2）同步、异步的讨论对象是被调用者。

### recvfrom 函数

recvfrom 函数(经 Socket 接收数据)，这里把它视为系统调用。

一个输入操作通常包括两个不同的阶段：

- 1）等待数据准备好；
- 2）从内核向进程复制数据。

对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。当所等待分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到应用进程缓冲区。

实际应用程序在系统调用完成上面的 2 步操作时，调用方式的阻塞、非阻塞，操作系统在处理应用程序请求时，处理方式的同步、异步处理的不同，可以分为 5 种 I/O 模型。（参考《UNIX网络编程卷1》）



## 1) IO模型

#### 1.1 阻塞式 I/O 模型(blocking I/O）

  ![高性能网络编程(五)：一文读懂高性能网络编程中的I/O模型_2.jpeg](http://www.52im.net/data/attachment/forum/201809/05/212717yp8iwt5z8j1niw8a.jpeg)    

在这里，把recvfrom函数视为系统调用。进程调用recvfrom，其系统调用直到数据报到达且被复制到应用进程的缓冲区中或者发生错误才返回。进程在从调用recvfrom开始到它返回的整段时间内是被阻塞的。recvfrom成功返回后，应用进程开始处理数据报。



**优点：**

程序简单，在阻塞等待数据期间进程/线程挂起，基本不会占用 CPU 资源。

缺点：

每个连接需要独立的进程/线程单独处理，当并发请求量大时为了维护程序，内存、线程切换开销较大，这种模型在实际生产中很少使用。

#### 1.2 非阻塞式 I/O 模型(non-blocking I/O）

  ![高性能网络编程(五)：一文读懂高性能网络编程中的I/O模型_3.jpeg](http://www.52im.net/data/attachment/forum/201809/05/212910wn44nrr6zp5siiuo.jpeg)    

前三次调用recvfrom时没有数据可返回，因此内核转而立即返回一个EWOULDBLOCK错误。
 第四次调用recvfrom时已有一个数据报准备好，它被复制到应用进程缓冲区，于是recvfrom成功返回。我们接着处理数据。

当一个应用进程像这样对一个非阻塞描述符`循环调用`recvfrom时，我们称之为`轮询（polling）`。应用进程持续轮询内核，以查看某个操作是否就绪。这么做往往耗费大量CPU时间。



**优点：**

不会阻塞在内核的等待数据过程，每次发起的 I/O 请求可以立即返回，不用阻塞等待，实时性较好。

**缺点：**

轮询将会不断地询问内核，这将占用大量的 CPU 时间，系统资源利用率较低，所以一般 Web 服务器不使用这种 I/O 模型。

#### 1.3 I/O 复用模型(I/O multiplexing）

  ![高性能网络编程(五)：一文读懂高性能网络编程中的I/O模型_4.jpeg](http://www.52im.net/data/attachment/forum/201809/05/213041mtejdsoeojfjy7dd.jpeg)    

我们阻塞于select调用，等待数据报套接字`变为可读`。当select返回套接字可读这一条件时，我们调用recvfrom把所读数据报`复制`到应用进程缓冲区。

比较前两个io模型，I/O复用并不显得有什么优势，事实上由于使用select需要两个而不是单个系统调用，I/O复用还稍有劣势。不过，使用select的`优势在于`我们可以**等待多个描述符就绪**。



优点：

可以基于一个阻塞对象，同时在多个描述符上等待就绪，而不是使用多个线程(每个文件描述符一个线程)，这样可以大大节省系统资源。

缺点：

当连接数较少时效率相比多线程+阻塞 I/O 模型效率较低，可能延迟更大，因为单个连接处理需要 2 次系统调用，占用时间会有增加。

众所周之，Nginx这样的高性能互联网反向代理服务器大获成功的关键就是得益于Epoll。

#### 1.4 信号驱动式 I/O 模型（signal-driven I/O)

  ![高性能网络编程(五)：一文读懂高性能网络编程中的I/O模型_5.jpeg](http://www.52im.net/data/attachment/forum/201809/05/213143a7n3mnxb38ybgxy3.jpeg)    

1. 首先开启套接字的信号驱动式I/O功能，并通过 sigaction 系统调用安装一个信号处理函数。该系统调用将立即返回，我们的进程继续工作，也就是说它没有被阻塞。
2. 当数据报准备好读取时，内核就为该进程产生一个SIGIO信号。
3. 我们随后既可以在信号处理函数中调用recvfrom读取数据报，并通知主循环数据已准备好待处理，也可以立即通知主循环，让它读取数据报。

无论如何处理SIGIO信号，这种模型的优势在于等待数据报到达期间进程不被阻塞。主循环可以继续执行，只要等待来自信号处理函数的通知：既可以是数据已准备好被处理，也可以是数据报已准备好被读取。



优点：

线程并没有在等待数据时被阻塞，可以提高资源的利用率。

缺点：

信号 I/O 在大量 IO 操作时可能会因为信号队列溢出导致没法通知。

 信号驱动 I/O 尽管对于处理 UDP 套接字来说有用，即这种信号通知意味着到达一个数据报，或者返回一个异步错误。

 但是，对于 TCP 而言，信号驱动的 I/O 方式近乎无用，因为导致这种通知的条件为数众多，每一个来进行判别会消耗很大资源，与前几种方式相比优势尽失。

#### 1.5 异步 I/O 模型（即AIO，全称asynchronous I/O）

  ![高性能网络编程(五)：一文读懂高性能网络编程中的I/O模型_6.jpeg](http://www.52im.net/data/attachment/forum/201809/05/213218wbeovsvt6g7s4zsj.jpeg)    

由POSIX规范定义。

一般来说，这些函数的工作机制是：告知内核启动某个操作，并让内核在整个操作（包括将数据从内核复制到我们自己的缓冲区）完成后通知我们。

这种模型与信号驱动模型的主要区别在于：

- 信号驱动式I/O：内核通知我们`何时可以启动`一个I/O操作，
- 异步I/O模型：内核通知我们I/O操作`何时完成`。

我们调用aio_read函数（POSIX异步I/O函数以aio_或lio_开头），给内核传递`描述符`、`缓冲区指针`、`缓冲区大小`（与read相同的三个参数）和文件偏移（与lseek类似），并告诉内核当整个操作完成时如何通知我们。该系统调用`立即返回`，而且在等待I/O完成期间，我们的进程`不被阻塞`。

本例子中我们假设要求内核在操作完成时产生某个信号。该信号直到数据已复制到应用进程缓冲区才产生，这一点不同于信号驱动式I/O模型。

至此时，支持POSIX异步I/O模型的系统`仍较罕见`。我们不能确定这样的系统是否支持套接字上的这种模型。这儿我们只是用它作为一个与信号驱动式I/O模型相比照的例子。



优点：

异步 I/O 能够充分利用 DMA 特性，让 I/O 操作与计算重叠。

缺点：

要实现真正的异步 I/O，操作系统需要做大量的工作。目前 Windows 下通过 IOCP 实现了真正的异步 I/O。

 而在 Linux 系统下，Linux 2.6才引入，目前 AIO 并不完善，因此在 Linux 下实现高并发网络编程时都是以 IO 复用模型模式为主。

 关于AOI的介绍，请见：《Java新一代网络编程模型AIO原理及Linux系统AIO介绍》。

#### 1.6 5 种 I/O 模型总结

  ![高性能网络编程(五)：一文读懂高性能网络编程中的I/O模型_7.jpeg](http://www.52im.net/data/attachment/forum/201809/05/213459mmmhohhgwom24uoj.jpeg)    

 从上图中我们可以看出，越往后，阻塞越少，理论上效率也是最优。

 这五种 I/O 模型中，前四种属于同步 I/O，因为其中真正的 I/O 操作(recvfrom)将阻塞进程/线程，只有异步 I/O 模型才与 POSIX 定义的异步 I/O 相匹配。

## 2) 线程模型

[高性能网络编程(六)：一文读懂高性能网络编程中的线程模型](http://www.52im.net/thread-1939-1-1.html)

#### 2.1 线程模型1：传统阻塞 I/O 服务模型

  ![高性能网络编程(六)：一文读懂高性能网络编程中的线程模型_1.jpeg](http://www.52im.net/data/attachment/forum/201809/06/195333v2cj2o6y92d2zp5z.jpeg)    

特点：

- 1）采用阻塞式 I/O 模型获取输入数据；
- 2）每个连接都需要独立的线程完成数据输入，业务处理，数据返回的完整操作。
  

存在问题：

- 1）当并发数较大时，需要创建大量线程来处理连接，系统资源占用较大；
- 2）连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在 Read 操作上，造成线程资源浪费。
  

#### 2.2 线程模型2：Reactor 模式

##### 2.2.1基本介绍

针对传统阻塞 I/O 服务模型的 2 个缺点，比较常见的有如下解决方案： 

- 1）基于 I/O 复用模型：多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象上等待，无需阻塞等待所有连接。当某条连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理；
- 2）基于线程池复用线程资源：不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，一个线程可以处理多个连接的业务。
  

I/O 复用结合线程池，这就是 Reactor 模式基本设计思想，如下图：

  ![高性能网络编程(六)：一文读懂高性能网络编程中的线程模型_2.jpeg](http://www.52im.net/data/attachment/forum/201809/06/195839s5hi3te5pxueq5ze.jpeg)    

 Reactor 模式，是指通过一个或多个输入同时传递给服务处理器的服务请求的事件驱动处理模式。 

 服务端程序处理传入多路请求，并将它们同步分派给请求对应的处理线程，Reactor 模式也叫 Dispatcher 模式。

 即 I/O 多了复用统一监听事件，收到事件后分发(Dispatch 给某进程)，是编写高性能网络服务器的必备技术之一。

Reactor 模式中有 2 个关键组成：

- 1）Reactor：Reactor 在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对 IO 事件做出反应。 它就像公司的电话接线员，它接听来自客户的电话并将线路转移到适当的联系人；
- 2）Handlers：处理程序执行 I/O 事件要完成的实际事件，类似于客户想要与之交谈的公司中的实际官员。Reactor 通过调度适当的处理程序来响应 I/O 事件，处理程序执行非阻塞操作。
  

根据 Reactor 的数量和处理资源池线程的数量不同，有 3 种典型的实现：

- 1）单 Reactor 单线程；
- 2）单 Reactor 多线程；
- 3）主从 Reactor 多线程。
  

 下面详细介绍这 3 种实现方式。

##### 2.2.2单 Reactor 单线程

  ![高性能网络编程(六)：一文读懂高性能网络编程中的线程模型_3.jpeg](http://www.52im.net/data/attachment/forum/201809/06/200048bgll2l41w72174ot.jpeg)    

 其中，Select 是前面 I/O 复用模型介绍的标准网络编程 API，可以实现应用程序通过一个阻塞对象监听多路连接请求，其他方案示意图类似。

方案说明：

- 1）Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发；
- 2）如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理；
- 3）如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应；
- 4）Handler 会完成 Read→业务处理→Send 的完整业务流程。
  

**优点：**

模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成。

缺点：

性能问题，只有一个线程，无法完全发挥多核 CPU 的性能。Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈。

 可靠性问题，线程意外跑飞，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。

**使用场景：**

客户端的数量有限，业务处理非常快速，比如 Redis，业务处理的时间复杂度 O(1)。

##### 2.2.3单 Reactor 多线程

  ![高性能网络编程(六)：一文读懂高性能网络编程中的线程模型_4.jpeg](http://www.52im.net/data/attachment/forum/201809/06/200650wun9j9ghkgk7ngna.jpeg)    

方案说明：

- 1）Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发；
- 2）如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后续的各种事件；
- 3）如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应；
- 4）Handler 只负责响应事件，不做具体业务处理，通过 Read 读取数据后，会分发给后面的 Worker 线程池进行业务处理；
- 5）Worker 线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给 Handler 进行处理；
- 6）Handler 收到响应结果后通过 Send 将响应结果返回给 Client。
  

优点：

可以充分利用多核 CPU 的处理能力。

缺点：

多线程数据共享和访问比较复杂；Reactor 承担所有事件的监听和响应，在单线程中运行，高并发场景下容易成为性能瓶颈。

##### 2.2.4主从 Reactor 多线程

  ![高性能网络编程(六)：一文读懂高性能网络编程中的线程模型_5.jpeg](http://www.52im.net/data/attachment/forum/201809/06/200759gg777fr7v7wzcr7r.jpeg)    

 针对单 Reactor 多线程模型中，Reactor 在单线程中运行，高并发场景下容易成为性能瓶颈，可以让 Reactor 在多线程中运行。

方案说明：

- 1）Reactor 主线程 MainReactor 对象通过 Select 监控建立连接事件，收到事件后通过 Acceptor 接收，处理建立连接事件；
- 2）Acceptor 处理建立连接事件后，MainReactor 将连接分配 Reactor 子线程给 SubReactor 进行处理；
- 3）SubReactor 将连接加入连接队列进行监听，并创建一个 Handler 用于处理各种连接事件；
- 4）当有新的事件发生时，SubReactor 会调用连接对应的 Handler 进行响应；
- 5）Handler 通过 Read 读取数据后，会分发给后面的 Worker 线程池进行业务处理；
- 6）Worker 线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给 Handler 进行处理；
- 7）Handler 收到响应结果后通过 Send 将响应结果返回给 Client。
  

优点：

父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。

 父线程与子线程的数据交互简单，Reactor 主线程只需要把新连接传给子线程，子线程无需返回数据。

 这种模型在许多项目中广泛使用，包括 Nginx 主从 Reactor 多进程模型，Memcached 主从多线程，Netty 主从多线程模型的支持。

## 3）option和childOption参数设置说明

#### 3.1.通用参数

```
CONNECT_TIMEOUT_MILLIS :
  Netty参数，连接超时毫秒数，默认值30000毫秒即30秒。
MAX_MESSAGES_PER_READ
  Netty参数，一次Loop读取的最大消息数，对于ServerChannel或者NioByteChannel，默认值为16，其他Channel默认值为1。默认值这样设置，是因为：ServerChannel需要接受足够多的连接，保证大吞吐量，NioByteChannel可以减少不必要的系统调用select。
WRITE_SPIN_COUNT
  Netty参数，一个Loop写操作执行的最大次数，默认值为16。也就是说，对于大数据量的写操作至多进行16次，如果16次仍没有全部写完数据，此时会提交一个新的写任务给EventLoop，任务将在下次调度继续执行。这样，其他的写请求才能被响应不会因为单个大数据量写请求而耽误。
ALLOCATOR
  Netty参数，ByteBuf的分配器，默认值为ByteBufAllocator.DEFAULT，4.0版本为UnpooledByteBufAllocator，4.1版本为PooledByteBufAllocator。该值也可以使用系统参数io.netty.allocator.type配置，使用字符串值："unpooled"，"pooled"。
RCVBUF_ALLOCATOR
  Netty参数，用于Channel分配接受Buffer的分配器，默认值为AdaptiveRecvByteBufAllocator.DEFAULT，是一个自适应的接受缓冲区分配器，能根据接受到的数据自动调节大小。可选值为FixedRecvByteBufAllocator，固定大小的接受缓冲区分配器。
AUTO_READ
  Netty参数，自动读取，默认值为True。Netty只在必要的时候才设置关心相应的I/O事件。对于读操作，需要调用channel.read()设置关心的I/O事件为OP_READ，这样若有数据到达才能读取以供用户处理。该值为True时，每次读操作完毕后会自动调用channel.read()，从而有数据到达便能读取；否则，需要用户手动调用channel.read()。需要注意的是：当调用config.setAutoRead(boolean)方法时，如果状态由false变为true，将会调用channel.read()方法读取数据；由true变为false，将调用config.autoReadCleared()方法终止数据读取。
WRITE_BUFFER_HIGH_WATER_MARK
  Netty参数，写高水位标记，默认值64KB。如果Netty的写缓冲区中的字节超过该值，Channel的isWritable()返回False。
WRITE_BUFFER_LOW_WATER_MARK
  Netty参数，写低水位标记，默认值32KB。当Netty的写缓冲区中的字节超过高水位之后若下降到低水位，则Channel的isWritable()返回True。写高低水位标记使用户可以控制写入数据速度，从而实现流量控制。推荐做法是：每次调用channl.write(msg)方法首先调用channel.isWritable()判断是否可写。
MESSAGE_SIZE_ESTIMATOR
  Netty参数，消息大小估算器，默认为DefaultMessageSizeEstimator.DEFAULT。估算ByteBuf、ByteBufHolder和FileRegion的大小，其中ByteBuf和ByteBufHolder为实际大小，FileRegion估算值为0。该值估算的字节数在计算水位时使用，FileRegion为0可知FileRegion不影响高低水位。
SINGLE_EVENTEXECUTOR_PER_GROUP
  Netty参数，单线程执行ChannelPipeline中的事件，默认值为True。该值控制执行ChannelPipeline中执行ChannelHandler的线程。如果为Trye，整个pipeline由一个线程执行，这样不需要进行线程切换以及线程同步，是Netty4的推荐做法；如果为False，ChannelHandler中的处理过程会由Group中的不同线程执行。
```

#### 3.2.SocketChannel参数

```
SO_RCVBUF
  Socket参数，TCP数据接收缓冲区大小。该缓冲区即TCP接收滑动窗口，linux操作系统可使用命令：cat /proc/sys/net/ipv4/tcp_rmem查询其大小。一般情况下，该值可由用户在任意时刻设置，但当设置值超过64KB时，需要在连接到远端之前设置。
SO_SNDBUF
  Socket参数，TCP数据发送缓冲区大小。该缓冲区即TCP发送滑动窗口，linux操作系统可使用命令：cat /proc/sys/net/ipv4/tcp_smem查询其大小。
TCP_NODELAY
  TCP参数，立即发送数据，默认值为Ture（Netty默认为True而操作系统默认为False）。该值设置Nagle算法的启用，改算法将小的碎片数据连接成更大的报文来最小化所发送的报文的数量，如果需要发送一些较小的报文，则需要禁用该算法。Netty默认禁用该算法，从而最小化报文传输延时。
SO_KEEPALIVE
  Socket参数，连接保活，默认值为False。启用该功能时，TCP会主动探测空闲连接的有效性。可以将此功能视为TCP的心跳机制，需要注意的是：默认的心跳间隔是7200s即2小时。Netty默认关闭该功能。
SO_REUSEADDR
  Socket参数，地址复用，默认值False。有四种情况可以使用：(1).当有一个有相同本地地址和端口的socket1处于TIME_WAIT状态时，而你希望启动的程序的socket2要占用该地址和端口，比如重启服务且保持先前端口。(2).有多块网卡或用IP Alias技术的机器在同一端口启动多个进程，但每个进程绑定的本地IP地址不能相同。(3).单个进程绑定相同的端口到多个socket上，但每个socket绑定的ip地址不同。(4).完全相同的地址和端口的重复绑定。但这只用于UDP的多播，不用于TCP。
SO_LINGER
  Socket参数，关闭Socket的延迟时间，默认值为-1，表示禁用该功能。-1表示socket.close()方法立即返回，但OS底层会将发送缓冲区全部发送到对端。0表示socket.close()方法立即返回，OS放弃发送缓冲区的数据直接向对端发送RST包，对端收到复位错误。非0整数值表示调用socket.close()方法的线程被阻塞直到延迟时间到或发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。
IP_TOS
  IP参数，设置IP头部的Type-of-Service字段，用于描述IP包的优先级和QoS选项。
ALLOW_HALF_CLOSURE
  Netty参数，一个连接的远端关闭时本地端是否关闭，默认值为False。值为False时，连接自动关闭；为True时，触发ChannelInboundHandler的userEventTriggered()方法，事件为ChannelInputShutdownEvent。
```

#### 3.3.ServerSocketChannel参数

```
SO_RCVBUF
  已说明，需要注意的是：当设置值超过64KB时，需要在绑定到本地端口前设置。该值设置的是由ServerSocketChannel使用accept接受的SocketChannel的接收缓冲区。
SO_REUSEADDR
  已说明
SO_BACKLOG
  Socket参数，服务端接受连接的队列长度，如果队列已满，客户端连接将被拒绝。默认值，Windows为200，其他为128。
```

#### 3.4.DatagramChannel参数

```
SO_BROADCAST: Socket参数，设置广播模式。

SO_RCVBUF: 已说明

SO_SNDBUF:已说明

SO_REUSEADDR:已说明

IP_MULTICAST_LOOP_DISABLED:
  对应IP参数IP_MULTICAST_LOOP，设置本地回环接口的多播功能。由于IP_MULTICAST_LOOP返回True表示关闭，所以Netty加上后缀_DISABLED防止歧义。

IP_MULTICAST_ADDR:
  对应IP参数IP_MULTICAST_IF，设置对应地址的网卡为多播模式。

IP_MULTICAST_IF:
  对应IP参数IP_MULTICAST_IF2，同上但支持IPV6。

IP_MULTICAST_TTL:
  IP参数，多播数据报的time-to-live即存活跳数。

IP_TOS:
  已说明

DATAGRAM_CHANNEL_ACTIVE_ON_REGISTRATION:
  Netty参数，DatagramChannel注册的EventLoop即表示已激活。
```

