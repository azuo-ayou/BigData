### 1、概念

在Flink中，水位线是一种衡量Event Time进展的机制，用来处理实时数据中的乱序问题的，通常是水位线和窗口结合使用来实现。

从设备生成实时流事件，到Flink的source，再到多个oparator处理数据，过程中会受到网络延迟、背压等多种因素影响造成数据乱序。在进行窗口处理时，不可能无限期的等待延迟数据到达，当到达特定watermark时,认为在watermark之前的数据已经全部达到(即使后面还有延迟的数据), 可以触发窗口计算，这个机制就是 Watermark(水位线)，具体如下图所示。

![img](https://pic4.zhimg.com/80/v2-4a143f217608461ff004902e80374e33_1440w.webp)

## 2、水位线的计算

watermark本质上是一个时间戳，且是动态变化的，会根据当前最大事件时间产生。watermarks具体计算为：

> watermark = 进入 Flink 窗口的最大的事件时间(maxEventTime)— 指定的延迟时间(t)

当watermark时间戳大于等于窗口结束时间时，意味着窗口结束，需要触发窗口计算。

![img](https://pic3.zhimg.com/80/v2-b96607e4851b941133afdc8d03b3822e_1440w.webp)

## 3、水位线生成

### 3.1 生成的时机

水位线生产的最佳位置是在尽可能靠近数据源的地方，因为水位线生成时会做出一些有关元素顺序相对时间戳的假设。由于数据源读取过程是并行的，一切引起Flink跨行数据流分区进行重新分发的操作（比如：改变并行度，keyby等）都会导致元素时间戳乱序。但是如果是某些初始化的filter、map等不会引起元素重新分发的操作，可以考虑在生成水位线之前使用。

### 3.2 水位线分配器

- **Periodic Watermarks**

周期性分配水位线比较常用，是我们会指示系统以固定的时间间隔发出的水位线。在设置时间为事件时间时，会默认设置这个时间间隔为200ms, 如果需要调整可以自行设置。比如下面的例子是手动设置每隔1s发出水位线。

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); 
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime); 
// 手动设置时间间隔为1s 
env.getConfig().setAutoWatermarkInterval(1000); 
```

周期水位线需要实现接口：AssignerWithPeriodicWatermarks，下面是示例：

```text
public class TestPeriodWatermark implements AssignerWithPeriodicWatermarks<Tuple2<String, Long>> { 
    Long currentMaxTimestamp = 0L; 
    final Long maxOutOfOrderness = 1000L;// 延迟时长是1s 

    @Nullable 
    @Override public Watermark getCurrentWatermark() { 
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness); 
    } 

    @Override 
    public long extractTimestamp(Tuple2<String, Long> element, long previousElementTimestamp) { 
        long timestamp = element.f1;         
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp); 
        return timestamp; 
    } 
} 
```

- **Punctuated Watermarks**

定点水位线不是太常用，主要为输入流中包含一些用于指示系统进度的特殊元组和标记，方便根据输入元素生成水位线的场景使用的。

由于数据流中每一个递增的EventTime都会产生一个Watermark。
在实际的生产中Punctuated方式在TPS很高的场景下会产生大量的Watermark在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择Punctuated的方式进行Watermark的生成。

```text
public class TestPunctuateWatermark implements AssignerWithPunctuatedWatermarks<Tuple2<String, Long>> { 
    @Nullable 
    @Override 
    public Watermark checkAndGetNextWatermark(Tuple2<String, Long> lastElement, long extractedTimestamp) { 
        return new Watermark(extractedTimestamp); 
    } 
    
    @Override 
    public long extractTimestamp(Tuple2<String, Long> element, long previousElementTimestamp) { 
        return element.f1; 
    } 
} 
```

### 4、水位线与数据完整性

水位线可以用于平衡延迟和结果的完整性，它控制着执行某些计算需要等待的时间。这个时间是预估的，现实中不存在完美的水位线，因为总会存在延迟的记录。现实处理中，需要我们足够了解从数据生成到数据源的整个过程，来估算延迟的上线，才能更好的设置水位线。

如果水位线设置的过于宽松，好处是计算时能保证近可能多的数据被收集到，但由于此时的水位线远落后于处理记录的时间戳，导致产生的数据结果延迟较大。

如果设置的水位线过于紧迫，数据结果的时效性当然会更好，但由于水位线大于部分记录的时间戳，数据的完整性就会打折扣。

所以，水位线的设置需要更多的去了解数据，并在数据时效性和完整性上有一个权衡。