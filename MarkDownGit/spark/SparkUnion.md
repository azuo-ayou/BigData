可以参考如下博客

https://blog.csdn.net/IAmListening/article/details/94617939



分两种情况

1、参与的所有RDD的分区规则相同（分区器相同）

会生成PartitionerAwareUnionRDD：这时候新的RDD分区规则也相同，对应分区的数据，是所有RDD对应index的集合，比如RDD1、RDD2、RDD3进行union操作，那么newRDD的分区1中，就是三个RDD分区1数据的集合



2、参与的所有RDD分区规则不相同

这时候，就会生成UnionRDD：新的RDD分区是上游分区数量的总和，也就是上游的RDD的每一个分区对应newRDD的一个分区



总结无论哪种方式，都不会产生shuffle；但是可能重新分区