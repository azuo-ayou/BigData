

## 问题

写这次文档目的，为了纪念一次flink中遇到的bug，程序本来很正常，但是用了process方法，就报了如下错误

<font color=red>java.lang.IllegalArgumentException: Invalid timestamp: -9223372036854775808. Timestamp should always be non-negative or null.</font>

```scala
java.lang.IllegalArgumentException: Invalid timestamp: -9223372036854775808. Timestamp should always be non-negative or null.
```

时间戳异常，发生在sink写kafka的时候，但是我又全程没有操作过时间戳，那这是为什么呢？如果直接看问题，大家肯定连我想表达什么都不知道？所以我们先来了解下边知识吧

## 数据

flink streaming各个算子之间的数据流转，其实在底层是有封装的，本质上就是一个**StreamElement**，我们来看看这是个什么东西，其实就是一个抽象类，看不出来什么，但是他有四个实现，通过方法我们就能知道

```java
public abstract class StreamElement {
    public StreamElement() {
    }
}
```



他有四个实现：

### 1、StreamRecord

这个就是对我们真是数据流转的封装，有三个属性

1. value：我们需要的数据
2. timestamp：时间戳
3. hasTimestamp：是否有时间戳（而我刚才报错的就是指这里的时间戳）



### 2、StreamStatus

当我们的程序中有多条输入流时，由于watermark更新时会选择其中的最小值，如果有一条输入流没有数据了，水印也就不会更新，那么下游就无法触发计算，为了解决这个问题，引入了StreamStatus。

StreamStatus用来表示某一条数据流的状态：

StreamStatus.ACTIVE_STATUS 表示当前数据流处于活动状态

StreamStatus.IDLE_STATUS 表示当前输入流处于闲置状态

当一条输入流处理IDLE状态时，其不会再向下游发送数据和水印，下游在进行水印更新的时候，会忽略掉这条闲置流。这样就不会影响正常的数据处理逻辑。下游可以继续工作。

当闲置的数据流有了新的数据，其就会转变为ACTIVE_STATUS。因为水印是不能倒退的，所以只有这条流的watermark大于等于上次触发窗口计算的watermark时，才会参与本次的最小水印选择。


### 3、Watermark

watermark大家都比较熟悉了，在使用事件时间语义时，watermark起到推进时间进度，触发窗口计算的作用。

当某个watermark流动到某个operator时，表示小于watermark的数据都已经到达，可以触发窗口计算，触发定时器。也可以根据watermark处理迟到数据。如果出现watermark的值为Long.MAX_VALUE时，代表对应的数据源关闭了。


### 4、LatencyMarker

Flink为了监控数据处理延迟，在source端周期性插入LatencyMarker，其内部的markedTime属性表示了创建的时间。随着数据的流动，每当一个operator收到LatencyMarker时，就会用当前时间 - marker.getMarkedTime() 将结果报告给指标收集系统，然后把这个LatencyMarker向下游转发。

要注意使用LatencyMarker探测出的延迟并不是端到端延迟，因为其转发没有经过用户逻辑。LatencyMarker会直接emit。由于业务数据处理和LatencyMarker的处理是同步的，随着数据流的流动，其结果和端到端延迟近似。



## StreamRecord的生成

我们这里重点来看StreamRecord的生成，不同的算子是不一样的

#### map

参考：package org.apache.flink.streaming.api.operators.StreamMap

```java
public void processElement(StreamRecord<IN> element) throws Exception {
    this.output.collect(element.replace(((MapFunction)this.userFunction).map(element.getValue())));
}
```

上述我们知道，map算子生成的StreamRecord，只是通过StreamRecord的replace方法，将上一个StreamRecord的value替换了，所以时间戳是继承自上一个算子的StreamRecord 



### flatmap

参考：package org.apache.flink.streaming.api.operators.StreamFlatMap，主要关注open方法和processElement方法。

其中：open方法的collector中来设置时间戳，这个collector是一个TimestampedCollector，他设置时间戳是通过collector.setTimestamp，如下：如果上一个算子的StreamRecord有时间戳就继承，没有的话就擦除掉。

```java
public void processElement(StreamRecord<IN> element) throws Exception {
        this.collector.setTimestamp(element);
        ((FlatMapFunction)this.userFunction).flatMap(element.getValue(), this.collector);
    }

public void setTimestamp(StreamRecord<?> timestampBase) {
    if (timestampBase.hasTimestamp()) {
        this.reuse.setTimestamp(timestampBase.getTimestamp());
    } else {
        this.reuse.eraseTimestamp();
    }

}
```



## process

参考：package org.apache.flink.streaming.api.operators.ProcessOperator类的output方法：如下

他是new了一个StreamRecord，时间戳是获取上一个StreamRecord的时间戳，通过StreamRecord的.getTimestamp()获得，我们点进去方法看，发现是如果没有时间戳，就自动获取个 **-9223372036854775808L**



```java
// package org.apache.flink.streaming.api.operators.ProcessOperator
public <X> void output(OutputTag<X> outputTag, X value) {
    if (outputTag == null) {
        throw new IllegalArgumentException("OutputTag must not be null.");
    } else {
        ProcessOperator.this.output.collect(outputTag, new StreamRecord(value, this.element.getTimestamp()));
    }
}

//StreamRecord的 getTimestamp（）方法
public long getTimestamp() {
        return this.hasTimestamp ? this.timestamp : -9223372036854775808L;
    }

```





## 总结

到这里基本上是明白了上述报错了

map：直接替换上个算子生成StreamRecord的value，所以时间戳直接继承上一个StreamRecord

flatmap：上一个StreamRecord有时间戳就继承，没有擦除时间戳

process：上一个StreamRecord有时间戳就继承，没有就是-9223372036854775808L









龙哥，目前cpa这块，