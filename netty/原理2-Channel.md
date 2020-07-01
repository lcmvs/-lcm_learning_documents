# Channel

[Netty-Channel注册源码分析](https://blog.csdn.net/xianmingsu/article/details/103857768)

Channel注册分为ServerSocketChannel和SockerChannel注册，服务端启动的时候会先进行ServerSocketChannel的注册，启动完成后，每当有一个客户端连接过来，会进行一次SocketChannel注册。

![https://img-blog.csdn.net/20160928165809260](assets/20160928165809260)

Channel是网络Socket进行联系的纽带，可以完成诸如读、写、连接、绑定等I/O操作。

JDK中的Channel是通讯的载体，而Netty中的Channel在此基础上进行封装从而赋予了Channel更多的能力，用户可以使用Channel进行以下操作：

- 查询Channel的状态。
- 配置Channel的参数。
- 进行Channel支持的I/O操作（read，write，connect，bind）。
- 获取对应的ChannelPipeline，从而可以自定义处理I/O事件或者其他请求。

为了保证在使用Channel或者处理I/O操作时不出现错误，以下几点需要特别注意：

1. **所有的I/O操作都是异步的**
   由于采用事件驱动的机制，所以Netty中的所有IO操作都是异步的。这意味着当我们调用一个IO操作时，方法会立即返回并不保证操作已经完成。由上一章Future的讲解中，我们知道，这些IO操作会返回一个ChannelFuture对象，我们需要通过添加监听者的方式执行操作完成后需执行的代码。

2. **Channel是有等级的**
   如果一个Channel由另一个Channel创建，那么他们之间形成父子关系。比如说，当ServerSocketChannel通过accept()方法接受一个SocketChannel时，那么SocketChannel的父亲是ServerSocketChannel，调用SocketChannel的parent()方法返回该ServerSocketChannel对象。

3. **可以使用向下转型获取子类的特定操作**
   某些子类Channel会提供一些所需的特定操作，可以向下转型到这样的子类，从而获得特定操作。比如说，对于UDP的数据报的传输，有特定的join()和leave()操作，我们可以向下转型到DatagramChannel从而使用这些操作。

4. **释放资源**
   当一个Channel不再使用时，须调用close()或者close(ChannelPromise)方法释放资源。

   

# 配置参数

## 通用参数

**CONNECT_TIMEOUT_MILLIS**
         Netty参数，连接超时毫秒数，默认值30000毫秒即30秒。

**MAX_MESSAGES_PER_READ**
         Netty参数，一次Loop读取的最大消息数，对于ServerChannel或者NioByteChannel，默认值为16，其他Channel默认值为1。默认值这样设置，是因为：ServerChannel需要接受足够多的连接，保证大吞吐量，NioByteChannel可以减少不必要的系统调用select。

**WRITE_SPIN_COUNT**
         Netty参数，一个Loop写操作执行的最大次数，默认值为16。也就是说，对于大数据量的写操作至多进行16次，如果16次仍没有全部写完数据，此时会提交一个新的写任务给EventLoop，任务将在下次调度继续执行。这样，其他的写请求才能被响应不会因为单个大数据量写请求而耽误。

**ALLOCATOR**
         Netty参数，ByteBuf的分配器，默认值为ByteBufAllocator.DEFAULT，4.0版本为UnpooledByteBufAllocator，4.1版本为PooledByteBufAllocator。该值也可以使用系统参数io.netty.allocator.type配置，使用字符串值："unpooled"，"pooled"。

**RCVBUF_ALLOCATOR**
         Netty参数，用于Channel分配接受Buffer的分配器，默认值为AdaptiveRecvByteBufAllocator.DEFAULT，是一个自适应的接受缓冲区分配器，能根据接受到的数据自动调节大小。可选值为FixedRecvByteBufAllocator，固定大小的接受缓冲区分配器。

**AUTO_READ**
         Netty参数，自动读取，默认值为True。Netty只在必要的时候才设置关心相应的I/O事件。对于读操作，需要调用channel.read()设置关心的I/O事件为OP_READ，这样若有数据到达才能读取以供用户处理。该值为True时，每次读操作完毕后会自动调用channel.read()，从而有数据到达便能读取；否则，需要用户手动调用channel.read()。需要注意的是：当调用config.setAutoRead(boolean)方法时，如果状态由false变为true，将会调用channel.read()方法读取数据；由true变为false，将调用config.autoReadCleared()方法终止数据读取。

**WRITE_BUFFER_HIGH_WATER_MARK**
         Netty参数，写高水位标记，默认值64KB。如果Netty的写缓冲区中的字节超过该值，Channel的isWritable()返回False。

**WRITE_BUFFER_LOW_WATER_MARK**
         Netty参数，写低水位标记，默认值32KB。当Netty的写缓冲区中的字节超过高水位之后若下降到低水位，则Channel的isWritable()返回True。写高低水位标记使用户可以控制写入数据速度，从而实现流量控制。推荐做法是：每次调用channl.write(msg)方法首先调用channel.isWritable()判断是否可写。

**MESSAGE_SIZE_ESTIMATOR**
         Netty参数，消息大小估算器，默认为DefaultMessageSizeEstimator.DEFAULT。估算ByteBuf、ByteBufHolder和FileRegion的大小，其中ByteBuf和ByteBufHolder为实际大小，FileRegion估算值为0。该值估算的字节数在计算水位时使用，FileRegion为0可知FileRegion不影响高低水位。

**SINGLE_EVENTEXECUTOR_PER_GROUP**
         Netty参数，单线程执行ChannelPipeline中的事件，默认值为True。该值控制执行ChannelPipeline中执行ChannelHandler的线程。如果为Trye，整个pipeline由一个线程执行，这样不需要进行线程切换以及线程同步，是Netty4的推荐做法；如果为False，ChannelHandler中的处理过程会由Group中的不同线程执行。

## SocketChannel参数

**SO_RCVBUF**
         Socket参数，TCP数据接收缓冲区大小。该缓冲区即TCP接收滑动窗口，linux操作系统可使用命令：`cat /proc/sys/net/ipv4/tcp_rmem`查询其大小。一般情况下，该值可由用户在任意时刻设置，但当设置值超过64KB时，需要在连接到远端之前设置。

**SO_SNDBUF**
         Socket参数，TCP数据发送缓冲区大小。该缓冲区即TCP发送滑动窗口，linux操作系统可使用命令：`cat /proc/sys/net/ipv4/tcp_smem`查询其大小。

**TCP_NODELAY**
         TCP参数，立即发送数据，默认值为Ture（Netty默认为True而操作系统默认为False）。该值设置Nagle算法的启用，改算法将小的碎片数据连接成更大的报文来最小化所发送的报文的数量，如果需要发送一些较小的报文，则需要禁用该算法。Netty默认禁用该算法，从而最小化报文传输延时。

**SO_KEEPALIVE**
         Socket参数，连接保活，默认值为False。启用该功能时，TCP会主动探测空闲连接的有效性。可以将此功能视为TCP的心跳机制，需要注意的是：默认的心跳间隔是7200s即2小时。Netty默认关闭该功能。

**SO_REUSEADDR**
         Socket参数，地址复用，默认值False。有四种情况可以使用：(1).当有一个有相同本地地址和端口的socket1处于TIME_WAIT状态时，而你希望启动的程序的socket2要占用该地址和端口，比如重启服务且保持先前端口。(2).有多块网卡或用IP Alias技术的机器在同一端口启动多个进程，但每个进程绑定的本地IP地址不能相同。(3).单个进程绑定相同的端口到多个socket上，但每个socket绑定的ip地址不同。(4).完全相同的地址和端口的重复绑定。但这只用于UDP的多播，不用于TCP。

**SO_LINGER**
          Netty对底层Socket参数的简单封装，关闭Socket的延迟时间，默认值为-1，表示禁用该功能。-1以及所有<0的数表示socket.close()方法立即返回，但OS底层会将发送缓冲区全部发送到对端。0表示socket.close()方法立即返回，OS放弃发送缓冲区的数据直接向对端发送RST包，对端收到复位错误。非0整数值表示调用socket.close()方法的线程被阻塞直到延迟时间到或发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。

**IP_TOS**
         IP参数，设置IP头部的Type-of-Service字段，用于描述IP包的优先级和QoS选项。

**ALLOW_HALF_CLOSURE**
         Netty参数，一个连接的远端关闭时本地端是否关闭，默认值为False。值为False时，连接自动关闭；为True时，触发ChannelInboundHandler的userEventTriggered()方法，事件为ChannelInputShutdownEvent。

## ServerSocketChannel

**SO_RCVBUF**
         已说明，需要注意的是：当设置值超过64KB时，需要在绑定到本地端口前设置。该值设置的是由ServerSocketChannel使用accept接受的SocketChannel的接收缓冲区。

**SO_REUSEADDR**
         已说明

**SO_BACKLOG**
         Socket参数，服务端接受连接的队列长度，如果队列已满，客户端连接将被拒绝。默认值，Windows为200，其他为128。

## DatagramChannel参数

**SO_BROADCAST**
         Socket参数，设置广播模式。

**SO_RCVBUF**
         已说明

**SO_SNDBUF**
         已说明

**SO_REUSEADDR**
         已说明

**IP_MULTICAST_LOOP_DISABLED**
         对应IP参数IP_MULTICAST_LOOP，设置本地回环接口的多播功能。由于IP_MULTICAST_LOOP返回True表示关闭，所以Netty加上后缀_DISABLED防止歧义。

**IP_MULTICAST_ADDR**
         对应IP参数IP_MULTICAST_IF，设置对应地址的网卡为多播模式。

**IP_MULTICAST_IF**
         对应IP参数IP_MULTICAST_IF2，同上但支持IPV6。

**IP_MULTICAST_TTL**
         IP参数，多播数据报的time-to-live即存活跳数。

**IP_TOS**
         已说明

**DATAGRAM_CHANNEL_ACTIVE_ON_REGISTRATION**
         Netty参数，DatagramChannel注册的EventLoop即表示已激活。



# AbstractChannel

```java
    private final Channel parent;   // 父Channel
    private final Unsafe unsafe;    
    private final DefaultChannelPipeline pipeline;  // 处理通道
    private final ChannelFuture succeededFuture = new SucceededChannelFuture(this, null);
    private final VoidChannelPromise voidPromise = new VoidChannelPromise(this, true);
    private final VoidChannelPromise unsafeVoidPromise = new VoidChannelPromise(this, false);
    private final CloseFuture closeFuture = new CloseFuture(this);

    private volatile SocketAddress localAddress;    // 本地地址
    private volatile SocketAddress remoteAddress;   // 远端地址
    private volatile EventLoop eventLoop;   // EventLoop线程
    private volatile boolean registered;    // 是否注册到EventLoop

    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
```

newUnsafe()和newChannelPipeline()可由子类覆盖实现。在Netty的实现中每一个Channel都有一个对应的Unsafe内部类：AbstractChannel--AbstractUnsafe，AbstractNioChannel--AbstractNioUnsafe等等，newUnsafe()方法正好用来生成这样的对应关系。ChannelPipeline将在之后讲解，这里先了解它的功能：作为用户处理器Handler的容器为用户提供自定义处理I/O事件的能力即为用户提供业务逻辑处理。AbstractChannel中对I/O事件的处理，都委托给ChannelPipeline处理，代码都如出一辙：

```java
    public ChannelFuture bind(SocketAddress localAddress) {
        return pipeline.bind(localAddress);
    }
```

对于Channel的实现来说，其中的内部类Unsafe才是关键，因为其中含有I/O事件处理的细节。AbstractUnsafe作为AbstractChannel的内部类，定义了I/O事件处理的基本框架，其中的细节留给子类实现。我们将依次对各个事件框架进行分析。

## register

```java
    public final void register(EventLoop eventLoop, final ChannelPromise promise) {
        if (isRegistered()) {
            promise.setFailure(...);    // 已经注册则失败
            return;
        }
        if (!isCompatible(eventLoop)) { // EventLoop不兼容当前Channel
            promise.setFailure(...);
            return;
        }
        AbstractChannel.this.eventLoop = eventLoop;
        // 当前线程为EventLoop线程直接执行；否则提交任务给EventLoop线程
        if (eventLoop.inEventLoop()) {
            register0(promise);
        } else {
            try {
                eventLoop.execute(() -> { register0(promise); });
            } catch (Throwable t) {
                closeForcibly();    // 异常时关闭Channel
                closeFuture.setClosed();    
                safeSetFailure(promise, t);
            }
        }
    }
```
register0()方法定义了注册到EventLoop的整体框架，整个流程如下：

 (1).注册的具体细节由doRegister()方法完成，子类中实现。

 (2).注册后将处理业务逻辑的用户Handler添加到ChannelPipeline。

 (3).异步结果设置为成功，触发Channel的Registered事件。

 (4).对于服务端接受的客户端连接，如果首次注册，触发Channel的Active事件，如果已设置autoRead，则调用beginRead()开始读取数据。

 对于(4)的是因为fireChannelActive()中也根据autoRead配置，调用了beginRead()方法。beginRead()方法其实也是一个框架，细节由doBeginRead()方法在子类中实现：

```java
    private void register0(ChannelPromise promise) {
        try {
            // 确保Channel没有关闭
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }
            boolean firstRegistration = neverRegistered;
            doRegister();   // 模板方法，细节由子类完成
            neverRegistered = false;
            registered = true;
            pipeline.invokeHandlerAddedIfNeeded();  // 将用户Handler添加到ChannelPipeline
            safeSetSuccess(promise);
            pipeline.fireChannelRegistered();   // 触发Channel注册事件
            if (isActive()) {
                // ServerSocketChannel接受的Channel此时已被激活
                if (firstRegistration) {
                    // 首次注册且激活触发Channel激活事件
                    pipeline.fireChannelActive();   
                } else if (config().isAutoRead()) {
                    beginRead();   // doBeginRead()可视为模板方法 
                }
            }
        } catch (Throwable t) {
            closeForcibly();     // 可视为模板方法
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
```

## bind

```java
    public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
        assertEventLoop();
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return; // 确保Channel没有关闭
        }
        boolean wasActive = isActive();
        try {
            doBind(localAddress);   // 模板方法，细节由子类完成
        } catch (Throwable t) {
            safeSetFailure(promise, t);
            closeIfClosed();
            return;
        }
        if (!wasActive && isActive()) { 
            invokeLater(() -> { pipeline.fireChannelActive(); });   // 触发Active事件
        }
        safeSetSuccess(promise);
    }
```

bind事件框架较为简单，主要完成在Channel绑定完成后触发Channel的Active事件。其中的invokeLater()方法向Channel注册到的EventLoop提交一个任务：

```java
    private void invokeLater(Runnable task) {
        try {
            eventLoop().execute(task);
        } catch (RejectedExecutionException e) {
            logger.warn("Can't invoke task later as EventLoop rejected it", e);
        }
    }
```

## disconnect

```java
    public final void disconnect(final ChannelPromise promise) {
        assertEventLoop();
        if (!promise.setUncancellable()) {
            return;
        }
        boolean wasActive = isActive();
        try {
            doDisconnect(); // 模板方法，细节由子类实现
        } catch (Throwable t) {
            safeSetFailure(promise, t);
            closeIfClosed();
            return;
        }
        if (wasActive && !isActive()) {
            invokeLater(() ->{ pipeline.fireChannelInactive(); });  // 触发Inactive事件
        }
        safeSetSuccess(promise);
        closeIfClosed(); // disconnect框架可能会调用close框架
    }
```

## close

close事件框架保证只有一个线程执行了真正关闭的doClose()方法，prepareToClose()做一些关闭前的清除工作并返回一个Executor，如果不为空，需要在该Executor里执行doClose0()方法；为空，则在当前线程执行（为什么这样设计？）。写缓冲区outboundBuffer同时也作为一个标记字段，为空表示Channel正在关闭此时禁止写操作。fireChannelInactiveAndDeregister()方法需要invokeLater()使用EventLoop执行，是因为其中会调用deRegister()方法触发Inactive事件，而事件执行需要在EventLoop中执行。

```java
    public final void close(final ChannelPromise promise) {
        assertEventLoop();
        close(promise, CLOSE_CLOSED_CHANNEL_EXCEPTION, CLOSE_CLOSED_CHANNEL_EXCEPTION, false);
    }

    private void close(final ChannelPromise promise, final Throwable cause,
                       final ClosedChannelException closeCause, final boolean notify) {
        if (!promise.setUncancellable()) {
            return;
        }
        final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;   
        if (outboundBuffer == null) {   // outboundBuffer作为一个标记，为空表示Channel正在关闭
            if (!(promise instanceof VoidChannelPromise)) {
                // 当Channel关闭时，将此次close异步请求结果也设置为成功
                closeFuture.addListener( (future) -> { promise.setSuccess(); });
            }
            return;
        }
        if (closeFuture.isDone()) {
            safeSetSuccess(promise);    // 已经关闭，保证底层close只执行一次
            return;
        }
        final boolean wasActive = isActive();
        this.outboundBuffer = null; // 设置为空禁止write操作，同时作为标记字段表示正在关闭
        Executor closeExecutor = prepareToClose();
        if (closeExecutor != null) {
            closeExecutor.execute(() -> {
                try {
                    doClose0(promise);  // prepareToClose返回的executor执行
                } finally {
                    invokeLater( () -> { // Channel注册的EventLoop执行
                        // 写缓冲队列中的数据全部设置失败
                        outboundBuffer.failFlushed(cause, notify);
                        outboundBuffer.close(closeCause);
                        fireChannelInactiveAndDeregister(wasActive);
                    });
                }
            });
        } else {    // 当前调用线程执行
            try {
                doClose0(promise);
            } finally {
                outboundBuffer.failFlushed(cause, notify);
                outboundBuffer.close(closeCause);
            }
            if (inFlush0) {
                invokeLater( () -> { fireChannelInactiveAndDeregister(wasActive); });
            } else {
                fireChannelInactiveAndDeregister(wasActive);
            }
        }
    }
    
    private void doClose0(ChannelPromise promise) {
        try {
            doClose();  // 模板方法，细节由子类实现
            closeFuture.setClosed();
            safeSetSuccess(promise);
        } catch (Throwable t) {
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
```

## deregister

deregister事件框架的处理流程很清晰，其中，使用invokeLater()方法是因为：用户可能会在ChannlePipeline中将当前Channel注册到新的EventLoop，确保ChannelPipiline事件和doDeregister()在同一个EventLoop完成（[?][3]）。
 需要注意的是：事件之间可能相互调用，比如：disconnect->close->deregister。

```java
    public final void deregister(final ChannelPromise promise) {
        assertEventLoop();
        deregister(promise, false);
    }

    private void deregister(final ChannelPromise promise, final boolean fireChannelInactive) {
        if (!promise.setUncancellable()) {
            return;
        }
        if (!registered) {
            safeSetSuccess(promise);    // 已经deregister
            return;
        }
        invokeLater( () -> {
            try {
                doDeregister(); // 模板方法，子类实现具体细节
            } catch (Throwable t) {
                logger.warn(...);
            } finally {
                if (fireChannelInactive) {
                    pipeline.fireChannelInactive(); // 根据参数触发Inactive事件
                }
                if (registered) {
                    registered = false;
                    pipeline.fireChannelUnregistered(); // 首次调用触发Unregistered事件
                }
                safeSetSuccess(promise);
            }
        });
    }
```

## write

```java
    public final void write(Object msg, ChannelPromise promise) {
        assertEventLoop();
        ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        if (outboundBuffer == null) {
            // 联系close操作，outboundBuffer为空表示Channel正在关闭，禁止写数据
            safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
            ReferenceCountUtil.release(msg);    // 释放msg 防止泄露
            return;
        }
        int size;
        try {
            msg = filterOutboundMessage(msg);
            size = pipeline.estimatorHandle().size(msg);
            if (size < 0) {
                size = 0;
            }
        } catch (Throwable t) {
            safeSetFailure(promise, t);
            ReferenceCountUtil.release(msg);
            return;
        }
        outboundBuffer.addMessage(msg, size, promise);
    }
```

## flush

flush事件中执行真正的底层写操作，Netty对于写的处理引入了一个写缓冲区ChannelOutboundBuffer，由该缓冲区控制Channel的可写状态，其具体实现，将会在缓冲区一章中分析。

```java
    public final void flush() {
        assertEventLoop();
        ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        if (outboundBuffer == null) {
            return; // Channel正在关闭直接返回
        }
        outboundBuffer.addFlush();  // 添加一个标记
        flush0();
    }

    protected void flush0() {
        if (inFlush0) {
            return;     // 正在flush返回防止多次调用
        }
        final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        if (outboundBuffer == null || outboundBuffer.isEmpty()) {
            return; // Channel正在关闭或者已没有需要写的数据
        }
        inFlush0 = true;
        if (!isActive()) {
            // Channel已经非激活，将所有进行中的写请求标记为失败
            try {
                if (isOpen()) {
                    outboundBuffer.failFlushed(FLUSH0_NOT_YET_CONNECTED_EXCEPTION, true);
                } else {
                    outboundBuffer.failFlushed(FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                }
            } finally {
                inFlush0 = false;
            }
            return;
        }
        try {
            doWrite(outboundBuffer);    // 模板方法，细节由子类实现
        } catch (Throwable t) {
            if (t instanceof IOException && config().isAutoClose()) {
                close(voidPromise(), t, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
            } else {
                outboundBuffer.failFlushed(t, true);
            }
        } finally {
            inFlush0 = false;
        }
    }
```

# AbstractNioChannel

```java
public abstract class AbstractNioChannel extends AbstractChannel {
    
    private final SelectableChannel ch; // 包装的JDK Channel
    protected final int readInterestOp; // Read事件，服务端OP_ACCEPT，其他OP_READ
    volatile SelectionKey selectionKey; // JDK Channel对应的选择键
    private volatile boolean inputShutdown; // Channel的输入关闭标记
    private volatile boolean readPending;   // 底层读事件进行标记
    
    private ChannelPromise connectPromise;  // 连接异步结果
    private ScheduledFuture<?> connectTimeoutFuture;    // 连接超时检测任务异步结果
    private SocketAddress requestedRemoteAddress;   // 连接的远端地址
    
    protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        super(parent);
        this.ch = ch;
        this.readInterestOp = readInterestOp;
        try {
            ch.configureBlocking(false);    // 设置非阻塞模式
        } catch (IOException e) {
            try {
                ch.close();
            } catch (IOException e2) {
                // log
            }
            throw new ChannelException("Failed to enter non-blocking mode.", e);
        }
    }
    
    public interface NioUnsafe extends Unsafe {
        SelectableChannel ch(); // 对应NIO中的JDK实现的Channel
        void finishConnect();   // 连接完成
        void read();    // 从JDK的Channel中读取数据
        void forceFlush(); 
    }
}
```

## doRegister

对于Register事件，当Channel属于NIO时，已经可以确定注册操作的全部细节：将Channel注册到给定NioEventLoop的selector上即可。对于Deregister事件，选择键执行cancle()操作，选择键表示JDK Channel和selctor的关系，调用cancle()终结这种关系，从而实现从NioEventLoop中Deregister。需要注意的是：cancle操作调用后，注册关系不会立即生效，而会将cancle的key移入selector的一个取消键集合，当下次调用select相关方法或一个正在进行的select调用结束时，会从取消键集合中移除该选择键，此时注销才真正完成。一个Cancle的选择键为无效键，调用它相关的方法会抛出CancelledKeyException。

```java
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                //第二个参数0表示注册时不关心任何事件，第三个参数为Netty的NioChannel对象本身。
                selectionKey = javaChannel().register(eventLoop().selector, 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // 选择键取消重新selectNow()，清除因取消操作而缓存的选择键
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    throw e;
                }
            }
        }
    }
    
    protected void doDeregister() throws Exception {
        eventLoop().cancel(selectionKey()); // 设置取消选择键
    }
```

## doBeginRead

对于NioChannel的beginRead事件，只需将Read事件设置为选择键所关心的事件，则之后的select()调用如果Channel对应的Read事件就绪，便会触发Netty的read()操作。

```java
    protected void doBeginRead() throws Exception {
        if (inputShutdown) {
            return; // Channel的输入关闭？什么情况下发生？
        }
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return; // 选择键被取消而不再有效
        }
        readPending = true; // 设置底层读事件正在进行
        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {
            // 选择键关心Read事件
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```

## doClose

此处的doClose操作主要处理了连接操作相关的后续处理。并没有实际关闭Channel，所以需要子类继续增加细节实现。AbstractNioChannel中还有关于创建DirectBuffer的方法，将在以后必要时进行分析。

```java
    protected void doClose() throws Exception {
        ChannelPromise promise = connectPromise;
        if (promise != null) {
            // 连接操作还在进行，但用户调用close操作
            promise.tryFailure(DO_CLOSE_CLOSED_CHANNEL_EXCEPTION);
            connectPromise = null;
        }
        ScheduledFuture<?> future = connectTimeoutFuture;
        if (future != null) {
            // 如果有连接超时检测任务，则取消
            future.cancel(false);
            connectTimeoutFuture = null;
        }
    }
```

## AbstractNioUnsafe

### connect

```java
public final void connect(final SocketAddress remoteAddress, final SocketAddress localAddress,
                          final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return; // Channel已被关闭
    }
    try {
        if (connectPromise != null) {
            throw new ConnectionPendingException(); // 已有连接操作正在进行
        }
        boolean wasActive = isActive();
        // 模板方法，细节子类完成
        if (doConnect(remoteAddress, localAddress)) {
            fulfillConnectPromise(promise, wasActive);  // 连接操作已完成
        } else {
            // 连接操作尚未完成
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;
            // 这部分代码为Netty的连接超时机制
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
                connectTimeoutFuture = eventLoop().schedule(() -> {
                    ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                    ConnectTimeoutException cause = new ConnectTimeoutException("...");
                    if (connectPromise != null && connectPromise.tryFailure(cause)) {
                        close(voidPromise());
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }

            promise.addListener((ChannelFutureListener) (future) -> {
                if (future.isCancelled()) {
                    // 连接操作取消则连接超时检测任务取消
                    if (connectTimeoutFuture != null) {
                        connectTimeoutFuture.cancel(false);
                    }
                    connectPromise = null;
                    close(voidPromise());
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```

### finishConnect

finishConnect()只由EventLoop处理就绪selectionKey的OP_CONNECT事件时调用，从而完成连接操作。注意：连接操作被取消或者超时不会使该方法被调用。

```java
    public final void finishConnect() {
        assert eventLoop().inEventLoop();
        try {
            boolean wasActive = isActive();
            doFinishConnect();  // 模板方法
            fulfillConnectPromise(connectPromise, wasActive);   // 首次Active触发Active事件
        } catch (Throwable t) {
            fulfillConnectPromise(connectPromise, annotateConnectException(...));
        } finally {
            if (connectTimeoutFuture != null) {
                connectTimeoutFuture.cancel(false); // 连接完成，取消超时检测任务
            }
            connectPromise = null;
        }
    }
```

finishConnect()只由EventLoop处理就绪selectionKey的OP_CONNECT事件时调用，从而完成连接操作。

注意：连接操作被取消或者超时不会使该方法被调用。
Flush事件细节：

```java
    protected final void flush0() {
        if (isFlushPending()) {
            return; // 已经有flush操作，返回
        }
        super.flush0(); // 调用父类方法
    }

    public final void forceFlush() {
        super.flush0(); // 调用父类方法
    }

    private boolean isFlushPending() {
        SelectionKey selectionKey = selectionKey();
        return selectionKey.isValid() && 
                    (selectionKey.interestOps() & SelectionKey.OP_WRITE) != 0;
    }
```

### flush

```java
    protected final void flush0() {
        if (isFlushPending()) {
            return; // 已经有flush操作，返回
        }
        super.flush0(); // 调用父类方法
    }

    public final void forceFlush() {
        super.flush0(); // 调用父类方法
    }

    private boolean isFlushPending() {
        SelectionKey selectionKey = selectionKey();
        return selectionKey.isValid() && 
                    (selectionKey.interestOps() & SelectionKey.OP_WRITE) != 0;
    }
```

forceFlush()方法由EventLoop处理就绪selectionKey的OP_WRITE事件时调用，将缓冲区中的数据写入Channel。isFlushPending()方法容易导致困惑：为什么selectionKey关心OP_WRITE事件表示正在Flush呢？OP_WRITE表示通道可写，而一般情况下通道都可写，如果selectionKey一直关心OP_WRITE事件，那么将不断从select()方法返回从而导致死循环。Netty使用一个写缓冲区，write操作将数据放入缓冲区中，flush时设置selectionKey关心OP_WRITE事件，完成后取消关心OP_WRITE事件。所以，如果selectionKey关心OP_WRITE事件表示此时正在Flush数据。
 AbstractNioUnsafe还有最后一个方法removeReadOp()：

```java
    protected final void removeReadOp() {
        SelectionKey key = selectionKey();
        if (!key.isValid()) {
            return; // selectionKey已被取消
        }
        int interestOps = key.interestOps();
        if ((interestOps & readInterestOp) != 0) {
            key.interestOps(interestOps & ~readInterestOp); // 设置为不再感兴趣
        }
    }
```

Netty中将服务端的OP_ACCEPT和客户端的Read统一抽象为Read事件，在NIO底层I/O事件使用bitmap表示，一个二进制位对应一个I/O事件。当一个二进制位为1时表示关心该事件，readInterestOp的二进制表示只有1位为1，所以体会interestOps & ~readInterestOp的含义，可知removeReadOp()的功能是设置SelectionKey不再关心Read事件。类似的，还有setReadOp()、removeWriteOp()、setWriteOp()等等。

# AbstractNioMessageChannel

```java
public abstract class AbstractNioMessageChannel extends AbstractNioChannel {
    
}
```

AbstractNioMessageChannel是底层数据为消息的NioChannel。在Netty中，服务端Accept的一个Channel被认为是一条消息，UDP数据报也是一条消息。该类主要完善flush事件框架的doWrite细节和实现read事件框架（在内部类NioMessageUnsafe完成）。

## read

read事件框架的流程已在代码中注明，需要注意的是读取消息的细节doReadMessages(readBuf)方法由子类实现。
 我们主要分析NioServerSocketChannel，它不支持doWrite()操作，所以我们不再分析本类的flush事件框架的doWrite细节方法，直接转向下一个目标：NioServerSocketChannel。

```java
    public void read() {
        assert eventLoop().inEventLoop();
        final ChannelConfig config = config();
        if (!config.isAutoRead() && !isReadPending()) {
            // 此时读操作不被允许，既没有配置autoRead也没有底层读事件进行
            removeReadOp(); // 清除read事件，不再关心
            return;
        }
        
        final int maxMessagesPerRead = config.getMaxMessagesPerRead();
        final ChannelPipeline pipeline = pipeline();
        boolean closed = false;
        Throwable exception = null;
        try {
            try {
                for (;;) {
                    int localRead = doReadMessages(readBuf); // 模板方法，读取消息
                    if (localRead == 0) { // 没有数据可读
                        break;  
                    }
                    if (localRead < 0) { // 读取出错
                        closed = true;  
                        break;
                    }
                    if (!config.isAutoRead()) { //没有设置AutoRead
                        break;
                    }
                    if (readBuf.size() >= maxMessagesPerRead) { // 达到最大可读数
                        break;
                    }
                }
            } catch (Throwable t) {
                exception = t;
            }
            
            setReadPending(false);  // 已没有底层读事件
            int size = readBuf.size();
            for (int i = 0; i < size; i ++) {
                pipeline.fireChannelRead(readBuf.get(i));   //触发ChannelRead事件，用户处理
            }
            readBuf.clear();
            // ChannelReadComplete事件中如果配置autoRead则会调用beginRead，从而不断进行读操作
            pipeline.fireChannelReadComplete(); // 触发ChannelReadComplete事件，用户处理

            if (exception != null) {
                if (exception instanceof IOException && !(exception instanceof PortUnreachableException)) {
                    // ServerChannel异常也不能关闭，应该恢复读取下一个客户端
                    closed = !(AbstractNioMessageChannel.this instanceof ServerChannel);
                }
                pipeline.fireExceptionCaught(exception);
            }

            if (closed) {
                if (isOpen()) {
                    close(voidPromise());   // 非serverChannel且打开则关闭
                }
            }
        } finally {
            if (!config.isAutoRead() && !isReadPending()) {
                // 既没有配置autoRead也没有底层读事件进行
                removeReadOp();
            }
        }
    }
```

# NioServerSocketChannel

## OP_ACCEPT

```java
    public NioServerSocketChannel(ServerSocketChannel channel) {
        super(null, channel, SelectionKey.OP_ACCEPT);
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
```

其中的SelectionKey.OP_ACCEPT最为关键，Netty正是在此处将NioServerSocketChannel的read事件定义为NIO底层的OP_ACCEPT，统一完成read事件的抽象。



你肯定已经使用过NioServerSocketChannel，作为处于Channel最底层的子类，NioServerSocketChannel会实现I/O事件框架的底层细节。首先需要注意的是：NioServerSocketChannel**只支持bind、read和close操作**。

```java
   protected void doBind(SocketAddress localAddress) throws Exception {
        if (PlatformDependent.javaVersion() >= 7) { // JDK版本1.7以上
            javaChannel().bind(localAddress, config.getBacklog());
        } else {
            javaChannel().socket().bind(localAddress, config.getBacklog());
        }
    }
    
    protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = javaChannel().accept();
        try {
            if (ch != null) {
                // 一个NioSocketChannel为一条消息
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);
            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }
        return 0;
    }
    
    protected void doClose() throws Exception {
        javaChannel().close();
    }
```

其中的实现，都是调用JDK的Channel的方法，从而实现了最底层的细节。需要注意的是：此处的doReadMessages()方法每次最多返回一个消息（客户端连接），由此可知NioServerSocketChannel的read操作一次至多处理的连接数为config.getMaxMessagesPerRead()，也就是参数值MAX_MESSAGES_PER_READ。此外doClose()覆盖了AbstractNioChannel的实现，因为NioServerSocketChannel不支持connect操作，所以不需要连接超时处理。


# AbstractNioByteChannel

```java
    protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
        super(parent, ch, SelectionKey.OP_READ);
    }
```

其中的SelectionKey.OP_READ，说明AbstractNioByteChannel的read事件为NIO底层的OP_READ事件。

## read

AbstractNioByteChannel的read事件框架处理流程与AbstractNioMessageChannel的稍有不同：AbstractNioMessageChannel依次读取Message，最后统一触发ChannelRead事件；而AbstractNioByteChannel每读取到一定字节就触发ChannelRead事件。这是因为，AbstractNioMessageChannel需求高吞吐量，特别是ServerSocketChannel需要尽可能多地接受连接；而AbstractNioByteChannel需求快响应，要尽可能快地响应远端请求。

```java
    public final void read() {
        final ChannelConfig config = config();
        if (!config.isAutoRead() && !isReadPending()) {
            // 此时读操作不被允许，既没有配置autoRead也没有底层读事件进行
            removeReadOp();
            return;
        }

        final ChannelPipeline pipeline = pipeline();
        final ByteBufAllocator allocator = config.getAllocator();
        final int maxMessagesPerRead = config.getMaxMessagesPerRead();
        RecvByteBufAllocator.Handle allocHandle = this.allocHandle;
        if (allocHandle == null) {
            this.allocHandle = allocHandle = config.getRecvByteBufAllocator().newHandle();
        }

        ByteBuf byteBuf = null;
        int messages = 0;
        boolean close = false;
        try {
            int totalReadAmount = 0;
            boolean readPendingReset = false;
            do {
                byteBuf = allocHandle.allocate(allocator);  // 创建一个ByteBuf
                int writable = byteBuf.writableBytes(); 
                int localReadAmount = doReadBytes(byteBuf); // 模板方法，子类实现细节
                if (localReadAmount <= 0) { // 没有数据可读
                    byteBuf.release();
                    byteBuf = null;
                    close = localReadAmount < 0; // 读取数据量为负数表示对端已经关闭
                    break;
                }
                if (!readPendingReset) {
                    readPendingReset = true;
                    setReadPending(false);  // 没有底层读事件进行
                    // 此时，若autoRead关闭则必须调用beginRead，read操作才会读取数据
                }
                pipeline.fireChannelRead(byteBuf);  // 触发ChannelRead事件，用户处理
                byteBuf = null;

                if (totalReadAmount >= Integer.MAX_VALUE - localReadAmount) {   // 防止溢出
                    totalReadAmount = Integer.MAX_VALUE;
                    break;
                }
                totalReadAmount += localReadAmount;

                if (!config.isAutoRead()) { // 没有配置AutoRead
                    break;
                }
                if (localReadAmount < writable) {   // 读取数小于可写数，可能接受缓冲区已完全耗尽
                    break;
                }
            } while (++ messages < maxMessagesPerRead);

            // ReadComplete结束时，如果开启autoRead则会调用beginRead，从而可以继续read
            pipeline.fireChannelReadComplete();
            allocHandle.record(totalReadAmount);

            if (close) {
                closeOnRead(pipeline);
                close = false;
            }
        } catch (Throwable t) {
            handleReadException(pipeline, byteBuf, t, close);
        } finally {
            if (!config.isAutoRead() && !isReadPending()) {
                // 既没有配置autoRead也没有底层读事件进行
                removeReadOp(); 
            }
        }
    }
```

## doWrite

```java
    protected final Object filterOutboundMessage(Object msg) {
        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            if (buf.isDirect()) {
                return msg;
            }
            return newDirectBuffer(buf); // 非DirectBuf转为DirectBuf
        }
        if (msg instanceof FileRegion) {
            return msg;
        }
        throw new UnsupportedOperationException("...");
    }
```

可知，Netty支持的写数据类型只有两种：**DirectBuffer**和**FileRegion**。我们再看这些数据怎么写到Channel上，也就是doWrite()方法：

```java
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        int writeSpinCount = -1;
        boolean setOpWrite = false;
        for (;;) {
            Object msg = in.current();
            if (msg == null) {  // 数据已全部写完
                clearOpWrite();     // 清除OP_WRITE事件
                return;
            }

            if (msg instanceof ByteBuf) {
                ByteBuf buf = (ByteBuf) msg;
                int readableBytes = buf.readableBytes();
                if (readableBytes == 0) {
                    in.remove();
                    continue;
                }

                boolean done = false;
                long flushedAmount = 0;
                if (writeSpinCount == -1) {
                    writeSpinCount = config().getWriteSpinCount();
                }
                for (int i = writeSpinCount - 1; i >= 0; i --) {
                    int localFlushedAmount = doWriteBytes(buf); // 模板方法，子类实现细节
                    if (localFlushedAmount == 0) {
                        // NIO在非阻塞模式下写操作可能返回0表示未写入数据
                        setOpWrite = true;
                        break;
                    }

                    flushedAmount += localFlushedAmount;
                    if (!buf.isReadable()) {
                        // ByteBuf不可读，此时数据已写完
                        done = true;
                        break;
                    }
                }
                
                in.progress(flushedAmount); // 记录进度
                if (done) {
                    in.remove();    // 完成时，清理缓冲区
                } else {
                    break;  // 跳出循环执行incompleteWrite()
                }
            } else if (msg instanceof FileRegion) {
                // ....
            } else {
                throw new Error();  // 其他类型不支持
            }
        }
        incompleteWrite(setOpWrite);
    }
```

代码中省略了对FileRegion的处理，FileRegion是Netty对NIO底层的FileChannel的封装，负责将File中的数据写入到WritableChannel中。FileRegion的默认实现是DefaultFileRegion，如果你很感兴趣它的实现，可以自行查阅。
 我们主要分析对ByteBuf的处理。doWrite的流程简洁明了，核心操作是模板方法doWriteBytes(buf)，将ByteBuf中的数据写入到Channel，由于NIO底层的写操作返回已写入的数据量，在非阻塞模式下该值可能为0，此时会调用incompleteWrite()方法：

```java
    protected final void incompleteWrite(boolean setOpWrite) {
        if (setOpWrite) {
            setOpWrite();   // 设置继续关心OP_WRITE事件
        } else {
            // 此时已进行写操作次数writeSpinCount，但并没有写完
            Runnable flushTask = this.flushTask;
            if (flushTask == null) {
                flushTask = this.flushTask = (Runnable) () -> { flush(); };
            }
            // 再次提交一个flush()任务
            eventLoop().execute(flushTask);
        }
    }
```

该方法分两种情况处理，在上文提到的第一种情况（实际写0数据）下，设置SelectionKey继续关心OP_WRITE事件从而继续进行写操作；第二种情况下，也就是写操作进行次数达到配置中的writeSpinCount值但尚未写完，此时向EventLoop提交一个新的flush任务，此时可以响应其他请求，从而提交响应速度。这样的处理，不会使大数据的写操作占用全部资源而使其他请求得不到响应，可见这是一个较为公平的处理。这里引出一个问题：使用Netty如何搭建高性能文件服务器？

# NioSocketChannel

## doBind

```java
    protected void doBind(SocketAddress localAddress) throws Exception {
        doBind0(localAddress);
    }

    private void doBind0(SocketAddress localAddress) throws Exception {
        if (PlatformDependent.javaVersion() >= 7) {
            javaChannel().bind(localAddress);   // JDK版本1.7以上
        } else {
            javaChannel().socket().bind(localAddress);
        }
    }
```

## doConnect

```java
    protected boolean doConnect(SocketAddress remoteAddress, 
                                        SocketAddress localAddress) throws Exception {
        if (localAddress != null) {
            doBind0(localAddress);
        }

        boolean success = false;
        try {
            boolean connected = javaChannel().connect(remoteAddress);
            if (!connected) {
                // 设置关心OP_CONNECT事件，事件就绪时调用finishConnect()
                selectionKey().interestOps(SelectionKey.OP_CONNECT);
            }
            success = true;
            return connected;
        } finally {
            if (!success) {
                doClose();
            }
        }
    }

    protected void doFinishConnect() throws Exception {
        if (!javaChannel().finishConnect()) {
            throw new Error();
        }
    }
```

## doDisconnect

```java
    protected void doDisconnect() throws Exception {
        doClose();
    }

    protected void doClose() throws Exception {
        super.doClose();    // AbstractNioChannel中关于连接超时的处理
        javaChannel().close();
    }
```

## read和write

```java
    protected int doReadBytes(ByteBuf byteBuf) throws Exception {
        return byteBuf.writeBytes(javaChannel(), byteBuf.writableBytes());
    }

    protected int doWriteBytes(ByteBuf buf) throws Exception {
        final int expectedWrittenBytes = buf.readableBytes();
        return buf.readBytes(javaChannel(), expectedWrittenBytes);
    }

    protected long doWriteFileRegion(FileRegion region) throws Exception {
        final long position = region.transfered();
        return region.transferTo(javaChannel(), position);
    }
```

## doWrite

```java
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        for (;;) {
            int size = in.size();
            if (size == 0) {
                clearOpWrite(); // 所有数据已写完，不再关心OP_WRITE事件
                break;
            }
            long writtenBytes = 0;
            boolean done = false;
            boolean setOpWrite = false;

            ByteBuffer[] nioBuffers = in.nioBuffers();
            int nioBufferCnt = in.nioBufferCount();
            long expectedWrittenBytes = in.nioBufferSize();
            SocketChannel ch = javaChannel();

            switch (nioBufferCnt) {
                case 0: // 没有ByteBuffer，也就是只有FileRegion
                    super.doWrite(in);  // 使用父类方法进行普通处理
                    return;
                case 1: // 只有一个ByteBuffer，此时的处理等效于父类方法的处理
                    ByteBuffer nioBuffer = nioBuffers[0];
                    for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                        final int localWrittenBytes = ch.write(nioBuffer);
                        if (localWrittenBytes == 0) {
                            setOpWrite = true;
                            break;
                        }
                        expectedWrittenBytes -= localWrittenBytes;
                        writtenBytes += localWrittenBytes;
                        if (expectedWrittenBytes == 0) {
                            done = true;
                            break;
                        }
                    }
                    break;
                default: // 多个ByteBuffer，采用gathering方法处理
                    for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                        // gathering方法，此时一次写多个ByteBuffer
                        final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                        if (localWrittenBytes == 0) {
                            setOpWrite = true;
                            break;
                        }
                        expectedWrittenBytes -= localWrittenBytes;
                        writtenBytes += localWrittenBytes;
                        if (expectedWrittenBytes == 0) {
                            done = true;
                            break;
                        }
                    }
                    break;
            }
            in.removeBytes(writtenBytes);   // 清理缓冲区
            if (!done) {
                incompleteWrite(setOpWrite);    // 写操作并没有完成
                break;
            }
        }
    }
```