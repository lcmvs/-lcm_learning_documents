# ThreadLocal

线程本地变量是说，每个线程都有同一个变量的独有拷贝。它们访问的虽然是同一个变量local，但每个线程都有自己的独立的值，这就是线程本地变量的含义。

## 基本使用

```java
public static void main(String[] args) {
    //不带初始值
    ThreadLocal<Integer> local = new ThreadLocal<>();
    System.out.println(local.get());//null
    local.set(100);
    System.out.println(local.get());//100
    
    //带初始值
    ThreadLocal<Integer> local2 = ThreadLocal.withInitial(()->100);
    System.out.println(local.get());//100
}
```





## 使用场景

### DateFormat/SimpleDateFormat

SimpleDateFormat是线程不安全的，想要保证线程安全，需要每个线程创建自己的SimpleDateFormat，但是每个方法都创建，可能导致一个线程里面创建了多个，浪费资源。

```java
public class DateFormatSafeUtil {
    static ThreadLocal<DateFormat> df = ThreadLocal.withInitial(()->new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static String date2String(Date date) {
        return df.get().format(date);
    }

    public static Date string2Date(String str) throws ParseException {
        return df.get().parse(str);
    }
}
```





### 与线程池的关系

因为线程池会复用线程，所有线程本地变量内的值，也会被线程池内的线程复用。

当使用SimpleDateFormat，这是一个优势，因为，线程池可以复用这些对象，不需要再额外创建了。

但是如果这些线程池的每个任务，都需要一个新的值，那就应该remove掉旧的值。

```java
public class Test {

    private static ThreadLocal<Integer> local= ThreadLocal.withInitial(()->new Integer(1));

    public static void main(String[] args){
        test1();
        test2();
    }

    private static void test1(){
        Executor executor=new ThreadPoolExecutor(1, 1,
                1L, TimeUnit.MINUTES,
                new LinkedBlockingDeque<>(100));

        for(int i=0;i<5;i++){
            executor.execute(() -> {
                Integer j = local.get();
                j=j<<1;
                local.set(j);
                System.out.println(Thread.currentThread().getName()+":"+j);
            });
        }
    }

    private static void test2(){
        Executor executor=new ThreadPoolExecutor(1, 1,
                1L, TimeUnit.MINUTES,
                new LinkedBlockingDeque<>(100));

        for(int i=0;i<5;i++){
            executor.execute(() -> {
                Integer j = local.get();
                j=j<<1;
                local.set(j);
                System.out.println(Thread.currentThread().getName()+":"+j);
                local.remove();
            });
        }
    }
}
```


```
pool-1-thread-1:2
pool-2-thread-1:2
pool-1-thread-1:4
pool-2-thread-1:2
pool-1-thread-1:8
pool-2-thread-1:2
pool-1-thread-1:16
pool-2-thread-1:2
pool-1-thread-1:32
pool-2-thread-1:2
```



## 源码分析

[ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html)

ThreadLocal主要通过ThreadLocalMap实现，每个Thread类中都有一个属性ThreadLocalMap，默认null，第一次添加键值对的时候，会new一个对象，key为ThreadLocal，value为线程本地变量值。

```java
public class Thread implements Runnable {
	ThreadLocal.ThreadLocalMap threadLocals = null;
}
```



### set

