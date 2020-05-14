

[Netty 源码解析系列](https://www.javadoop.com/post/netty-part-1)



# ChannelPipeline



# 线程池

```java
// EchoClient 代码最开始：
EventLoopGroup group = new NioEventLoopGroup();

// EchoServer 代码最开始：
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

## NioEventLoopGroup

```java
public NioEventLoopGroup() {
    this(0);
}
public NioEventLoopGroup(int nThreads) {
    this(nThreads, (Executor) null);
}

...

// 参数最全的构造方法
public NioEventLoopGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory,
                         final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory,
                         final RejectedExecutionHandler rejectedExecutionHandler) {
    // 调用父类的构造方法
    super(nThreads, executor, chooserFactory, selectorProvider, selectStrategyFactory, rejectedExecutionHandler);
}
```

构造方法中的各个参数：

- nThreads：这个最简单，就是线程池中的线程数，也就是 NioEventLoop 的实例数量。
- executor：我们知道，我们本身就是要构造一个线程池（Executor），为什么这里传一个 executor 实例呢？它其实不是给线程池用的，而是给 NioEventLoop 用的，以后再说。
- chooserFactory：当我们提交一个任务到线程池的时候，线程池需要选择（choose）其中的一个线程来执行这个任务，这个就是用来实现选择策略的。
- selectorProvider：这个简单，我们需要通过它来实例化 JDK 的 Selector，可以看到每个线程池都持有一个 selectorProvider 实例。
- selectStrategyFactory：这个涉及到的是线程池中线程的工作流程，在介绍 NioEventLoop 的时候会说。
- rejectedExecutionHandler：这个也是线程池的好朋友了，用于处理线程池中没有可用的线程来执行任务的情况。在 Netty 中稍微有一点点不一样，这个是给 NioEventLoop 实例用的，以后我们再详细介绍。



## NioEventLoop

```java
private Selector selector;
private Selector unwrappedSelector;
private SelectedSelectionKeySet selectedKeys;

private final SelectorProvider provider;

private final AtomicBoolean wakenUp = new AtomicBoolean();

private final SelectStrategy selectStrategy;

private volatile int ioRatio = 50;
private int cancelledKeys;
private boolean needsToSelectAgain;
```