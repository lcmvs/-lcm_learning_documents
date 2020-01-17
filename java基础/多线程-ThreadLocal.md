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

Thread类中有ThreadLocalMap对象，但是默认是null的。

```java
//Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null;
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



### ThreadLocalMap

ThreadLocalMap使用了WeakReference弱引用，也就是说每当每当下次gc，如果线程执行完毕，该键值对就会被回收，不会因为ThreadLocalMap的引用，而影响ThreadLocal的gc。

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