当ThreadLocal第一次向一个线程set值的时候，会创建一个ThreadLocalMap，放入线程内，然后放入一个键值对，key为ThreadLocal对象，value为值。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    // 线性探测
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 找到对应的entry
        if (k == key) {
            e.value = value;
            return;
        }
        // 替换失效的entry
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold) {
        rehash();
    }
}
```

#### 替换清理失效entry

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // 向前扫描，查找最前的一个无效slot
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len)) {
        if (e.get() == null) {
            slotToExpunge = i;
        }
    }

    // 向后遍历table
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 找到了key，将其与无效的slot交换
        if (k == key) {
            // 更新对应slot的value值
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            /*
             * 如果在整个扫描过程中（包括函数一开始的向前扫描与i之前的向后扫描）
             * 找到了之前的无效slot则以那个位置作为清理的起点，
             * 否则则以当前的i作为清理起点
             */
            if (slotToExpunge == staleSlot) {
                slotToExpunge = i;
            }
            // 从slotToExpunge开始做一次连续段的清理，再做一次启发式清理
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果当前的slot已经无效，并且向前扫描过程中没有无效slot，则更新slotToExpunge为当前位置
        if (k == null && slotToExpunge == staleSlot) {
            slotToExpunge = i;
        }
    }

    // 如果key在table中不存在，则在原地放一个即可
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 在探测过程中如果发现任何无效slot，则做一次清理（连续段清理+启发式清理）
    if (slotToExpunge != staleSlot) {
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }
}

/**
 * 启发式地清理slot,
 * i对应entry是非无效（指向的ThreadLocal没被回收，或者entry本身为空）
 * n是用于控制控制扫描次数的
 * 正常情况下如果log n次扫描没有发现无效slot，函数就结束了
 * 但是如果发现了无效的slot，将n置为table的长度len，做一次连续段的清理
 * 再从下一个空的slot开始继续扫描
 * 
 * 这个函数有两处地方会被调用，一处是插入的时候可能会被调用，另外个是在替换无效slot的时候可能会被调用，
 * 区别是前者传入的n为元素个数，后者为table的容量
 */
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        // i在任何情况下自己都不会是一个无效slot，所以从下一个开始判断
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            // 扩大扫描控制因子
            n = len;
            removed = true;
            // 清理一个连续段
            i = expungeStaleEntry(i);
        }
    } while ((n >>>= 1) != 0);
    return removed;
}



```

### 扩容

```java
private void rehash() {
    // 做一次全量清理
    expungeStaleEntries();

    /*
     * 因为做了一次清理，所以size很可能会变小。
     * ThreadLocalMap这里的实现是调低阈值来判断是否需要扩容，
     * threshold默认为len*2/3，所以这里的threshold - threshold / 4相当于len/2
     */
    if (size >= threshold - threshold / 4) {
        resize();
    }
}

/**
 * 扩容，因为需要保证table的容量len为2的幂，所以扩容即扩大2倍
 */
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; 
            } else {
                // 线性探测来存放Entry
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null) {
                    h = nextIndex(h, newLen);
                }
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

#### 全量清理

```
/*
 * 做一次全量清理
 */
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null) {
            /*
             * 个人觉得这里可以取返回值，如果大于j的话取了用，这样也是可行的。
             * 因为expungeStaleEntry执行过程中是把连续段内所有无效slot都清理了一遍了。
             */
            expungeStaleEntry(j);
        }
    }
}
```



### get

get的时候会去当前Thread中，获取ThreadLocalMap，如果map中有值则返回值，否则调用setInitialValue()。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

private Entry getEntry(ThreadLocal<?> key) {
    // 根据key这个ThreadLocal的ID来获取索引，也即哈希值
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 对应的entry存在且未失效且弱引用指向的ThreadLocal就是key，则命中返回
    if (e != null && e.get() == key) {
        return e;
    } else {
        // 因为用的是线性探测，所以往后找还是有可能能够找到目标Entry的。
        return getEntryAfterMiss(key, i, e);
    }
}

/*
 * 调用getEntry未直接命中的时候调用此方法
 */
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
   
    
    // 基于线性探测法不断向后探测直到遇到空entry。
    while (e != null) {
        ThreadLocal<?> k = e.get();
        // 找到目标
        if (k == key) {
            return e;
        }
        if (k == null) {
            // 该entry对应的ThreadLocal已经被回收，调用expungeStaleEntry来清理无效的entry
            expungeStaleEntry(i);
        } else {
            // 环形意义下往后面走
            i = nextIndex(i, len);
        }
        e = tab[i];
    }
    return null;
}
```

#### 清理

