## MIT 6.830 Lab2 SimpleDB Operators

### 简介

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在这个系列的课程中，我们会完成一个简单的**关系型数据库**，了解数据库的底层实现。

------

### 1.Filter and Join

+ Filter：该运算符只返回满足Predicate类的tuple，该tuple被指定为其构造函数的一部分。因此他过滤了所有不匹配的tuple。
+ Join：该运算符联结两个tuple，它们由构造函数制定。这只需要一个简单的嵌套循环连接。

### 2. Fiels and Tuples

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;tuples是数据库里基础的组成部分，它们由一系列Field物体构成，每个field构成一个tuple。Field是不同数据类型的实现接口。tuple由底层访问创建。tuple也有一种类型叫做 _tuple descriptor_，被TupleDesc类代表。这个对象由一系列Type构成，每个tuple里的field描述相应字段的类型。

### 3.Catalog

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;catalog由一个表格和模式的列表构成，支持添加新表、获取特定数据。与每个表关联的是一个TupleDesc对象，它允许操作者确定表中字段的类型和数量。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;全局的catalog是一个单例，通过Database.getCatalog()获取。

### 4.BufferPool

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;缓冲池负责将最近读过的page缓存起来。所有从不同文件读写page的操作都要经过缓冲池，它包含固定数字的page。

### 5.HeapFile access method

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;访问方法提供从硬盘读写数据的特定方法。一般的访问方法有heap files和b树。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HeapFIle对象被排列成一组page，每个都包含固定byte的tuples，包括header。在SimpleDB中，每个table都有一个HeapFile，每个HeapFile里的page被排列成一组槽，每个slot里有一个tuple。除了这些槽，每个page包含一个含有bitmap的header（每个比特来自一个tuple槽）。如果相应比特为1，则这个tuple可用，反之不行。page储存在缓冲池中，但是读写它们要通过HeapFile类。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SimpleDB存储的heap files或多或少有相同的格式，它们存储在内存中的磁盘。每个文件由磁盘上连续排列的页面数据组成。每个页面由一个或多个字节组成，代表header，接着是实际页面内容的页面大小字节。每个元组要求元组大小为其内容的8位，header的1位。因此，可以在单个页面中匹配的元组数是：

```
_tuples per page_ = floor((_page size_ * 8) / (_tuple size_ * 8 + 1))
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;tuple的大小是是这个page中的tuple的byte的大小。这里的想法是，每个tuple要求在header中存储额外的位。我们计算在一个页面中的比特数（page大小乘8），并将该数量的tuple中的比特数（包括额外的header）得到的每个page的uple数量。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每个字节的低（最少）位表示文件前面的槽的状态。因此，第一个字节的最低位表示页面中的第一个槽是否在使用中。第一个字节的第二个最低位表示页面中的第二个槽是否在使用，等等。另外，注意最后一个字节的高阶位可能与实际在文件中的插槽不对应，因为插槽的数目可能不是8的倍数。也注意到所有的java虚拟机的大小。



### 6.Operators

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;运算符负责查询计划的实际执行。它们实现关系代数的运算。在SimpleDB，运营商是基于迭代器的；每个经营者实施dbiterator接口。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;操作符通过将低级操作符传递到高级操作符的构造函数中，即通过将它们链接在一起，将它们连接到一个计划中。“计划”叶子的特殊访问方法运算符负责从磁盘读取数据（因此在它们下面没有任何操作符）。
在计划的顶部，与SimpleDB程序只需调用在根算子GETNEXT；这个操作员随后呼吁儿童GETNEXT，等等，直到这些叶者被称为。他们把元组从磁盘和山上的树（返回参数GETNEXT）；元组传播了这样的计划，直到他们在根或合并或被另一家运营商在计划输出。