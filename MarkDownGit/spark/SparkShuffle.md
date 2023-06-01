### spark shuffle的演变

- hash shuffle，为了解决mr不必要的排序，每一个maptask都为每一个reduce产生了一个文件。同时这样产生的文件过多
- 改进的hash shuffle，同一个core上执行的maptask 可以写入同一个文件，这个可以减少文件数量，但是还是很多
- sort shuffle：
  - 每一个maptask产生一个数据文件和一个索引文件，但是会强制按分区id排序，还是有性能消耗，现在spark默认的也是这种shuffle
    - 不需要combine：by pass sort shuffle
    - 需要combine：默认常见的shuffle
- 唐森 sort shuffle（**Serialized Shuffle**）：将数据序列化存储在堆外内存，合并的时候可以直接对文件进行合并，（当然了只适用于不需要combine，比如group by）



ShuffleWrite的四种方式

- （BypassMergeSortShuffleWriter）map不需要合并、不需要排序：分区数量<200,直接为每个分区分配buffer进行shuffle
- （Serialized Shuffle）map不需要合并、不需要排序：分区数量>200,且小于，通过一个数组，且可以利用堆外内存，进行序列化shuffle
- （ArrayShuffle）map不需要合并、需要排序：利用数组，不够的时候扩容，然后排序溢写
- （HashMapShuffle）map需要合并、需要或者不需要排序：利用数组结构的hashmap



### Spark 任务提交

1. 客户端提交jar包到集群
2. 想 RM申请资源，启动AM（driver）
3. AM启动后，再次向RM申请一批资源，用于启动Excutor
4. Executor启动后，会反向注册给AM所在的节点的Driver
5. 在AM（driver）上完成SC的初始化，依赖的生成、stage划分，DAG的生成
6. 将task分发到每一个结点上
7. 任务执行完成，释放资源

https://blog.csdn.net/qq_41587198/article/details/100159205

### flink启动步骤

**StreamGraph**：是根据用户通过 Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。

**JobGraph**：StreamGraph经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。

**ExecutionGraph**：JobManager 根据 JobGraph 生成的分布式执行图，是调度层最核心的数据结构。

**物理执行图**：JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。
