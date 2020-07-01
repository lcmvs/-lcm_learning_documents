# PooledByteBufAllocator

[Netty源码-PooledByteBufAllocator静态变量初始化](https://www.jianshu.com/p/420809d36d5b)

[Netty源码-PoolThreadCache](https://www.jianshu.com/p/004aa3a8cd5f)

```java
public static final PooledByteBufAllocator DEFAULT =
        new PooledByteBufAllocator(PlatformDependent.directBufferPreferred());

private final PoolArena<byte[]>[] heapArenas;
private final PoolArena<ByteBuffer>[] directArenas;
private final int tinyCacheSize;
private final int smallCacheSize;
private final int normalCacheSize;
private final List<PoolArenaMetric> heapArenaMetrics;
private final List<PoolArenaMetric> directArenaMetrics;
private final PoolThreadLocalCache threadCache;
private final int chunkSize;
private final PooledByteBufAllocatorMetric metric;
```
## 简单使用

```java
ByteBufAllocator alloc = PooledByteBufAllocator.DEFAULT;

//tiny规格内存分配 会变成大于等于16的整数倍的数：这里254 会规格化为256
ByteBuf byteBuf = alloc.directBuffer(254);
```



# 默认值

```java
//默认的堆内内存Arena个数
private static final int DEFAULT_NUM_HEAP_ARENA;
//默认的堆外内存（直接内存）Arena个数
private static final int DEFAULT_NUM_DIRECT_ARENA;

//默认的Page（页）字节数 8192，即8k
private static final int DEFAULT_PAGE_SIZE;

//order决定了chunk的大小，chunk大小等于page大小<<order
// 8192 << 11 = 16 MiB per chunk
private static final int DEFAULT_MAX_ORDER; 

//最小的page字节数
private static final int MIN_PAGE_SIZE = 4096;
//每个chunk的最大字节数，这个最大值会约束page和order的大小
//默认等于1G
private static final int MAX_CHUNK_SIZE = (int) (((long) Integer.MAX_VALUE + 1) / 2);


//tiny 默认512
private static final int DEFAULT_TINY_CACHE_SIZE;、
//samll 默认256
private static final int DEFAULT_SMALL_CACHE_SIZE;
//normal 默认64
private static final int DEFAULT_NORMAL_CACHE_SIZE;
//缓存默认最大容量 32kb
private static final int DEFAULT_MAX_CACHED_BUFFER_CAPACITY;
//缓存清理阈值 8kb
private static final int DEFAULT_CACHE_TRIM_INTERVAL;
//是否对所有线程使用缓存 默认true
private static final boolean DEFAULT_USE_CACHE_FOR_ALL_THREADS;
//直接内存缓存内存对齐
private static final int DEFAULT_DIRECT_MEMORY_CACHE_ALIGNMENT;
```

## page

```java
//从配置参数io.netty.allocator.pageSize获取用户
//指定的page大小，如果没有配置则为8192，即8k
int defaultPageSize = SystemPropertyUtil.getInt("io.netty.allocator.pageSize", 8192);
Throwable pageSizeFallbackCause = null;
try {
    //这里主要检查指定的page大小要是2的指数方并且大于
    //上面介绍的page大小的最小值MIN_PAGE_SIZE,即最小4k
    validateAndCalculatePageShifts(defaultPageSize);
} catch (Throwable t) {
    //如果不满足上面的验证，则page大小为默认值8k
    pageSizeFallbackCause = t;
    defaultPageSize = 8192;
}
DEFAULT_PAGE_SIZE = defaultPageSize;


private static int validateAndCalculatePageShifts(int pageSize) {
    //page大小要小于MIN_PAGE_SIZE，4k
    if (pageSize < MIN_PAGE_SIZE) {
        throw new IllegalArgumentException("pageSize: " + pageSize + " (expected: " + MIN_PAGE_SIZE + ")");
    }

    //必须为2的指数
    if ((pageSize & pageSize - 1) != 0) {
        throw new IllegalArgumentException("pageSize: " + pageSize + " (expected: power of 2)");
    }

    //返回值没有用到
    // Logarithm base 2. At this point we know that pageSize is a power of two.
    return Integer.SIZE - 1 - Integer.numberOfLeadingZeros(pageSize);
}
```

## chunk

```java
//上面介绍了chunk大小等于page大小<<order
//这里则计算order大小
//首先从参数io.netty.allocator.maxOrder读取用户配置
//默认11
int defaultMaxOrder = SystemPropertyUtil.getInt("io.netty.allocator.maxOrder", 11);
Throwable maxOrderFallbackCause = null;
try {
    //这里传入上面用户指定或者默认的order，以及刚计算出的page大小
    //检验order小于14，并且page << order小于最大chunk大小
    //MAX_CHUNK_SIZE，1G
    validateAndCalculateChunkSize(DEFAULT_PAGE_SIZE, defaultMaxOrder);
} catch (Throwable t) {
    maxOrderFallbackCause = t;
    defaultMaxOrder = 11;
}
DEFAULT_MAX_ORDER = defaultMaxOrder;



private static int validateAndCalculateChunkSize(int pageSize, int maxOrder) {
    //order不能大于14
    if (maxOrder > 14) {
        throw new IllegalArgumentException("maxOrder: " + maxOrder + " (expected: 0-14)");
    }
    //保证page << order，不大于最大的chunk大小，MAX_CHUNK_SIZE
    // Ensure the resulting chunkSize does not overflow.
    int chunkSize = pageSize;
    for (int i = maxOrder; i > 0; i --) {
        if (chunkSize > MAX_CHUNK_SIZE / 2) {
            throw new IllegalArgumentException(String.format(
                    "pageSize (%d) << maxOrder (%d) must not exceed %d", pageSize, maxOrder, MAX_CHUNK_SIZE));
        }
        chunkSize <<= 1;
    }
    return chunkSize;
}
```

## arena数组

```java
//默认最小arena数量 = netty处理器个数*2 默认就是cpu核心数
final int defaultMinNumArena = NettyRuntime.availableProcessors() * 2;
//默认的chunk大小为page << order
final int defaultChunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER;


//从参数io.netty.allocator.numHeapArenas读取用户指定的堆arena个数
//如果没有指定，则取上面defaultMinNumArena和runtime.maxMemory() / defaultChunkSize / 2 / 3二者较小值
//除2是为了保证【堆内内存池】arena大小不能超过可用【堆内存】的一半
//除3是为了保证每个arena中至少有三个chunk
DEFAULT_NUM_HEAP_ARENA = Math.max(0,
        SystemPropertyUtil.getInt(
                "io.netty.allocator.numHeapArenas",
                (int) Math.min(
                        defaultMinNumArena,
                        runtime.maxMemory() / defaultChunkSize / 2 / 3)));

//和上面同理
//但是这里保证的【直接内存池】arena不超过配置的【最大直接内存】的一半
DEFAULT_NUM_DIRECT_ARENA = Math.max(0,
        SystemPropertyUtil.getInt(
                "io.netty.allocator.numDirectArenas",
                (int) Math.min(
                        defaultMinNumArena,
                        PlatformDependent.maxDirectMemory() / defaultChunkSize / 2 / 3)));

//如果按照page=8k, order=11，则chunk = 8k << 11 = 16M
//那么16 * 2 * 3 = 96M，所以默认情况下最少配置96M的堆内存或者直接内存才会启用内存池
```



## 缓存

```java
// cache sizes
DEFAULT_TINY_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.tinyCacheSize", 512);
DEFAULT_SMALL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.smallCacheSize", 256);
DEFAULT_NORMAL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.normalCacheSize", 64);

// 32 kb 缓存默认最大容量
DEFAULT_MAX_CACHED_BUFFER_CAPACITY = SystemPropertyUtil.getInt(
        "io.netty.allocator.maxCachedBufferCapacity", 32 * 1024);

// 空闲缓存释放阈值 默认8kb
DEFAULT_CACHE_TRIM_INTERVAL = SystemPropertyUtil.getInt(
        "io.netty.allocator.cacheTrimInterval", 8192);

// 释放对所有线程使用缓存 默认true
DEFAULT_USE_CACHE_FOR_ALL_THREADS = SystemPropertyUtil.getBoolean(
        "io.netty.allocator.useCacheForAllThreads", true);

//直接内存缓存对齐
DEFAULT_DIRECT_MEMORY_CACHE_ALIGNMENT = SystemPropertyUtil.getInt(
        "io.netty.allocator.directMemoryCacheAlignment", 0);
```

# 构造函数

```java
public static final PooledByteBufAllocator DEFAULT =
        new PooledByteBufAllocator(PlatformDependent.directBufferPreferred());
   
public PooledByteBufAllocator(boolean preferDirect) {
    this(preferDirect, DEFAULT_NUM_HEAP_ARENA, DEFAULT_NUM_DIRECT_ARENA, DEFAULT_PAGE_SIZE, DEFAULT_MAX_ORDER);
}

    
public PooledByteBufAllocator(boolean preferDirect, int nHeapArena, int nDirectArena, int pageSize, int maxOrder,
                              int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                              boolean useCacheForAllThreads, int directMemoryCacheAlignment) {
    super(preferDirect);
    threadCache = new PoolThreadLocalCache(useCacheForAllThreads);
    this.tinyCacheSize = tinyCacheSize;
    this.smallCacheSize = smallCacheSize;
    this.normalCacheSize = normalCacheSize;
    chunkSize = validateAndCalculateChunkSize(pageSize, maxOrder);

    if (nHeapArena < 0) {
        throw new IllegalArgumentException("nHeapArena: " + nHeapArena + " (expected: >= 0)");
    }
    if (nDirectArena < 0) {
        throw new IllegalArgumentException("nDirectArea: " + nDirectArena + " (expected: >= 0)");
    }

    if (directMemoryCacheAlignment < 0) {
        throw new IllegalArgumentException("directMemoryCacheAlignment: "
                                           + directMemoryCacheAlignment + " (expected: >= 0)");
    }
    if (directMemoryCacheAlignment > 0 && !isDirectMemoryCacheAlignmentSupported()) {
        throw new IllegalArgumentException("directMemoryCacheAlignment is not supported");
    }

    if ((directMemoryCacheAlignment & -directMemoryCacheAlignment) != directMemoryCacheAlignment) {
        throw new IllegalArgumentException("directMemoryCacheAlignment: "
                                           + directMemoryCacheAlignment + " (expected: power of two)");
    }

    int pageShifts = validateAndCalculatePageShifts(pageSize);

    if (nHeapArena > 0) {
        heapArenas = newArenaArray(nHeapArena);
        List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(heapArenas.length);
        for (int i = 0; i < heapArenas.length; i ++) {
            PoolArena.HeapArena arena = new PoolArena.HeapArena(this,
                                                                pageSize, maxOrder, pageShifts, chunkSize,
                                                                directMemoryCacheAlignment);
            heapArenas[i] = arena;
            metrics.add(arena);
        }
        heapArenaMetrics = Collections.unmodifiableList(metrics);
    } else {
        heapArenas = null;
        heapArenaMetrics = Collections.emptyList();
    }

    if (nDirectArena > 0) {
        directArenas = newArenaArray(nDirectArena);
        List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(directArenas.length);
        for (int i = 0; i < directArenas.length; i ++) {
            PoolArena.DirectArena arena = new PoolArena.DirectArena(
                this, pageSize, maxOrder, pageShifts, chunkSize, directMemoryCacheAlignment);
            directArenas[i] = arena;
            metrics.add(arena);
        }
        directArenaMetrics = Collections.unmodifiableList(metrics);
    } else {
        directArenas = null;
        directArenaMetrics = Collections.emptyList();
    }
    metric = new PooledByteBufAllocatorMetric(this);
}
```



# PoolThreadLocalCache

PoolThreadLocalCache是PooledByteBufAllocator的内部类

```java
final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {
    private final boolean useCacheForAllThreads;

    PoolThreadLocalCache(boolean useCacheForAllThreads) {
        this.useCacheForAllThreads = useCacheForAllThreads;
    }


}
```
## 初始化PoolThreadCache

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

# PoolThreadCache

```java
final class PoolThreadCache {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(PoolThreadCache.class);

    final PoolArena<byte[]> heapArena;
    final PoolArena<ByteBuffer> directArena;

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

    private int allocations;
}
```
## 构造函数

```java
PoolThreadCache(PoolArena<byte[]> heapArena, PoolArena<ByteBuffer> directArena, int tinyCacheSize, int smallCacheSize, int normalCacheSize, int maxCachedBufferCapacity, int freeSweepAllocationThreshold) {
    if (maxCachedBufferCapacity < 0) {
        throw new IllegalArgumentException("maxCachedBufferCapacity: " + maxCachedBufferCapacity + " (expected: >= 0)");
    }

    // 初始化参数
    this.heapArena = heapArena;
    this.directArena = directArena;
    this.freeSweepAllocationThreshold = freeSweepAllocationThreshold;
    if (directArena != null) {
        // 初始化缓存数组
        tinySubPageDirectCaches = createSubPageCaches(tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
        smallSubPageDirectCaches = createSubPageCaches(smallCacheSize, directArena.numSmallSubpagePools, SizeClass.Small);
        numShiftsNormalDirect = log2(directArena.pageSize);
        normalDirectCaches = createNormalCaches(normalCacheSize, maxCachedBufferCapacity, directArena);
        // 每次将Arena分配给PoolThreadCache，numThreadCaches都会加一
        directArena.numThreadCaches.getAndIncrement();
    } else {
        tinySubPageDirectCaches = null;
        smallSubPageDirectCaches = null;
        normalDirectCaches = null;
        numShiftsNormalDirect = -1;
    }

    if (heapArena != null) {
        // 初始化缓存数组
        tinySubPageHeapCaches = createSubPageCaches(tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
        smallSubPageHeapCaches = createSubPageCaches(smallCacheSize, heapArena.numSmallSubpagePools, SizeClass.Small);
        numShiftsNormalHeap = log2(heapArena.pageSize);
        normalHeapCaches = createNormalCaches(normalCacheSize, maxCachedBufferCapacity, heapArena);
        // 每次将Arena分配给PoolThreadCache，numThreadCaches都会加一
        heapArena.numThreadCaches.getAndIncrement();
    } else {
        tinySubPageHeapCaches = null;
        smallSubPageHeapCaches = null;
        normalHeapCaches = null;
        numShiftsNormalHeap = -1;
    }

    if (tinySubPageDirectCaches != null || smallSubPageDirectCaches != null || normalDirectCaches != null || tinySubPageHeapCaches != null || smallSubPageHeapCaches != null || normalHeapCaches != null) {
        // Only check freeSweepAllocationThreshold when there are caches in use.
        if (freeSweepAllocationThreshold < 1) {
            throw new IllegalArgumentException("freeSweepAllocationThreshold: " + freeSweepAllocationThreshold + " (expected: > 0)");
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


## MemoryRegionCache

这里，新引入一个数据类型`MemoryRegionCache`，其内部是一个`ByteBuf`队列。每个节点是一个`ByteBuf`的说法并不准确，切确的说，是不再使用的ByteBuf待释放的**内存空间**，可以再次使用这部分空间构建`ByteBuf`对象。根据分配请求大小的不同，`MemoryRegionCache`可以分为Tiny，Small，Normal三种。为了更方便的根据请求分配时的大小找到满足需求的缓存空间，每一种`MemoryRegionCache`又根据规范化后的大小依次组成数组，Tiny、Small、Normal的数组大小依次为32、4、12。

其中`ByteBuf`队列的长度是有限制的，Tiny、Small、Normal依次为512、256、64。为了更好的理解，举例子如下：

```css
16B  -- TinyCache[1]  -- (Buf512-...-Buf3-Buf2-Buf1)
32B  -- TinyCache[2]  -- ()
496B -- TinyCache[31] -- (Buf2-Buf1)
512B -- SmallCache[0] -- (Buf256-...-Buf3-Buf2-Buf1)
8KB  -- NormalCache[0] - (Buf64 -...-Buf3-Buf2-Buf1)
```

在线程缓存中，待回收的空间根据大小排列，比如，最大空间为16B的`ByteBuf`被缓存时，将被置于数组索引为1的`MemoryRegionCache`中，其中又由其中的队列存放该`ByteBuf`的空间信息，队列的最大长度为512。也就是说，16B的`ByteBuf`空间可以缓存512个，512B可以缓存256个，8KB可以缓存64个。

```java
    private final int size; // 队列长度
    private final Queue<Entry<T>> queue; // 队列
    private final SizeClass sizeClass; // Tiny/Small/Normal
    private int allocations; // 分配次数
```



## 回收

当一个`ByteBuf`不再使用，arena首先调用如下方法尝试缓存：

```java
    boolean add(PoolArena<?> area, PoolChunk chunk, long handle, 
                    int normCapacity, SizeClass sizeClass) {
        // 在缓存数组中找到符合的元素
        MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
        if (cache == null) {
            return false;
        }
        return cache.add(chunk, handle);
    }
    
    private MemoryRegionCache<?> cache(PoolArena<?> area, int normCapacity, SizeClass sizeClass) {
        switch (sizeClass) {
        case Normal:
            return cacheForNormal(area, normCapacity);
        case Small:
            return cacheForSmall(area, normCapacity);
        case Tiny:
            return cacheForTiny(area, normCapacity);
        default:
            throw new Error();
        }
    }
    
    private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
        // normCapacity >>> 4, 即16B的索引为1
        int idx = PoolArena.tinyIdx(normCapacity);  
        if (area.isDirect()) {
            return cache(tinySubPageDirectCaches, idx);
        }
        return cache(tinySubPageHeapCaches, idx);
    }

	//MemoryRegionCache
    public final boolean add(PoolChunk<T> chunk, long handle) {
        Entry<T> entry = newEntry(chunk, handle);
        boolean queued = queue.offer(entry);
        if (!queued) {
            // 队列已满，不缓存，立即回收entry对象进行下一次分配
            entry.recycle();
        }

        return queued;
    }
    
    private static Entry newEntry(PoolChunk<?> chunk, long handle) {
        Entry entry = RECYCLER.get(); // 从池中取出entry对象
        entry.chunk = chunk;
        entry.handle = handle;
        return entry;
    }
```

## 分配

以Tiny请求分配为例

```java
    boolean allocateTiny(PoolArena<?> area, PooledByteBuf<?> buf, 
                            int reqCapacity, int normCapacity) {
        return allocate(cacheForTiny(area, normCapacity), buf, reqCapacity);
    }
    
    private boolean allocate(MemoryRegionCache<?> cache, PooledByteBuf buf, int reqCapacity) {
        if (cache == null) {
            return false;   // 缓存数组中没有
        }
        boolean allocated = cache.allocate(buf, reqCapacity); // 实际的分配
        // 分配次数达到整理阈值
        if (++ allocations >= freeSweepAllocationThreshold) {
            allocations = 0;
            trim(); // 整理
        }
        return allocated;
    }

	//MemoryRegionCache
    public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
        Entry<T> entry = queue.poll();  // 从队列头部取出
        if (entry == null) {
            return false;
        }
        // 在之前ByteBuf同样的内存位置分配一个新的`ByteBuf`对象
        initBuf(entry.chunk, entry.handle, buf, reqCapacity);
        entry.recycle(); // entry对象回收利用

        ++ allocations; // 该值是内部量，和上一方法不同
        return true;
    }
    
    protected void initBuf(PoolChunk<T> chunk, long handle, PooledByteBuf<T> buf, 
                int reqCapacity) {
        chunk.initBufWithSubpage(buf, handle, reqCapacity);
    }
```

## 释放

```java
    void free() {
        if (freeTask != null) {
            assert deathWatchThread != null;
            ThreadDeathWatcher.unwatch(deathWatchThread, freeTask);
        }
        free0();
    }
    
    private void free0() {
        int numFreed = free(tinySubPageDirectCaches) + ... +
                free(normalHeapCaches);

        if (directArena != null) {
            directArena.numThreadCaches.getAndDecrement();
        }

        if (heapArena != null) {
            heapArena.numThreadCaches.getAndDecrement();
        }
    }

	//MemoryRegionCache
    public final int free() {
        return free(Integer.MAX_VALUE);
    }

    private int free(int max) {
        int numFreed = 0;
        for (; numFreed < max; numFreed++) {
            Entry<T> entry = queue.poll();
            if (entry != null) {
                freeEntry(entry);
            } else {
                return numFreed; // 队列中所有节点都被释放
            }
        }
        return numFreed;
    }
    
    private  void freeEntry(Entry entry) {
        PoolChunk chunk = entry.chunk;
        long handle = entry.handle;

        entry.recycle(); // 回收entry对象
        chunk.arena.freeChunk(chunk, handle, sizeClass); // 释放实际的内存空间
    }
```