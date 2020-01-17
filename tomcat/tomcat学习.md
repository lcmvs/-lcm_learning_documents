

## Tomcat线程池

tomcat线程池和jdk默认线程池最大不同那就是，阻塞队列和最大线程池这两个条件的顺序问题。

jdk中的线程池，当达到核心线程数量，就进入阻塞队列，阻塞队列满了，才继续创建新的工作线程，直到达到最大线程数量，拒绝新的任务。

tomcat的线程池，达到核心线程数量，则是会先继续创建新的工作线程，直到达到最大线程数量，才进入阻塞队列 ，阻塞队列满了才拒绝新的任务。

[tomcat线程池策略](https://segmentfault.com/a/1190000008052008)

- 场景1：接受一个请求，此时tomcat启动的线程数还没有达到corePoolSize(`tomcat里头叫minSpareThreads`)，tomcat会启动一个线程来处理该请求；
- 场景2：接受一个请求，此时tomcat启动的线程数已经达到了corePoolSize，tomcat把该请求放入队列(`offer`)，如果放入队列成功，则返回，放入队列不成功，则尝试增加工作线程，在当前线程个数<maxThreads的时候，可以继续增加线程来处理，超过maxThreads的时候，则继续往等待队列里头放，等待队列放不进去，则抛出RejectedExecutionException；



### 重写阻塞队列

tomcat使用了一个无界队列作为线程池的阻塞队列，但是重写了队列的offer方法，**当其线程池大小小于maximumPoolSize的时候，返回false**。

```java
public class TaskQueue extends LinkedBlockingQueue<Runnable> {
    @Override
    public boolean offer(Runnable o) {
      //we can't do any checks
        if (parent==null) return super.offer(o);
        //we are maxed out on threads, simply queue the object
        if (parent.getPoolSize() == parent.getMaximumPoolSize()) return super.offer(o);
        //we have idle threads, just add it to the queue
        if (parent.getSubmittedCount()<=(parent.getPoolSize())) return super.offer(o);
        //if we have less threads than maximum force creation of a new thread
        if (parent.getPoolSize()<parent.getMaximumPoolSize()) return false;
        //if we reached here, we need to add it to the queue
        return super.offer(o);
    }
}    
```



### 捕捉任务拒绝策略

```java
public class ThreadPoolExecutor extends java.util.concurrent.ThreadPoolExecutor {
/**
 * Executes the given command at some time in the future.  The command
 * may execute in a new thread, in a pooled thread, or in the calling
 * thread, at the discretion of the <tt>Executor</tt> implementation.
 * If no threads are available, it will be added to the work queue.
 * If the work queue is full, the system will wait for the specified
 * time and it throw a RejectedExecutionException if the queue is still
 * full after that.
 */
public void execute(Runnable command, long timeout, TimeUnit unit) {
    submittedCount.incrementAndGet();
    try {
        super.execute(command);
    } catch (RejectedExecutionException rx) {
        if (super.getQueue() instanceof TaskQueue) {
            final TaskQueue queue = (TaskQueue)super.getQueue();
            try {
                if (!queue.force(command, timeout, unit)) {
                    submittedCount.decrementAndGet();
                    throw new RejectedExecutionException(sm.getString("threadPoolExecutor.queueFull"));
                }
            } catch (InterruptedException x) {
                submittedCount.decrementAndGet();
                throw new RejectedExecutionException(x);
            }
        } else {
            submittedCount.decrementAndGet();
            throw rx;
        }

    }
}
}
```



## 最大连接数实现

```java
public abstract class AbstractEndpoint<S,U> {
    private int maxConnections = 10000;
    public void setMaxConnections(int maxCon) {
        this.maxConnections = maxCon;
        LimitLatch latch = this.connectionLimitLatch;
        if (latch != null) {
            // Update the latch that enforces this
            if (maxCon == -1) {
                releaseConnectionLatch();
            } else {
                latch.setLimit(maxCon);
            }
        } else if (maxCon > 0) {
            initializeConnectionLatch();
        }
    }
}    
```



### 共享锁LimitLatch

使用一个原子变量limit，cas的方式去增减，小于limit，允许共享，否则获取锁失败。

```java
ublic class LimitLatch {

    private static final Log log = LogFactory.getLog(LimitLatch.class);

    private class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1L;

        public Sync() {
        }

        @Override
        protected int tryAcquireShared(int ignored) {
            long newCount = count.incrementAndGet();
            if (!released && newCount > limit) {
                // Limit exceeded
                count.decrementAndGet();
                return -1;
            } else {
                return 1;
            }
        }

        @Override
        protected boolean tryReleaseShared(int arg) {
            count.decrementAndGet();
            return true;
        }
    }

    private final Sync sync;
    private final AtomicLong count;
    private volatile long limit;
    private volatile boolean released = false;
}
```



## 使用NIO

[Tomcat 中的 NIO 源码分析](https://www.javadoop.com/post/tomcat-nio)
![Tomcat NIO 线程模型](assets/0.png)

```java
public abstract class AbstractEndpoint<S,U> {
protected int acceptorThreadCount = 1;
private int pollerThreadCount = Math.min(2,Runtime.getRuntime().availableProcessors());
public void bind() throws Exception {
    initServerSocket();

    // 初始化acceptor, poller线程数
    if (acceptorThreadCount == 0) {
        acceptorThreadCount = 1;
    }
    if (pollerThreadCount <= 0) {
        pollerThreadCount = 1;
    }
    setStopLatch(new CountDownLatch(pollerThreadCount));

    // Initialize SSL if needed
    initialiseSsl();

    selectorPool.open();
}

protected void initServerSocket() throws Exception {
    if (!getUseInheritedChannel()) {
        //开启 ServerSocketChannel
        serverSock = ServerSocketChannel.open();
        socketProperties.setProperties(serverSock.socket());
        InetSocketAddress addr = new InetSocketAddress(getAddress(), getPortWithOffset());
        // ServerSocketChannel 绑定地址、端口，
   		// 第二个参数 backlog 默认为 100，超过 100 的时候，新连接会被拒绝(不过源码注释也说了，这个值的真实语义取决于具体实现)
        serverSock.socket().bind(addr,getAcceptCount());
    } else {
        // Retrieve the channel provided by the OS
        Channel ic = System.inheritedChannel();
        if (ic instanceof ServerSocketChannel) {
            serverSock = (ServerSocketChannel) ic;
        }
        if (serverSock == null) {
            throw new IllegalArgumentException(sm.getString("endpoint.init.bind.inherited"));
        }
    }
    serverSock.configureBlocking(true); //mimic APR behavior
}
}    
```
123

```java
public void startInternal() throws Exception {

    if (!running) {
        running = true;
        paused = false;

        processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getProcessorCache());
        eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                        socketProperties.getEventCache());
        nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getBufferPool());

        //创建工作线程组
        if ( getExecutor() == null ) {
            createExecutor();
        }

        //初始化共享锁，限制最大连接数
        initializeConnectionLatch();

        //开启poller线程
        pollers = new Poller[getPollerThreadCount()];
        for (int i=0; i<pollers.length; i++) {
            pollers[i] = new Poller();
            Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();
        }

        //开启acceptor线程
        startAcceptorThreads();
    }
}
```





### acceptor线程

```java
public class Acceptor<U> implements Runnable {
    private static final Log log = LogFactory.getLog(Acceptor.class);
    private static final StringManager sm = StringManager.getManager(Acceptor.class);

    private static final int INITIAL_ERROR_DELAY = 50;
    private static final int MAX_ERROR_DELAY = 1600;

    private final AbstractEndpoint<?,U> endpoint;
    private String threadName;
    protected volatile AcceptorState state = AcceptorState.NEW;
public void run() {

    int errorDelay = 0;

    // Loop until we receive a shutdown command
    while (endpoint.isRunning()) {

        // Loop if endpoint is paused
        while (endpoint.isPaused() && endpoint.isRunning()) {
            state = AcceptorState.PAUSED;
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                // Ignore
            }
        }

        if (!endpoint.isRunning()) {
            break;
        }
        state = AcceptorState.RUNNING;

        try {
            //达到最大连接数，等待
            endpoint.countUpOrAwaitConnection();

            // Endpoint might have been paused while waiting for latch
            // If that is the case, don't accept new connections
            if (endpoint.isPaused()) {
                continue;
            }

            U socket = null;
            try {
                // 接受连接
                socket = endpoint.serverSocketAccept();
            } catch (Exception ioe) {
                // We didn't get a socket
                endpoint.countDownConnection();
                if (endpoint.isRunning()) {
                    // Introduce delay if necessary
                    errorDelay = handleExceptionWithDelay(errorDelay);
                    // re-throw
                    throw ioe;
                } else {
                    break;
                }
            }
            // Successful accept, reset the error delay
            errorDelay = 0;

            // Configure the socket
            if (endpoint.isRunning() && !endpoint.isPaused()) {
                // 封装成一个channel，注册到poller线程
                if (!endpoint.setSocketOptions(socket)) {
                    endpoint.closeSocket(socket);
                }
            } else {
                endpoint.destroySocket(socket);
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            String msg = sm.getString("endpoint.accept.fail");
            // APR specific.
            // Could push this down but not sure it is worth the trouble.
            if (t instanceof Error) {
                Error e = (Error) t;
                if (e.getError() == 233) {
                    // Not an error on HP-UX so log as a warning
                    // so it can be filtered out on that platform
                    // See bug 50273
                    log.warn(msg, t);
                } else {
                    log.error(msg, t);
                }
            } else {
                    log.error(msg, t);
            }
        }
    }
    state = AcceptorState.ENDED;
}
}    
```



```java
protected boolean setSocketOptions(SocketChannel socket) {
    // Process the connection
    try {
        //disable blocking, APR style, we are gonna be polling it
        socket.configureBlocking(false);
        Socket sock = socket.socket();
        socketProperties.setProperties(sock);

        NioChannel channel = nioChannels.pop();
        if (channel == null) {
            SocketBufferHandler bufhandler = new SocketBufferHandler(
                    socketProperties.getAppReadBufSize(),
                    socketProperties.getAppWriteBufSize(),
                    socketProperties.getDirectBuffer());
            if (isSSLEnabled()) {
                channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
            } else {
                channel = new NioChannel(socket, bufhandler);
            }
        } else {
            channel.setIOChannel(socket);
            channel.reset();
        }
        getPoller0().register(channel);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        try {
            log.error(sm.getString("endpoint.socketOptionsError"), t);
        } catch (Throwable tt) {
            ExceptionUtils.handleThrowable(tt);
        }
        // Tell to close the socket
        return false;
    }
    return true;
}
```

### poller线程

Poller是NioEndPoint的成员内部类

```java
public class Poller implements Runnable {

    private Selector selector;
    private final SynchronizedQueue<PollerEvent> events =
        new SynchronizedQueue<>();

    private volatile boolean close = false;
    private long nextExpiration = 0;//optimize expiration handling

    private AtomicLong wakeupCounter = new AtomicLong(0);

    private volatile int keyCount = 0;

    public Poller() throws IOException {
        this.selector = Selector.open();
    }
    public void register(final NioChannel socket) {
        socket.setPoller(this);
        NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
        socket.setSocketWrapper(ka);
        ka.setPoller(this);
        ka.setReadTimeout(getConnectionTimeout());
        ka.setWriteTimeout(getConnectionTimeout());
        ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
        ka.setSecure(isSSLEnabled());
        PollerEvent r = eventCache.pop();
        ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
        if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
        else r.reset(socket,ka,OP_REGISTER);
        addEvent(r);
    }
    public void run() {
        // Loop until destroy() is called
        while (true) {

            boolean hasEvents = false;

            try {
                if (!close) {
                    hasEvents = events();
                    if (wakeupCounter.getAndSet(-1) > 0) {
                        //if we are here, means we have other stuff to do
                        //do a non blocking select
                        keyCount = selector.selectNow();
                    } else {
                        keyCount = selector.select(selectorTimeout);
                    }
                    wakeupCounter.set(0);
                }
                if (close) {
                    events();
                    timeout(0, false);
                    try {
                        selector.close();
                    } catch (IOException ioe) {
                        log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                    }
                    break;
                }
            } catch (Throwable x) {
                ExceptionUtils.handleThrowable(x);
                log.error(sm.getString("endpoint.nio.selectorLoopError"), x);
                continue;
            }
            //either we timed out or we woke up, process events first
            if ( keyCount == 0 ) hasEvents = (hasEvents | events());

            Iterator<SelectionKey> iterator =
                keyCount > 0 ? selector.selectedKeys().iterator() : null;
            //遍历执行就绪事件
            while (iterator != null && iterator.hasNext()) {
                SelectionKey sk = iterator.next();
                NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
                // Attachment may be null if another thread has called
                // cancelledKey()
                if (attachment == null) {
                    iterator.remove();
                } else {
                    iterator.remove();
                    processKey(sk, attachment);
                }
            }

            //process timeouts
            timeout(keyCount,hasEvents);
        }//while

        getStopLatch().countDown();
    }
}
```




## Spring Boot配置


```properties
server. Port = xxxx
server. Address =
server. contextPath =
server. displayName =
server. servletPath =
server. contextParameters =
server. useForwardHeaders =
server. serverHeader =
server. maxHttpHeaderSize =
server. maxHttpPostSize =
server. connectionTimeout =
server. session.timeout =
server. session.trackingModes =
server. session.persistent =
server.session.storeDir =
server.cookie. name =
server.cookie. domain =
server.cookie. path =
server.cookie. comment =
server.cookie. httpOnly =
server.cookie. secure =
server.cookie. maxAge =
server. ssl. Enabled =
server.ssl. clientAuth =
server.ssl. ciphers =
server.ssl. enabledProtocols =
server.ssl. keyAlias =
server.ssl. keyPassword =
server.ssl. keyStore =
server.ssl. keyStorePassword =
server.ssl. keyStoreType =
server.ssl. keyStoreProvider =
server.ssl. trustStore =
server.ssl. trustStorePassword =
server.ssl. trustStoreType =
server.ssl. trustStoreProvider =
server.ssl. protocol =
server.compression. enabled =
server.compression.mimeTypes =
server.compression.excludedUserAgents =
server.compression.minResponseSize =
server. jspServlet. className =
server.jspServlet. initParameters =
server.jspServlet.registered =
server.tomcat.accesslog.enabled =
server.tomcat.accesslog.pattern =
server.tomcat.accesslog.directory =
server.tomcat.accesslog.prefix =
server.tomcat.accesslog.suffix =
server.tomcat.accesslog.rotate =
server.tomcat.accesslog.renameOnRotate =
server.tomcat.accesslog.requestAttributesEnabled=
server.tomcat.accesslog.buffered =
server.tomcat.internalProxies =
server.tomcat.protocolHeader =
server.tomcat.protocolHeaderHttpsValue =
server.tomcat.portHeader =
server.tomcat.remoteIpHeader=
server.tomcat.basedir =
server.tomcat.backgroundProcessorDelay =
server.tomcat.maxThreads =
server.tomcat.minSpareThreads =
server.tomcat.maxHttpPostSize =
server.tomcat.maxHttpHeaderSize =
server.tomcat.redirectContextRoot =
server.tomcat.uriEncoding =
server.tomcat.maxConnections =
server.tomcat.acceptCount =
server.tomcat.additionalTldSkipPatterns =
```