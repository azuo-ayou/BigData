### 分布式一致性的理解

本质上讲，分布式一致性问题可以分为两种：[分布式事务](https://cloud.tencent.com/product/dtf?from=10680)和分布式主备。

**分布式事务：**其中分布式事务是指一个写操作请求，对应多个不同类型的写操作，比如跨银行转账，一定是一个银行账号存款减少，另一个银行账号存款增加。

**分布式主备：**而分布式主备则是指一个写操作请求，对应多个相同类型的写操作，比如ZooKeeper集群，在Leader机器上的写操作，最终是会在集群内的所有备份机器上执行同样的操作。



### ZAB（）

两阶段提交

《从Paxos到ZooKeeper分布式一致性原理与实践》这本书第4章在介绍崩溃恢复时提到，ZAB协议需要保证以下两条特性：

- ZAB协议需要确保那些已经在Leader服务器上commit的事务最终被所有服务器都commit
- ZAB协议需要确保丢弃那些只在Leader服务器上被提出的事务



问题：最大的事务id是指提交的id还是，未提交的最大id

- 一个新选举出来的Leader必须要先依次commit前一个Leader将会commit的所有Proposal，然后才能开始提交自身Proposal。
- 任何时刻，都不会有两台以上的服务器获得了超过半数机器支持。

为了实现第一个要求，新选出来的Leader需要确保它已经获得了超过半数的机器支持，然后它才能以Leader身份运行。而且，新的Leader的初始状态必须要保证前一任Leader提交的两类Proposal都被commit：一类是已经被commit的Proposal，还有一类就是已经被接受但还没被commit。

从这段话明确说了，新的Leader必须要保证：只要旧的Leader提交的Proposal被除了旧Leader自身以外的Follower接受（即便这些Follower还未收到commit消息，即便收到这条消息的机器数还不过半），这些Proposal都需要被新的Leader提交并最终被commit。也就是说，我们前面提到的场景一和场景二，对于Zab协议来说，最终都是会被新的Leader提交并最终被commit。



### ZAB和RAFT以及PAXOS核心区别



先看总结

1. ZAB未提交的最大值，来选举
2. RAFT已提交的最大值，来选举
3. 各个节点先通讯，确定哪些改提交，哪些不该提交，最终大家一致，都有可能是leader



- **Leader候选机器的差异**

ZAB是具有**最大ZXID编号（包括未commit的Proposal）**的机器才有资格成为新的Leader；而RAFT则是具有**最大已commit编号（不包括未commit的Proposal）**的机器才有资格成为新的Leader；PAXOS则是**任意机器**都可以成为新的Leader。由此可见，对于ZAB协议，新任Leader无需考虑自身还未被commit的Proposal是该被舍弃还是该被继续提交（因为全都需要被继续提交，积极策略）；对于RAFT协议，新的Leader只会关心自身机器上还未commit的Proposal，因而即使某一个Proposal已经被其它半数以上的机器确认，也可能会被覆盖（消极策略）；对于PAXOS，由于任意机器都可以成为新的Leader，因而新的Leader需要和其它Acceptor进行通信，以确认哪些Proposal该提交，哪些该舍弃（本质上新旧Leader相当于同时有多个Leader的场景，介于积极和消极之间）。

- **Quorum机制的作用范围差异**

对于ZAB和RAFT，Quorum机制只在正常的Leader任期（广播阶段）内有效，对于同步/读阶段，两者都是以自身的Proposal为准，而不会关注其它机器的情况，其中ZAB是因为自身拥有了最新的Proposal，所以不需要关心；对于Paxos协议，Quorum机制在读阶段和写阶段（相当于ZAB的广播阶段）都有效，因而在读阶段需要和其它Acceptor进行沟通，以确认哪些该提交，哪些该作废。

- **Leader数量的差异（不考虑极端情况）**

ZAB和RAFT都是单Leader，而PAXOS则支持多Leader（当然，工业应用中，绝大多数还是采用单Leader策略）。

- **新Leader处理旧Leader还未commit消息的差异**

《ZooKeeper Distributed Process COORDINATION》这本书说到，当新的Leader在恢复阶段，会尝试将本服务器上还未同步的消息同步给所有Follower，不过同步时使用的epoch将不再是旧Leader的epoch，而是新Leader的epoch，这意味着新的Leader需要修改自身的日志记录中的epoch值；相反，RAFT论文中说到，新的Leader不会直接将本服务器上旧term的未commit消息直接同步给Follower，而是通过将本服务器上新term的**新的消息**同步给Follower并最终commit的机制，来**间接提交**旧term的未commit的消息（Raft never commits log entries from previous terms by counting replicas. Only log entries from the leader’s current term are committed by counting replicas; once an entry from the current term has been committed in this way, then all prior entries are committed indirectly because of the Log Matching Property）。当然，RAFT不用新term提交旧消息的另一个原因是用旧的term便于进行日志追踪（reason about）。