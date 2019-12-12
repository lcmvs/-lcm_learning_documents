# 位图算法和BitSet

## 位图算法

位图算法其实就是用一个bit位来映射一个数据，比如一个int类型的数据，需要4byte，而使用位图算法只需要一个位bit，只需要int的1/32。

用于处理大数据量的排序，判断某个数是否在在某个数组中，数组中是否有重复数据。

jdk实现了一个BitSet。

## java中的BitSet

BitSet是位操作的对象，值只有0或1即false和true，内部维护了一个long数组，初始只有一个long，所以BitSet最小的size是64，当随着存储的元素越来越多，BitSet内部会动态扩充，最终内部是由N个long来存储，这些针对操作都是透明的。

在没有外部同步的情况下，多个线程操作一个 `BitSet` 是不安全的。 

用1位来表示一个数据是否出现过，0为没有出现过，1表示出现过。使用用的时候既可根据某一个是否为0表示，此数是否出现过。

一个1G的空间，有 8*1024*1024*1024=8.58*10^9bit，也就是可以表示85亿个不同的数


|  void    | and(BitSet set)             对此目标位 set 和参数位 set 执行逻辑与操作。 |
| ---------- | ------------------------------------------------------------ |
|  void    | andNot(BitSet set)             清除此 BitSet 中所有的位，其相应的位在指定的 BitSet 中已设置。 |
|  int     | cardinality()             返回此 BitSet 中设置为 true 的位数。 |
|  void    | clear()             将此 BitSet 中的所有位设置为 false。 |
|  void    | clear(int bitIndex)             将索引指定处的位设置为 false。 |
|  void    | clear(int fromIndex, int toIndex)             将指定的 fromIndex（包括）到指定的 toIndex（不包括）范围内的位设置为 false。 |
|  Object  | clone()             复制此 BitSet，生成一个与之相等的新 BitSet。 |
|  boolean | equals(Object obj)             将此对象与指定的对象进行比较。 |
|  void    | flip(int bitIndex)             将指定索引处的位设置为其当前值的补码。 |
|  void    | flip(int fromIndex, int toIndex)             将指定的 fromIndex（包括）到指定的 toIndex（不包括）范围内的每个位设置为其当前值的补码。 |
|  boolean | get(int bitIndex)             返回指定索引处的位值。   |
|  BitSet  | get(int fromIndex, int toIndex)             返回一个新的 BitSet，它由此 BitSet 中从 fromIndex（包括）到 toIndex（不包括）范围内的位组成。 |
|  int     | hashCode()             返回此位 set 的哈希码值。       |
|  boolean | intersects(BitSet set)             如果指定的 BitSet 中有设置为 true 的位，并且在此 BitSet 中也将其设置为 true，则返回 ture。 |
|  boolean | isEmpty()             如果此 BitSet 中没有包含任何设置为 true 的位，则返回 ture。 |
|  int     | length()             返回此 BitSet 的“逻辑大小”：BitSet 中最高设置位的索引加 1。 |
|  int     | nextClearBit(int fromIndex)             返回第一个设置为 false 的位的索引，这发生在指定的起始索引或之后的索引上。 |
|  int     | nextSetBit(int fromIndex)             返回第一个设置为 true 的位的索引，这发生在指定的起始索引或之后的索引上。 |
|  void    | or(BitSet set)             对此位 set 和位 set 参数执行逻辑或操作。 |
|  void    | set(int bitIndex)             将指定索引处的位设置为 true。 |
|  void    | set(int bitIndex, boolean value)             将指定索引处的位设置为指定的值。 |
|  void    | set(int fromIndex, int toIndex)             将指定的 fromIndex（包括）到指定的 toIndex（不包括）范围内的位设置为 true。 |
|  void    | set(int fromIndex, int toIndex, boolean value)             将指定的 fromIndex（包括）到指定的 toIndex（不包括）范围内的位设置为指定的值。 |
|  int     | size()             返回此 BitSet 表示位值时实际使用空间的位数。 |
|  String  | toString()             返回此位 set 的字符串表示形式。 |
|  void    | xor(BitSet set)             对此位 set 和位 set 参数执行逻辑异或操作。 |

## 示例：大数据排序

1.创建一个10亿的正整形数据

```java
public static void main(String[] args){
    Long start=System.currentTimeMillis();
    long i=1000000000L;
    BufferedWriter writer = null;
    try {
        writer=new BufferedWriter(new FileWriter("E:\\data\\data.txt"));
        Random random=new Random();
    for(int j=0;j<i;j++){
        writer.write(random.nextInt(Integer.MAX_VALUE)+"");
        writer.newLine();
    }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if(writer!=null){
            try {
                writer.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    System.out.println((System.currentTimeMillis()-start)/1000);
}
```

2.使用BitSet对大数据文件进行排序

```java
public class BitSortData {

    public static void main(String[] args) throws InterruptedException {
        System.out.println("最大可用内存(MB)："+Runtime.getRuntime().maxMemory()/(1024*1024));
        sort("E:\\data\\data.txt","E:\\data\\data2.txt");

        get("E:\\data\\data.txt",0,100);
        System.out.println("---------------------------------");
        get("E:\\data\\data2.txt",0,100);
    }

    static void sort(String from,String to){
        Long start=System.currentTimeMillis();
        BufferedReader reader = null;
        BufferedWriter writer = null;
        try {
            reader = new BufferedReader(new FileReader(from));
            writer = new BufferedWriter(new FileWriter(to));
            BitSet bitSet=new BitSet(100000000);
            String str = null;
            while ((str=reader.readLine())!=null){
                bitSet.set(Integer.parseInt(str));
            }
            for(int i=bitSet.nextSetBit(0);i>=0;i=bitSet.nextSetBit(i+1)){
                writer.write(i+"");
                writer.newLine();
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if(reader!=null){
                try {
                    reader.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if(writer!=null){
                try {
                    writer.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        System.out.println("排序时间(s)："+(System.currentTimeMillis()-start)/1000);
    }

    static void get(String path,int from,int to){
        BufferedReader reader=null;
        try {
            reader=new BufferedReader(new FileReader(path));
            String str=null;
            for(int i=from;i<to&&(str=reader.readLine())!=null;i++){
                System.out.println(str);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (reader!=null){
                try {
                    reader.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

    }

}
```