```java
/**
 * 这个函数是ThreadLocal中核心清理函数，它做的事情很简单：
 * 就是从staleSlot开始遍历，将无效（弱引用指向对象被回收）清理，即对应entry中的value置为null，将指向这个entry的table[i]置为null，直到扫到空entry。
 * 另外，在过程中还会对非空的entry作rehash。
 * 可以说这个函数的作用就是从staleSlot开始清理连续段中的slot（断开强引用，rehash slot等）
 */
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 因为entry对应的ThreadLocal已经被回收，value设为null，显式断开强引用
    tab[staleSlot].value = null;
    // 显式设置该entry为null，以便垃圾回收
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 清理对应ThreadLocal已经被回收的entry
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            /*
             * 对于还没有被回收的情况，需要做一次rehash。
             * 
             * 如果对应的ThreadLocal的ID对len取模出来的索引h不为当前位置i，
             * 则从h向后线性探测到第一个空的slot，把当前的entry给挪过去。
             */
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                
                /*
                 * 在原代码的这里有句注释值得一提，原注释如下：
                 *
                 * Unlike Knuth 6.4 Algorithm R, we must scan until
                 * null because multiple entries could have been stale.
                 *
                 * 这段话提及了Knuth高德纳的著作TAOCP（《计算机程序设计艺术》）的6.4章节（散列）
                 * 中的R算法。R算法描述了如何从使用线性探测的散列表中删除一个元素。
                 * R算法维护了一个上次删除元素的index，当在非空连续段中扫到某个entry的哈希值取模后的索引
                 * 还没有遍历到时，会将该entry挪到index那个位置，并更新当前位置为新的index，
                 * 继续向后扫描直到遇到空的entry。
                 *
                 * ThreadLocalMap因为使用了弱引用，所以其实每个slot的状态有三种也即
                 * 有效（value未回收），无效（value已回收），空（entry==null）。
                 * 正是因为ThreadLocalMap的entry有三种状态，所以不能完全套高德纳原书的R算法。
                 *
                 * 因为expungeStaleEntry函数在扫描过程中还会对无效slot清理将之转为空slot，
                 * 如果直接套用R算法，可能会出现具有相同哈希值的entry之间断开（中间有空entry）。
                 */
                while (tab[h] != null) {
                    h = nextIndex(h, len);
                }
                tab[h] = e;
            }
        }
    }
    // 返回staleSlot之后第一个空的slot索引
    return i;
}
```

### setInitialValue

当线程中的没有ThreadLocalMap或者map中没有对应的key的时候，会初始化值，默认为null。

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```

可以通过重写initialValue()来让ThreadLocal拥有特定的值，jdk1.8扩展可以使用withInitial，jdk1.7或者之前的需要自己手动扩展。

```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}

static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```



### remove

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```



## ThreadLocalMap

```java
static class ThreadLocalMap {
    
//初始容量，必须为2的幂
private static final int INITIAL_CAPACITY = 16;

//Entry表，大小必须为2的幂
private Entry[] table;

//表里entry的个数
private int size = 0;

//重新分配表大小的阈值，默认为0
private int threshold; 

//设置resize阈值以维持最坏2/3的装载因子
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}

//环形意义的下一个索引
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

//环形意义的上一个索引
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}

/**
 * 构造一个包含firstKey和firstValue的map。
 * ThreadLocalMap是惰性构造的，所以只有当至少要往里面放一个元素的时候才会构建它。
 */
ThreadLocalMap(java.lang.ThreadLocal<?> firstKey, Object firstValue) {
    // 初始化table数组
    table = new Entry[INITIAL_CAPACITY];
    // 用firstKey的threadLocalHashCode与初始大小16取模得到哈希值
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    // 初始化该节点
    table[i] = new Entry(firstKey, firstValue);
    // 设置节点表大小为1
    size = 1;
    // 设定扩容阈值
    setThreshold(INITIAL_CAPACITY);
}
}
```

### 线性探索法

#### 哈希函数

构造函数中的`int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);`这一行代码。

ThreadLocal类中有一个被final修饰的类型为int的threadLocalHashCode，它在该ThreadLocal被构造的时候就会生成，相当于一个ThreadLocal的ID。

可以看出，它是在上一个被构造出的ThreadLocal的ID/threadLocalHashCode的基础上加上一个魔数0x61c88647的。这个魔数的选取与斐波那契散列有关，0x61c88647对应的十进制为1640531527。斐波那契散列的乘数可以用(long)((1L << 31) * (Math.sqrt(5) - 1))可以得到2654435769，如果把这个值给转为带符号的int，则会得到-1640531527。换句话说`(1L << 32) - (long) ((1L << 31) * (Math.sqrt(5) - 1))`得到的结果就是1640531527也就是0x61c88647。通过理论与实践，当我们用0x61c88647作为魔数累加为每个ThreadLocal分配各自的ID也就是threadLocalHashCode再与2的幂取模，得到的结果分布很均匀。

