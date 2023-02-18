## count distinct 和 group by

- count distinct 的reducer个数只有一个，也就是最后所有数据会在一个节点去重，可能是存在一个类似于hashmap的结构中，这样的话，数据差异很大，比如id等字段，可能会发生oom
- group by 后然后再count，这样会多出一个stage，但是最后一个stage只是将每个任务的数据sum一下即可

这个理解起来相对容易，就不细说啦？参考大佬https://zhuanlan.zhihu.com/p/410745825



## join的实现

建议先看一看hive关于join的官方文档：https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Joins

### join最后一张表的流式加载

1、hive的官方文档有这么一段话

```
In every map/reduce stage of the join, the last table in the sequence is streamed through the reducers where as the others are buffered. Therefore, it helps to reduce the memory needed in the reducer for buffering the rows for a particular value of the join key by organizing the tables such that the largest tables appear last in the sequence.
google 翻译
在连接的每个 map/reduce 阶段，序列中的最后一个表通过 reducer 流式传输，而其他表则被缓冲。 因此，通过组织表以使最大的表出现在序列中的最后，有助于减少reducer 中用于缓冲连接键的特定值的行所需的内存。
```

所以注意要小表放前，有助于节省内存

2、可以指定需要被流式加载的表，比如

```
SELECT /*+ STREAMTABLE(a) */ a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
```



### hiveSQL操作的注意点

1. 为什么要小表在join的左边？

2. **set hive.skewjoin.key = 10000**和**set hive.optimize.skewjoin=true**参数含义
   1. skewjoin.key是用于大表和大表join的有数据倾斜时的优化，只对skew key进行map join，join时候将skew key存在hdfs目录，普通key正常join，skew key用map join
   
   2. 我暂时理解，这两个参数是一起使用的
   3. 只有INNER JOIN才可以！🤔️ 如果数据倾斜的Key 出现在Join的最后一张表时 , 是不会触发Skew Join 的优化!
   
3. set hive.auto.convert.join=true 开启mapjoin



https://blog.csdn.net/hellojoy/article/details/82931827

skew join 原理 ：https://blog.csdn.net/CPP_MAYIBO/article/details/111350138

Skew join : https://issues.apache.org/jira/browse/HIVE-8406