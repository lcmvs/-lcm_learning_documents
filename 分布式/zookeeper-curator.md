# Curator

[官方文档](https://curator.apache.org/curator-examples/index.html)

[官方doc](https://curator.apache.org/apidocs/index.html)

[Zookeeper客户端Curator使用详解](http://throwable.coding.me/2018/12/16/zookeeper-curator-usage/)

Curator是Netflix公司开源的一套zookeeper客户端框架，解决了很多Zookeeper客户端非常底层的细节开发工作，包括连接重连、反复注册Watcher和NodeExistsException异常等等。Patrixck Hunt（Zookeeper）以一句“Guava is to Java that Curator to Zookeeper”给Curator予高度评价。

- **curator-framework**：对zookeeper的底层api的一些封装。

- **curator-client**：提供一些客户端的操作，例如重试策略等。

- **curator-recipes**：封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式计数器、分布式Barrier等。

  ```xml
  <dependency>
      <groupId>org.apache.curator</groupId>
      <artifactId>curator-recipes</artifactId>
      <version>2.12.0</version>
  </dependency>
  ```



## 简单使用

```java
//重试机制
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
//获取连接
CuratorFramework client = CuratorFrameworkFactory.newClient(zookeeperConnectionString, retryPolicy);
client.start();

//创建节点
client.create().forPath("/my/path", myData);

//分布式锁InterProcessMutex
InterProcessMutex lock = new InterProcessMutex(client, lockPath);
if ( lock.acquire(maxWait, waitUnit) ) 
{
    try 
    {
        // do some work inside of the critical section here
    }
    finally
    {
        lock.release();
    }
};

//leader选举
LeaderSelectorListener listener = new LeaderSelectorListenerAdapter()
{
    public void takeLeadership(CuratorFramework client) throws Exception
    {
        // this callback will get called when you are the leader
        // do whatever leader work you need to and only exit
        // this method when you want to relinquish leadership
    }
}

LeaderSelector selector = new LeaderSelector(client, path, listener);
selector.autoRequeue();  // not required, but this is behavior that you will probably expect
selector.start();
```

## 分布式锁

InterProcessMutex类似于java中的ReentrantLock，可重入，可以重复使用，不需要每次创建新的实例。

具体实现：

向zookeeper添加临时顺序节点，未持有锁。

获取所有子节点列表，如果是最小节点，获取锁。

获取锁失败，向比自己小的节点注册watcher，等待wait。

[InterProcessMutex](https://curator.apache.org/curator-recipes/shared-reentrant-lock.html)

```java
public class InterProcessMutex implements InterProcessLock, Revocable<InterProcessMutex>
{
    private final LockInternals internals;
    private final String basePath;

    private final ConcurrentMap<Thread, LockData> threadData = Maps.newConcurrentMap();

    private static class LockData
    {
        final Thread owningThread;
        final String lockPath;
        final AtomicInteger lockCount = new AtomicInteger(1);

        private LockData(Thread owningThread, String lockPath)
        {
            this.owningThread = owningThread;
            this.lockPath = lockPath;
        }
    }

    private static final String LOCK_NAME = "lock-";
    
    private boolean internalLock(long time, TimeUnit unit) throws Exception
    {
        Thread currentThread = Thread.currentThread();
		//
        LockData lockData = threadData.get(currentThread);
        if ( lockData != null )
        {
            // 可重入性
            lockData.lockCount.incrementAndGet();
            return true;
        }

        String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
        if ( lockPath != null )
        {
            LockData newLockData = new LockData(currentThread, lockPath);
            threadData.put(currentThread, newLockData);
            return true;
        }

        return false;
    }
```

### 分布式队列



### 分布式计数器



### 分布式屏障