```java
//ThreadLocal
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

### 弱引用

因为如果这里使用普通的key-value形式来定义存储结构，实质上就会造成节点的生命周期与线程强绑定，只要线程没有销毁，那么节点在GC分析中一直处于可达状态，没办法被回收，而程序本身也无法判断是否可以清理节点。弱引用是Java中四档引用的第三档，比软引用更加弱一些，如果一个对象没有强引用链可达，那么一般活不过下一次GC。当某个ThreadLocal已经没有强引用可达，则随着它被垃圾回收，在ThreadLocalMap里对应的Entry的键值会失效，这为ThreadLocalMap本身的垃圾清理提供了便利。

其实目的就是让ThreadLocalMap里面节点持有的引用不影响ThreadLocal的gc。

```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    
    private Entry[] table;
}
```

### 内存泄漏

关于ThreadLocal是否会引起内存泄漏也是一个比较有争议性的问题，其实就是要看对内存泄漏的准确定义是什么。
 认为ThreadLocal会引起内存泄漏的说法是因为如果一个ThreadLocal对象被回收了，我们往里面放的value对于**【当前线程->当前线程的threadLocals(ThreadLocal.ThreadLocalMap对象）->Entry数组->某个entry.value】**这样一条强引用链是可达的，因此value不会被回收。
 认为ThreadLocal不会引起内存泄漏的说法是因为ThreadLocal.ThreadLocalMap源码实现中自带一套自我清理的机制。

之所以有关于内存泄露的讨论是因为在有线程复用如线程池的场景中，一个线程的寿命很长，大对象长期不被回收影响系统运行效率与安全。如果线程不会复用，用完即销毁了也不会有ThreadLocal引发内存泄露的问题。《Effective  Java》一书中的第6条对这种内存泄露称为`unintentional object retention`(无意识的对象保留）。

当我们仔细读过ThreadLocalMap的源码，我们可以推断，如果在使用的ThreadLocal的过程中，显式地进行remove是个很好的编码习惯，这样是不会引起内存泄漏。
 那么如果没有显式地进行remove呢？只能说如果对应线程之后调用ThreadLocal的get和set方法都有**很高的概率**会顺便清理掉无效对象，断开value强引用，从而大对象被收集器回收。

但无论如何，我们应该考虑到何时调用ThreadLocal的remove方法。一个比较熟悉的场景就是对于一个请求一个线程的server如tomcat，在代码中对web api作一个切面，存放一些如用户名等用户信息，在连接点方法结束后，再显式调用remove。

#### 自动清理

get和set的时候会有很大记录触发自动清理。

#### 最佳实践

```java
public class Dynamicxx {
    
    private static final ThreadLocal<String> threadLocal = new ThreadLocal<>();

    public void dosomething(){
        try {
             threadLocal.set("name");
            // 其它业务逻辑
        } finally {
            threadLocal.remove();
        }
    }

}
```

### 父子线程共享变量

[InheritableThreadLocal使用详解](https://www.cnblogs.com/54chensongxia/p/12015443.html)

对于InheritableThreadLocal，本文不作过多介绍，只是简单略过。
 ThreadLocal本身是线程隔离的，InheritableThreadLocal提供了一种父子线程之间的数据共享机制。

它的具体实现是在Thread类中除了threadLocals外还有一个`inheritableThreadLocals`对象。

可以看一下其中的具体实现

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                // 这里的childValue方法在InheritableThreadLocal中默认实现为返回本身值，可以被重写
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

还是比较简单的，做的事情就是以父线程的`inheritableThreadLocalMap`为数据源，过滤出有效的entry，初始化到自己的`inheritableThreadLocalMap`中。其中childValue可以被重写。

需要注意的地方是`InheritableThreadLocal`只是在子线程创建的时候会去拷一份父线程的`inheritableThreadLocals`。如果父线程是在子线程创建后再set某个InheritableThreadLocal对象的值，对子线程是不可见的。