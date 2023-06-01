参考：https://www.zhihu.com/people/wang-xi-76-46/posts

### 一、Checkpoint

获取分布式数据流和算子状态的一致性快照是Flink容错机制的核心，这些快照在Flink作业恢复时作为一致性检查点存在。

### 1.1 原理

**1.1.1 Barriers**
Barrier是由流数据源（stream source）注入数据流中，并作为数据流的一部分与数据记录一起往下游流动。Barriers将流里的记录分隔为一段一段的记录集，每一个记录集都对应一个快照。每个Barrier会携带一个快照的ID，这个快照对应Barrier前面的记录集。如下图所示。

当一个算子从所有输入流都接收到一个快照（n）的barrier时，它首先会生成该算子的状态快照，然后往该算子的所有下游广播一个barrier。这个算子是sink算子时，它会告知检查点的 coordinator（一般是flink的jobManager）,当所有sink算子都告知接收到了一个快照的barrier时，该快照生成结束。



**1.1.2 对齐检查点（aligned Checkpoint)**
当一个算子接收到多于一个输入流时，就需要进行这些流的barrier对齐。当一个算子接收到第一个输入流的快照barrier n时，它不能继续处理该流的其他数据，而是需要等待接收到最后一个流的barrier n，才可以生成算子的状态快照和发送挂起的输出记录，然后发送快照barrier n。否则，检查点快照n跟检查点n+1就会混淆。



检查点对齐保证了状态的准确性，但由于对齐操作是阻塞式的，会使检查点生成时长不可控，降低吞吐量，当作业出现反压时，会加剧反压，甚至导致作业不稳定等问题。



**1.1.3 非对齐检查点（Unaligned Checkpoint)**
为规避上述风险，从Flink 1.11开始，检查点也可以是非对齐的。具体方法比较类似于[Chandy-Lamport算法](http://link.zhihu.com/?target=https%3A//www.microsoft.com/en-us/research/uploads/prod/2016/12/Determining-Global-States-of-a-Distributed-System.pdf)，但Flink仍会在数据源插入barrier来避免检查点coordinator负载过重。

具体处理过程是这样的：算子在接收到第一个数据流的barrier n时就立即开始生成快照，将该barrier发往下游，将其他流中后续到来的该快照的记录进行异步存储，并创建各自的状态快照。

非对齐检查点可以保证barrier尽快到达sink, 非常适合算子拓扑中至少有一条缓慢路径的场景。然而，由于会增大I/O压力，如果写入状态后端是处理瓶颈的话，使用非对齐检查点就不太合适了。

### 二、Savepoint

savepoint是使用检查点机制创建的，作业执行状态的全局镜像，可用于flink的停止与恢复，升级等。savepoint有两部分构成：一是在稳定存储（如：HDFS、S3等）中保存了二进制文件的目录，二是元数据文件。这些文件表示了作业执行状态的镜像，其中元数据文件主要保存了以绝对路径表示的指针。



### 1.2 配置算子ID

使用savepoint进行恢复时，是根据算子ID来[匹配算子状态](http://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttps%3A%2F%2Fci.apache.org%2Fprojects%2Fflink%2Fflink-docs-stable%2Fops%2Fupgrading.html%23matching-operator-state)在savepoint中的存储位置的。官方文档强烈建议给每个算子手动配置一个算子ID，这个ID可以通过uid(String)方法配置。

当没有手动配置时，程序会根据算子在的程序算子拓扑中的位置生成一个ID。如果程序没有改变，是可以从savepoint中恢复的，但如果程序改变了，同一个算子在程序中的位置也就改变了，相应的算子ID也会变化，就无法从之前的savepoint恢复了。



https://www.jianshu.com/p/9c587bd491fc





### 三、Checkpoint与Savepoint的异同

### 3.1 相同点

- 都用于作业的恢复
- 创建时都使用ckeckpoint机制，使用相同的一套代码和数据格式

### 3.2 不同点

- **设计目的不同：** checkpoint是作为Flink作业容错机制存在的，用于作业潜在失败时的恢复，savepoint是作为需要作业重启时（比如：Flink版本升级、作业拓扑改变，并行度修改，作业A/B测试等场景）保存状态并恢复的一种机制。
- **生命周期不同：** checkpoint的生命周期由flink来管理，flink负责checkpoint的创建、维护和释放，过程中没有与用户交互。savepoint就不同了，它是由用户来创建、维护和删除的，savepoint的是事先规划好的、手动备份并用于恢复。
- **具体实现不同：** checkpoint作为用于恢复，需要定期触发并保存状态的机制。实现上需要满足两点：1）创建时尽量轻量级，2）恢复时越快越好。savepoint在创建和恢复时比checkpoint更重一些，更偏重于便捷性及对作业前述改动的支持。







### 为什么对齐就是Exactly-Once，非对齐就是At-Least-Once？

https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/concepts/stateful-stream-processing/#checkpointing

官网上就是这么说的，但是我不太能理解，说是非对齐会继续处理数据，但是我理解这个行为是异步的行为，当第一个barrier到达的时候，就已经开始做checkpoint了，这时候和对齐有什么区别吗？对齐的checkpoint也是当所有的barrier到达以后，做异步的checkpoint



那我这里就考虑一个问题：（这样思考对于官网的wiki是可以解释的，这个猜想还是需要去看源码才行）

非对齐，当第一个barrier到达以后，其实还是会更新当前版本状态数据的；而对齐checkpoint，之后是不会更新当前版本状态的数据



9aefedaabfb3bf86f62d3fd40d3d3d7ae2e2f759
