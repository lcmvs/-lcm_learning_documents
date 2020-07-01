# ByteBuffer

## ByteBuffer vs ByteBuf

1.ByteBuffer长度固定，一旦分配无法动态扩展，ByteBuf可以进行自动扩容，write的时候如果容量不足，将期望最小容量转换成2的次方，将旧byte数组复制到新数组。

2.ByteBuffer只有一个标识位置的指针position，读写需要手工调用flip()和rewind()来切换读写模式，ByteBuf读写索引是分开的，readerIndex和writerIndex，所以不需要手动切换。

3.ByteBuf支持更多高级功能，比如池化。

# ByteBuf

引入缓冲区是为了解决速度不匹配的问题，在网络通讯中，CPU处理数据的速度大大快于网络传输数据的速度，所以引入缓冲区，将网络传输的数据放入缓冲区，累积足够的数据再送给CPU处理。

`ByteBuf`是一个可存储字节的缓冲区，其中的数据可提供给`ChannelHandler`处理或者将用户需要写入网络的数据存入其中，待时机成熟再实际写到网络中。由此可知，`ByteBuf`有读操作和写操作，为了便于用户使用，该缓冲区维护了两个索引：读索引和写索引。一个`ByteBuf`缓冲区示例如下：

```ruby
+-------------------+------------------+------------------+
| discardable bytes |  readable bytes  |  writable bytes  |
|                   |     (CONTENT)    |                  |
+-------------------+------------------+------------------+
|                   |                  |                  |
0      <=      readerIndex   <=   writerIndex    <=    capacity
```

可知，`ByteBuf`由三个片段构成：废弃段、可读段和可写段。其中，可读段表示缓冲区实际存储的可用数据。当用户使用`readXXX()`或者`skip()`方法时，将会增加读索引。读索引之前的数据将进入废弃段，表示该数据已被使用。此外，用户可主动使用`discardReadBytes（）`清空废弃段以便得到跟多的可写空间，示意图如下：

```ruby
清空前：
    +-------------------+------------------+------------------+
    | discardable bytes |  readable bytes  |  writable bytes  |
    +-------------------+------------------+------------------+
    |                   |                  |                  |
    0      <=      readerIndex   <=   writerIndex    <=    capacity
清空后：
    +------------------+--------------------------------------+
    |  readable bytes  |    writable bytes (got more space)   |
    +------------------+--------------------------------------+
    |                  |                                      |
readerIndex (0) <= writerIndex (decreased)       <=       capacity
```

对应可写段，用户可使用`writeXXX()`方法向缓冲区写入数据，也将增加写索引。

## 引用计数

