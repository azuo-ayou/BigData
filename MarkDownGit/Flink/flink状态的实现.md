参考文献：https://mp.weixin.qq.com/s?__biz=MzkxOTE3MDU5MQ==&mid=2247484334&idx=1&sn=1aafab741bfd0e2e72652a4b459579c7&scene=21#wechat_redirect

万字长文讲解flink中状态的实现，flink状态实现的三个特点

1. 渐行式的rehash策略：不是一次扩容，当达到了扩容状态后，每次迁移至少四个元素，且每次必须完整迁移一个桶；查取数据的时候首先需要判断数据是在老的map还是新的map
2. 异步快照：数据的浅拷贝+副本操作；当需要进行快照时，那么就行浅拷贝数据，这时候还是引用的同一份数据，当数据需要被修改时就修改primaryTable的数据；当然也要保证尽可能多的数据被共享；包含一下场景