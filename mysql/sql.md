# sql语法相关

### 1.MySQL七种join

<https://www.cnblogs.com/dinglinyong/p/6656315.html>

SELECT * FROM all_area a

left JOIN hy_area b

ON a.area_name=b.NAME

WHERE  b.id is NULL;



### 2.插入查询数据

**INSERT** **INTO** all_area_detail (id,pid,shortname,**NAME**,merger_name,`level`,pinyin,**CODE**,zip_code,lng,lat,area_code)

**SELECT** a.id,a.pid,a.shortname,a.**NAME**,a.merger_name,a.`level`,a.pinyin,a.**CODE**,a.zip_code,a.lng,a.lat,b.area_code

**FROM** hy_area a

**left** **JOIN** all_area b

**ON** a.**NAME**=b.area_name **AND** a.**CODE**=b.city_code;



### 3.一个字段更新为另外一个字段

**UPDATE** all_area **SET** short_name=**LEFT**(short_name,**char_LENGTH**(short_name)-1);

**UPDATE** all_area_detail a,all_area b

**SET** a.area_code=b.area_code

**WHERE** a.area_code **IS** **NULL** **AND** a.**NAME**=b.area_name;



### 4.MySQL解决不能同时update和select同一个表

<https://blog.csdn.net/younglao/article/details/76570187>

**UPDATE** all_area_detail **SET** area_code = **NULL**

**WHERE** id **IN**(

**SELECT** **x**.id **FROM**

(**SELECT** a.id **FROM** all_area_detail a

**LEFT** **JOIN** all_area b

**ON** a.area_code=b.area_code

**WHERE** a.area_code **IN**(

**SELECT** area_code **FROM** all_area_detail

**GROUP** **BY** area_code

**HAVING** **COUNT**(*)>1)

**AND** a.**CODE**!=b.city_code) **x**

);



### 5.**MySQL字符串按照逗号切分**

**SELECT** area_code,**SUBSTRING_INDEX**(center,',',1) lng,**SUBSTRING_INDEX**(center,',',1) lat

**FROM** all_area **LIMIT** 10;



### 6.**统计字符串某个字符出现的次数**

**select** enclosure_name 城市,**length**(lnglat_gd)-**length**(**replace**(lnglat_gd,';','')) 高德_num,

**length**(lnglat_wgs)-**length**(**replace**(lnglat_wgs,';','')) WGS84_num,

**length**(lnglat_bd)-**length**(**replace**(lnglat_bd,';','')) 百度_num

**FROM** vehicle_enclosure;



### 7.MySQL更新字段为同一个值时，不更新

If you set a column to the value it currently has, MySQL notices this and does not update it.

<http://dev.mysql.com/doc/refman/5.6/en/update.html>



### 8.数据库索引为什么使用B+树而不是hash或者平衡二叉树？                

1、hash表只能匹配是否相等，不能实现范围查找

2、当需要按照索引进行order by时，hash值没办法支持排序

3、组合索引可以支持部分索引查询，如(a,b,c)的组合索引，查询中只用到了阿和b也可以查询的，如果使用hash表，组合索引会将几个字段合并hash，没办法支持部分索引
 4、当数据量很大时，hash冲突的概率也会非常大
 5、B+树作为索引时，非叶子节点只保存索引，叶子节点才会保存数据，这样方便扫库，只需要扫一遍叶子结点即可，但是B树因为其分支结点同样存储着数据，我们要找到具体的数据，需要进行一次中序遍历按序来扫，所以B+树更加适合在区间查询的情况，所以通常B+树用于数据库索引。

6.b树其实就是平衡多叉排序树，如果都是在内存中操作，平衡二叉排序树效率甚至略高于b树，但是因为db的索引不可能全部存入内存，平衡多叉树的高度低于平衡二叉树，所有**磁盘的寻址加载次数**会更少，在读取相同磁盘块的同时，尽可能多的加载索引数据，来提高索引命中效率，从而达到减少磁盘IO的读取次数。

#### 为什么存在磁盘块？

读取方便：由于扇区的数量比较小，数目众多在寻址时比较困难，所以操作系统就将相邻的扇区组合在一起，形成一个块，再对块进行整体的操作。

分离对底层的依赖：操作系统忽略对底层物理存储结构的设计。通过虚拟出来磁盘块的概念，在系统中认为块是最小的单位。

1. 扇区： 硬盘的最小读写单元
2. 块/簇： 是操作系统针对硬盘读写的最小单元
3. page： 是内存与操作系统之间操作的最小单元。

[硬盘基本知识（磁头、磁道、扇区、柱面）](https://www.jianshu.com/p/9aa66f634ed6)



### 9.MySQL5.7子查询优化

[MySQL5.7性能优化系列（二）——SQL语句优化（2）——使用 Semi-Join半连接变换优化子查询，派生表和视图](https://blog.csdn.net/t131452n/article/details/76697777)

- 它必须是没有UNION结构的单个SELECT。
- 它不能包含GROUP BY或HAVING子句。
- 它不能被隐式分组（它不能包含聚合函数）
- 它不能有限制的ORDER BY。
- 语句不能在外部查询中使用STRAIGHT_JOIN连接类型。
- STRAIGHT_JOIN修饰符不能存在。
- 外表和内表的数量必须小于连接中允许的最大表数。

子查询可能相关或不相关。允许使用DISTINCT，除非使用ORDER BY，否则为LIMIT。

如果子查询符合上述条件，MySQL将其转换为半连接，并从以下策略中进行基于成本的选择：

- 将子查询转换为连接，或使用表拉出，并将查询作为子查询表和外部表之间的内部连接运行。表拉出将子查询中的表从外部查询中拉出。
- 重复删除：运行半连接，就像它是一个连接，并使用临时表删除重复的记录。
- FirstMatch：扫描内部表格的行组合，并且有多个给定值组的实例时，选择一个，而不是全部返回。这种“快捷方式”扫描并消除了不必要的行的生成。
- LooseScan：使用启用从每个子查询的值组中选择单个值的索引来扫描子查询表。
- 将子查询实现为用于执行连接的索引临时表，其中索引用于删除重复项。当使用外部表格加入临时表时，索引也可能用于查找;如果没有，表被扫描。

可以使用以下optimizer_switch系统变量标志来启用或禁用这些策略中的每一个：

- semijoin标志控制是否使用半连接。
- 如果semijoin被启用，首先匹配，失去扫描，重复输出和实现标志可以对允许的半连接策略进行更好的控制。
- 如果禁用重复输入半连接策略，则除非所有其他适用的策略也被禁用，否则不会使用它。
- 如果重复输入被禁用，有时优化器可能生成远离最优的查询计划。这是由于在贪婪搜索期间启发式修剪，这可以通过设置optimizer_prune_level = 0来避免