[《Netty官方文档》引用计数对象](https://ifeve.com/reference-counted-objects/)

服务端的网络通讯应用在处理一个客户端的请求时，基本都需要创建一个缓冲区`ByteBuf`，直到将字节数据解码为POJO对象，该缓冲区才不再有用。由此可知，当面对大量客户端的并发请求时，如果能有效利用这些缓冲区而不是每次都创建，将大大提高服务端应用的性能。
 或许你会有疑问：既然已经有了JAVA的GC自动回收不再使用的对象，为什么还需要其他的回收技术？因为：1.GC回收或者引用队列回收效率不高，难以满足高性能的需求；2.缓冲区对象还需要尽可能的重用。有鉴于此，Netty4开始引入了引用计数的特性，缓冲区的生命周期可由引用计数管理，当缓冲区不再有用时，可快速返回给对象池或者分配器用于再次分配，从而大大提高性能，进而保证请求的实时处理。
 需要注意的是：引用计数并不专门服务于缓冲区`ByteBuf`。用户可根据实际需求，在其他对象之上实现引用计数接口`ReferenceCounted`。下面将详细介绍引用计数特性。

引用计数有如下两个基本规则：

- 对象的初始引用计数为1
- 引用计数为0的对象不能再被使用，只能被释放

在代码中，可以使用`retain()`使引用计数增加1，使用`release()`使引用计数减少1，这两个方法都可以指定参数表示引用计数的增加值和减少值。当我们使用引用计数为0的对象时，将抛出异常`IllegalReferenceCountException`。

### 谁负责释放对象

通用的原则是：谁最后使用含有引用计数的对象，谁负责释放或销毁该对象。一般来说，有以下两种情况：

1. 一个发送组件将引用计数对象传递给另一个接收组件时，发送组件无需负责释放对象，由接收组件决定是否释放。
2. 一个消费组件消费了引用计数对象并且明确知道不再有其他组件使用该对象，那么该消费组件应该释放引用计数对象。

```java
public ByteBuf a(ByteBuf input) {
    input.writeByte(42);
    return input;
}

public ByteBuf b(ByteBuf input) {
    try {
        output = input.alloc().buffer(input.readableBytes() + 1);
        output.writeBytes(input);
        output.writeByte(42);
        return output;
    } finally {
        input.release();
    }
}

public void c(ByteBuf input) {
    System.out.println(input);
    input.release();
}

public void main() {
    ByteBuf buf = Unpooled.buffer(1);
    c(b(a(buf)));
}
```

其中`main()`作为发送组件传递buf给`a()`，`a()`也仅仅写入数据然后发送给`b()`，`b()`同时作为消费者和发送者，消费input同时生成output发送给`c()`，`c()`仅仅作为消费者，不再产生新的引用计数对象。所以，`a()`不负责释放对象；`b()`完全消费了input，所以需要释放input，生成的output发送给`c()`，所以不负责释放output；`c()`完全消费了`b()`的output，故需要释放。



### 派生缓冲区

我们已经知道通过`duplicate()`，`slice()`等等生成的派生缓冲区`ByteBuf`会共享原生缓冲区的内部存储区域。此外，派生缓冲区并没有自己独立的引用计数而需要共享原生缓冲区的引用计数。也就是说，当我们需要将派生缓冲区传入下一个组件时，一定要注意先调用`retain()`方法。Netty的编解码处理器中，正是使用了这样的方法，可认为是下面代码的变形：

```java
ByteBuf parent = ctx.alloc().directBuffer(512);
parent.writeBytes(...);

try {
    while (parent.isReadable(16)) {
        ByteBuf derived = parent.readSlice(16);
        derived.retain();   // 一定要先增加引用计数
        process(derived);   // 传递给下一个组件
    }
} finally {
    parent.release();   // 原生缓冲区释放
}
...

public void process(ByteBuf buf) {
    ...
    buf.release();  // 派生缓冲区释放
} 
```

另外，实现`ByteBufHolder`接口的对象与派生缓冲区有类似的地方：共享所Hold缓冲区的引用计数，所以要注意对象的释放。在Netty，这样的对象包括`DatagramPacket`，`HttpContent`和`WebSocketframe`。

### 缓冲区泄露检测

没有什么东西是十全十美的，引用计数也不例外，虽然它大大提高了`ByteBuf`的使用效率，但也引入了一个新的问题：引用计数对象的内存泄露。由于JVM并没有意识到Netty实现的引用计数对象，它仍会将这些引用计数对象当做常规对象处理，也就意味着，当不为0的引用计数对象变得不可达时仍然会被GC自动回收。一旦被GC回收，引用计数对象将不再返回给创建它的对象池，这样便会造成内存泄露。
 为了便于用户发现内存泄露，Netty提供了相应的检测机制并定义了四个检测级别：

1.  `DISABLED` 完全关闭内存泄露检测，并不建议
2.  `SIMPLE` 以1%的抽样率检测是否泄露，默认级别
3.  `ADVANCED` 抽样率同`SIMPLE`，但显示详细的泄露报告
4.  `PARANOID` 抽样率为100%，显示报告信息同`ADVANCED` 

有以下两种方法，可以更改泄露检测的级别：

1.使用JVM参数`-Dio.netty.leakDetectionLevel`：

```java
    java -Dio.netty.leakDetectionLevel=advanced ...
```

2.直接使用代码

```java
    ResourceLeakDetector.setLevel(ResourceLeakDetector.Level.ADVANCED);
```

#### 最佳实践

- 单元测试和集成测试使用`PARANOID`级别
- 使用`SIMPLE`级别运行足够长的时间以确定没有内存泄露，然后再将应用部署到集群
- 如果发现有内存泄露，调到`ADVANCED`级别以提供寻找泄露的线索信息
- 不要将有内存泄露的应用部署到整个集群

此外，如果在单元测试中创建一个缓冲区，很容易忘了释放。这会产生一个
 内存泄露的警告，但并不意味着应用中有内存泄露。为了减少在单元测试代码中充斥大量的`try-finally`代码块用于释放缓冲区，Netty提供了一个通用方法`ReferenceCountUtil.releaseLater()`，当测试线程结束时，将会自动释放缓冲区，使用示例如下：

```java
import static io.netty.util.ReferenceCountUtil.*;

@Test
public void testSomething() throws Exception {
    ByteBuf buf = releaseLater(Unpooled.directBuffer(512));
    ...
}
```





# PooledByteBuf

```java
ByteBufAllocator alloc = PooledByteBufAllocator.DEFAULT;

//tiny规格内存分配 会变成大于等于16的整数倍的数：这里254 会规格化为256
ByteBuf byteBuf = alloc.directBuffer(254);
byteBuf.release();

ByteBuf byteBuf2 = alloc.directBuffer(254);
byteBuf2.release();
```

PooledByteBuf一般使用PooledByteBufAllocator来构造，构造一般分为两部分，一部分是构造一个PooledByteBuf对象，另外一部分则是为这个对象分配内存，因为是池化，所以这里涉及到对象池和内存池。

```java
private final PoolThreadLocalCache threadCache;
final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {}
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    
    //线程本地变量中获取缓存
    PoolThreadCache cache = threadCache.get();
    //线程缓存中获取相关的内存池
    PoolArena<ByteBuffer> directArena = cache.directArena;

    final ByteBuf buf;
    if (directArena != null) {
        //构造PooledByteBuf
        buf = directArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        buf = PlatformDependent.hasUnsafe() ?
                UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }

    return toLeakAwareBuffer(buf);
}

PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
    //对象池中获取对象
    PooledByteBuf<T> buf = newByteBuf(maxCapacity);
    //内存池中分配内存
    allocate(cache, buf, reqCapacity);
    return buf;
}
```

## 对象池

```java
final class PooledUnsafeDirectByteBuf extends PooledByteBuf<ByteBuffer> {
    
    private static final Recycler<PooledUnsafeDirectByteBuf> RECYCLER = new Recycler<PooledUnsafeDirectByteBuf>() {
        @Override
        protected PooledUnsafeDirectByteBuf newObject(Handle<PooledUnsafeDirectByteBuf> handle) {
            return new PooledUnsafeDirectByteBuf(handle, 0);
        }
    };

    static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
        PooledUnsafeDirectByteBuf buf = RECYCLER.get();
        buf.reuse(maxCapacity);
        return buf;
    }
}
```



## 内存池

PoolChunkList 通过使用率进行流转

PoolChunk 伙伴分配算法

PoolSubpage 位图算法

```java
abstract class PoolArena<T> implements PoolArenaMetric {
    
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
    final int normCapacity = normalizeCapacity(reqCapacity);
    if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
        int tableIdx;
        PoolSubpage<T>[] table;
        boolean tiny = isTiny(normCapacity);
        if (tiny) { // < 512
            if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = tinyIdx(normCapacity);
            table = tinySubpagePools;
        } else {
            if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = smallIdx(normCapacity);
            table = smallSubpagePools;
        }

        final PoolSubpage<T> head = table[tableIdx];

        /**
         * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
         * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
         */
        synchronized (head) {
            final PoolSubpage<T> s = head.next;
            if (s != head) {
                assert s.doNotDestroy && s.elemSize == normCapacity;
                long handle = s.allocate();
                assert handle >= 0;
                s.chunk.initBufWithSubpage(buf, handle, reqCapacity);
                incTinySmallAllocation(tiny);
                return;
            }
        }
        synchronized (this) {
            allocateNormal(buf, reqCapacity, normCapacity);
        }

        incTinySmallAllocation(tiny);
        return;
    }
    if (normCapacity <= chunkSize) {
        if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
            // was able to allocate out of the cache so move on
            return;
        }
        synchronized (this) {
            allocateNormal(buf, reqCapacity, normCapacity);
            ++allocationsNormal;
        }
    } else {
        // Huge allocations are never served via the cache so just call allocateHuge
        allocateHuge(buf, reqCapacity);
    }
}
    
}
```



## PoolThreadLocalCache

```java
public void freeThreadLocalCache() {
    threadCache.remove();
}

final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {
    private final boolean useCacheForAllThreads;

    PoolThreadLocalCache(boolean useCacheForAllThreads) {
        this.useCacheForAllThreads = useCacheForAllThreads;
    }

    @Override
    protected void onRemoval(PoolThreadCache threadCache) {
        threadCache.free();
    }

    private <T> PoolArena<T> leastUsedArena(PoolArena<T>[] arenas) {
        if (arenas == null || arenas.length == 0) {
            return null;
        }

        PoolArena<T> minArena = arenas[0];
        for (int i = 1; i < arenas.length; i++) {
            PoolArena<T> arena = arenas[i];
            if (arena.numThreadCaches.get() < minArena.numThreadCaches.get()) {
                minArena = arena;
            }
        }

        return minArena;
    }
}
```

### 初始化

构造一个PoolThreadCache，分配一个最少使用的PoolArena

```java
@Override
protected synchronized PoolThreadCache initialValue() {
    final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
    final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);

    if (useCacheForAllThreads || Thread.currentThread() instanceof FastThreadLocalThread) {
        return new PoolThreadCache(
                heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);
    }
    // No caching for non FastThreadLocalThreads.
    return new PoolThreadCache(heapArena, directArena, 0, 0, 0, 0, 0);
}
```



## PoolThreadCache

### 成员变量

```java
    // 各类型的Cache数组
    private final MemoryRegionCache<byte[]>[] tinySubPageHeapCaches;
    private final MemoryRegionCache<byte[]>[] smallSubPageHeapCaches;
    private final MemoryRegionCache<byte[]>[] normalHeapCaches;
    private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;
    private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;
    private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;
    
    // 用于计算normal请求的数组索引 = log2(pageSize)
    private final int numShiftsNormalDirect;
    private final int numShiftsNormalHeap;
    
    private int allocations;    // 分配次数
    private final int freeSweepAllocationThreshold; // 分配次数到达该阈值则检测释放

    private final Thread deathWatchThread; // 线程结束观察者
    private final Runnable freeTask;
```

### 构造函数

```java
PoolThreadCache(PoolArena<byte[]> heapArena, PoolArena<ByteBuffer> directArena,
                int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                int maxCachedBufferCapacity, int freeSweepAllocationThreshold) {
    if (maxCachedBufferCapacity < 0) {
        throw new IllegalArgumentException("maxCachedBufferCapacity: "
                + maxCachedBufferCapacity + " (expected: >= 0)");
    }
    this.freeSweepAllocationThreshold = freeSweepAllocationThreshold;
    this.heapArena = heapArena;
    this.directArena = directArena;
    if (directArena != null) {
        tinySubPageDirectCaches = createSubPageCaches(
                tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
        smallSubPageDirectCaches = createSubPageCaches(
                smallCacheSize, directArena.numSmallSubpagePools, SizeClass.Small);

        numShiftsNormalDirect = log2(directArena.pageSize);
        normalDirectCaches = createNormalCaches(
                normalCacheSize, maxCachedBufferCapacity, directArena);

        directArena.numThreadCaches.getAndIncrement();
    } else {
        // No directArea is configured so just null out all caches
        tinySubPageDirectCaches = null;
        smallSubPageDirectCaches = null;
        normalDirectCaches = null;
        numShiftsNormalDirect = -1;
    }
    if (heapArena != null) {
        // Create the caches for the heap allocations
        tinySubPageHeapCaches = createSubPageCaches(
                tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
        smallSubPageHeapCaches = createSubPageCaches(
                smallCacheSize, heapArena.numSmallSubpagePools, SizeClass.Small);

        numShiftsNormalHeap = log2(heapArena.pageSize);
        normalHeapCaches = createNormalCaches(
                normalCacheSize, maxCachedBufferCapacity, heapArena);

        heapArena.numThreadCaches.getAndIncrement();
    } else {
        // No heapArea is configured so just null out all caches
        tinySubPageHeapCaches = null;
        smallSubPageHeapCaches = null;
        normalHeapCaches = null;
        numShiftsNormalHeap = -1;
    }

    // We only need to watch the thread when any cache is used.
    if (tinySubPageDirectCaches != null || smallSubPageDirectCaches != null || normalDirectCaches != null
            || tinySubPageHeapCaches != null || smallSubPageHeapCaches != null || normalHeapCaches != null) {
        // Only check freeSweepAllocationThreshold when there are caches in use.
        if (freeSweepAllocationThreshold < 1) {
            throw new IllegalArgumentException("freeSweepAllocationThreshold: "
                    + freeSweepAllocationThreshold + " (expected: > 0)");
        }
        freeTask = new Runnable() {
            @Override
            public void run() {
                free0();
            }
        };

        deathWatchThread = Thread.currentThread();

        // 观察线程每隔一个周期检测当前线程是否存活
        ThreadDeathWatcher.watch(deathWatchThread, freeTask);
    } else {
        freeTask = null;
        deathWatchThread = null;
    }
}
```