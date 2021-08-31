# ES

倒排索引+前缀树变种（后缀压缩）

![img](https://img2020.cnblogs.com/blog/1816118/202102/1816118-20210220214810192-621202462.png)

Term Index 都是在内存中，查询时，先通过其快速定位到 Term Dictionary 对应的大致范围，然后再进行磁盘读取查找对应的 Term，这样就大大减少了磁盘 I/O 的次数。

#### 联合索引查询

了解了 ElasticSearch 的倒排索引后，我们再来看看其如何处理复杂的联合索引查询。比如上述书籍例子中，我们需要查询评分等于2.2并且作者名称叫 Tom的书籍。

理论上，我们只需要分别按照 Score 和 Author 字段的倒排索引进行查询，获取响应的 Posting List，再将其做交集合并即可。

而 ElasticSearch 则支持使用跳表 Skip List和 Bitset 的方式将数据集进行合并。

- 使用 Skip List 结构，同时遍历 Score 和 Author 查询出来的 Posting List，利用其 Skip List 结构，相互跳跃对比，得出合集。
- 使用 Bitset 结构，对 Score 和 Author 查询出来的 Posting List 的值计算出各自的 Bitset，然后进行 AND 操作。



#### 跳表合并策略

ElasticSearch 在存储 Posting List 数据时，就保存了对应的多级跳表结构响应的数据，这也体现了其空间换时间的基本思想。

比如，按照 Score 查出来的 Posting List 为[2,3,4,5,7,9,10,11]，按照 Author 查出来的结果为 [3,8,9,12,13]，则二者的跳表结构如下图所示。

![img](https://img2020.cnblogs.com/blog/1816118/202102/1816118-20210220214835481-1844363387.png)

具体合并过程则是先选最短的 posting list，也就是 Author 的结果集，从其最小的一个 id 开始，将其作为当前最大值。然后依次剩余 posting list 中查找大于或等于该值的位置。

比如上述结果集中，先去 Score 结果集中查找 3，找到后，就表明 3是二者的合集元素之一；然后再重新开启一轮，选取 Author 结果集中 3 的下一个值 8 ，去 Score 结果集查询 8，发现了大于等于 8 的最小的值是 9 ，所以不可能有共同的值 8，然后再去 Author 结果集查找 9 ，发现其大于等于 9 的最小值是 12，所以再去 Score 结果集中查找大于等于 12的值，发现并不存在；最终得出二者的合集就只有[3]。

在查询过程中，每个 posting list 都可以根据当前 id 通过 skip list 快速跳过不符合的 id 值，加速整个合并取交集的过程。



### **Frame of Reference**

在进行查询的时候经常会进行组合查询，比如查询同时包含`choice`和`the`的文档，那么就需要分别查出包含这两个单词的文档的id，然后取这两个id列表的**交集**；如果是查包含`choice`**或者**`the`的文档，那么就需要分别查出posting list然后取**并集**。为了能够高效的进行交集和并集的操作，posting list里面的id都是有序的。同时为了减小存储空间，所有的id都会进行**delta编码**（delta-encoding，我觉得可以翻译成**增量编码**）

比如现在有id列表`[73, 300, 302, 332, 343, 372]`，转化成每一个id相对于前一个id的增量值（第一个id的前一个id默认是0，增量就是它自己）列表是`[73, 227, 2, 30, 11, 29]`。在这个新的列表里面，所有的id都是小于255的，所以每个id只需要**一个字节**存储。

实际上ES会做的更加精细，它会把所有的文档分成很多个block，每个block正好包含256个文档，然后单独对每个文档进行增量编码，计算出存储这个block里面所有文档最多需要多少位来保存每个id，并且把这个位数作为头信息（header）放在每个block 的前面。这个技术叫Frame of Reference，我翻译成索引帧。

比如对上面的数据进行压缩（假设每个block只有3个文件而不是256），压缩过程如下

![img](https://pic4.zhimg.com/80/v2-43b5dca8a0d3ea9b4e4f958d25e0d693_720w.jpg)

在返回结果的时候，其实也并不需要把所有的数据直接解压然后一股脑全部返回，可以直接返回一个迭代器`iterator`，直接通过迭代器的`next`方法逐一取出压缩的id，这样也可以极大的节省计算和内存开销。

通过以上的方式可以极大的节省posting list的空间消耗，提高查询性能。不过ES为了提高filter过滤器查询的性能，还做了更多的工作，那就是缓存。



#### Bitset 合并策略

ElasticSearch除了使用 skipList 来进行数据磁盘读取时的合并操作外，还会将一些查询条件对应的结果集 posting list 进行内存缓存，也就是所谓的 Filter Cache，为了后续再次复用。

为了减少内存缓存所消耗的内存空间大小，ElasticSearch 没有使用单纯的数组和 bitset 来存储 posting list，而是使用要压缩效率更高的 Roaring Bitmap。

我们可以先来讲一下单纯数组或 bitset 数据结构为什么并不使用。比如如下一道较为常见的面试题目：

> 给定含有40亿个不重复的位于[0, 2^32 - 1]区间内的整数的集合，如何快速判定某个数是否在该集合内？

如果我们要使用 unsigned long 数组来存储它的话，也就需要消耗 40亿 * 32 位 = 160 Byte，大致是 16000 MB。

如果要使用位图 Bitset 来存储的话，即某个数位于原集合内，就将它对应的位图内的比特置为1，否则保持为0。这样只需要消耗 2 ^ 32 位 = 512 MB，***这可只有原来的 3.2 % 左右***。

但是，Bitset 也有其缺陷，也就是稀疏存储的问题，比如上述集合并不是 40亿，而是只有2，3个，那么 Bitset 中只有少数几位是1，其他位都是 0，但是它仍然占用了 512 MB。

而 RoaringBitmap 就是为了解决稀疏存储的问题。下图就是 RoaringBitmap 的基本原理示意图。

![img](https://img2020.cnblogs.com/blog/1816118/202102/1816118-20210220214855144-1872868114.png)

- ### **Roaring Bitmaps**

  ES会缓存频率比较高的filter查询，其中的原理也比较简单，即生成`(fitler, segment)`和id列表的映射，但是和倒排索引不同，我们只把常用的filter缓存下来而倒排索引是保存所有的，并且filter缓存应该足够快，不然直接查询不就可以了。ES直接把缓存的filter放到内存里面，映射的posting list放入磁盘中。

  ES在filter缓存使用的压缩方式和倒排索引的压缩方式并不相同，filter缓存使用了roaring bitmap的数据结构，在查询的时候相对于上面的Frame of Reference方式CPU消耗要小，查询效率更高，代价就是需要的存储空间（磁盘）更多。

  Roaring Bitmap是由int数组和bitmap这两个数据结构改良过的成果——int数组速度快但是空间消耗大，bitmap相对来说空间消耗小但是不管包含多少文档都需要12.5MB的空间，即使只有一个文件也要12.5MB的空间，这样实在不划算，所以权衡之后就有了下面的Roaring Bitmap。

  1. Roaring Bitmap首先会根据每个id的高16位分配id到对应的block里面，比如第一个block里面id应该都是在0到65535之间，第二个block的id在65536和131071之间
  2. 对于每一个block里面的数据，根据id数量分成两类

  - 如果数量**小于4096**，就是用short数组保存
  - 数量**大于等于4096**，就使用bitmap保存

  在每一个block里面，一个数字实际上只需要**2个字节**来保存就行了，因为高16位在这个block里面都是相同的，高16位就是block的id，block id和文档的id都用short保存。

  ![img](https://pic4.zhimg.com/80/v2-08cbaf3270bfe6486adfb1edb4a51c7f_720w.jpg)

  至于4096这个分界线，因为当数量小于4096的时候，如果用bitmap就需要8kB的空间，而使用2个字节的数组空间消耗就要少一点。比如只有2048个值，每个值2字节，一共只需要4kB就能保存，但是bitmap需要8kB。

ElasticSearch 就是使用 Roaring Bitmap 来缓存不同条件查询出来的 posting list，然后再进行与操作计算出最终结果集。