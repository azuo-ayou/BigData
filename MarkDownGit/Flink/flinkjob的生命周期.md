flink job的生命周期中，可以做的事情，请看官网的这篇文章

https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/cli/

主要可以的操作接口包括

- Submitting a Job
- Job Monitoring
- Creating a Savepoint
- Terminating a Job
- Starting a Job from a Savepoint



其中flink任务结束可以包含两种情况

```java
//1、All sources are bound and they processed all the input records. The job will finish after all the input records are processed and all the result are committed to external systems.

//2、Users execute stop-with-savepoint [--drain]. The job will take a savepoint and then finish. With –-drain, the job will be stopped permanently and is also required to commit all the data. However, without --drain the job might be resumed from the savepoint later, thus not all data are required to be committed, as long as the state of the data could be recovered from the savepoint.
```

翻译：

1. 所有的流数据都被正常处理
2. 通过一个savepoint，结束任务：请看上述文章





之前的流程

- MAX_WATERMARK
- EndOfPartitionEvent

为了解决最后阶段的数据提交问题，引入了如下



- MAX_WATERMARK，用于触发时间时间、窗口等
- EndofData，用于说明数据已经结束，调用endInput()和finish()方法
- barrier，用于做最后的checkpoint
- EndOfPartitionEvent，任务结束调用close方法

