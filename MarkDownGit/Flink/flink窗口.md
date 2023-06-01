窗口是flink处理无限流的核心,窗口将流拆分为有限大小的“桶”，我们可以在这些桶上进行计算。

### 1、Keyed vs Non-Keyed Windows

根据上游数据是否为Keyed Stream类型(是否将数据按照某个指定的Key进行分区)，将窗口划分为Keyed Window和Non-Keyed Windows。两者的区别在于KeyStream调用相应的window()方法来指定window类型，数据会根据Key在不同的Task中并行计算，而Non-Keyed Stream需要调用WindowsAll()方法来指定window类型，所有的数据都会在一个Task进行计算，相当于没有并行。

### 1.1 Keyed Windows

```text
stream
       .keyBy(...)               <-  keyed versus non-keyed windows
       .window(...)              <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"
```

### 1.2 Non-Keyed Windows

```text
stream
       .windowAll(...)           <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"
```

### 2、窗口分配器

窗口分配器负责将一个事件分配给一个或多个窗口，内置窗口包括： 滚动窗口（Tumbling Windows）、滑动窗口（Sliding Windows）、会话窗口（Session Windows）、全局窗口（Global Windows），也可以通过继承WindowAssigner类来自定义窗口。

### 2.1 基于时间的窗口

Flink中所有的内置窗口（全局窗口除外）都有基于时间的实现，这个时间可以是事件时间（event time），也可以是处理时间（processing time）。其中，处理滚动窗口和滑动窗口的算子，在1.12版本之前使用timeWindow()，在1.12版本被标记为[废弃](https://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttps%3A%2F%2Fci.apache.org%2Fprojects%2Fflink%2Fflink-docs-stable%2Frelease-notes%2Fflink-1.12.html)，转而使用window()来作为窗口处理算子，这里只介绍最新版本的使用算子。

- **滚动时间窗口（Tumbling Time Windows）**
  滚动窗口将每一个事件分配给一个有特定大小的窗口，滚动窗口有固定大小，不会重叠。比如一个滚动窗口大小（window size）为5分钟。

![img](https://pic1.zhimg.com/80/v2-7ab47b77c4899224f225ead0879f1a4c_1440w.webp)

使用示例如下：

```text
DataStream<T> input = ...;
// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);
// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);
// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
```

由于Flink默认使用的时间基准是UTC±00:00时间，在中国需要使用UTC+08:00时间，所以最后一个示例中窗口大小为1天，时间偏移量就是8小时。

- **滑动窗口（Sliding Time Windows）**
  跟滚动窗口类似，滑动窗口也是将每一个事件分配给特定大小的窗口，且窗口有固定的大小，但它有一个窗口滑动的参数，标识一个窗口滑动的频率，或者说是每隔多久窗口滑动一次。比如一个窗口的大小为10秒钟，滑动频率为5秒。

![img](https://pic3.zhimg.com/80/v2-22da15ebe4f89a36a7910c295381b5de_1440w.webp)

使用示例如下：

```text
DataStream<T> input = ...;
// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);
// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);
// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
```

最后一个示例，中的Time.hours(-8)含义与滚动窗口一致。从滑动窗口的使用来看，滚动窗口其实是滑动窗口的一个特例，但窗口大小和滑动间隔相等的时候，滑动窗口就是一个滚动窗口。

- **会话窗口（Session Windows）**
  会话窗口按活动的会话对事件进行分组。与滑动窗口和滚动窗口相比，会话窗口没有固定的大小，也没有固定的起止时间，它是以一段时间没有接收到事件为窗口结束条件的。会话窗口分配器可以配置成一个固定的session gap，或者定义成一个session gap提取函数，在函数中定义一个不活跃的时长，一旦这个时长结束，当前会话就结束。

![img](https://pic3.zhimg.com/80/v2-325ec49d0d8167c8622e435e506c9252_1440w.webp)

使用示例如下：

```text
DataStream<T> input = ...;
// event-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
// event-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);
// processing-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
// processing-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);
```

动态的会话gap需要实现SessionWindowTimeGapExtractor接口。

### 2.2 基于计数的窗口

基于计数的窗口是根据事件的个数来对窗口进行划分的，概念跟基于时间的滚动窗口差不多，只不过窗口大小的划分，有时间变成了事件的个数。

- **滚动计数窗口（Tumbling Count Windows）**

```text
 stream
      .keyBy(1)
      .countWindow(100) \\100为事件的个数,即窗口的大小
      .sum(1);
```

- **滑动计数窗口（Sliding Count Windows）**

```text
stream
      .keyBy(1)
      .countWindow(100, 10) \\100为事件的个数，即窗口的大小，10为滑动的间隔
      .sum(1);
```

### 2.3 全局窗口（Global Windows）

全局窗口分配器将所有具有相同key的元素分配到同一个全局窗口中，这个窗口模式仅适用于用户还需自定义触发器的情况。否则，由于全局窗口没有一个自然的结尾，无法执行元素的聚合，将不会有计算被执行。

![img](https://pic1.zhimg.com/80/v2-177b1da98d513da0935452bcd9f13924_1440w.webp)

使用示例如下：

```text
DataStream<T> input = ...;
input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>);
```

### 3、触发器(Triggers)

触发器决定了一个窗口何时可以被窗口函数处理，每一个窗口分配器都有一个默认的触发器，如果默认的触发器不能满足你的需要，你可以通过调用trigger(...)来指定一个自定义的触发器。触发器的接口有5个方法来允许触发器处理不同的事件:

- onElement()方法,每个元素被添加到窗口时调用
- onEventTime()方法,当一个已注册的事件时间计时器启动时调用
- onProcessingTime()方法,当一个已注册的处理时间计时器启动时调用
- onMerge()方法，与状态性触发器相关，当使用会话窗口时，两个触发器对应的窗口合并时，合并两个触发器的状态。
- 最后一个clear()方法执行任何需要清除的相应窗口

Flink有一些内置的触发器:

- EventTimeTrigger(前面提到过)触发是根据由水印衡量的事件时间的进度来的
- ProcessingTimeTrigger 根据处理时间来触发
- CountTrigger 一旦窗口中的元素个数超出了给定的限制就会触发
- PurgingTrigger 作为另一个触发器的参数并将它转换成一个清除类型
  如果你想实现一个自定义的触发器需要继承[Trigger](https://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttps%3A%2F%2Fgithub.com%2Fapache%2Fflink%2Fblob%2Fmaster%2F%2Fflink-streaming-java%2Fsrc%2Fmain%2Fjava%2Forg%2Fapache%2Fflink%2Fstreaming%2Fapi%2Fwindowing%2Ftriggers%2FTrigger.java)类

GlobalWindow默认的触发器是NeverTrigger，是永远不会触发的，因此，如果你使用的是GlobalWindow的话，需要定义一个自定义触发器。

### 4、驱逐器(Evictors)

Flink的窗口模型允许指定一个除了WindowAssigner和Trigger之外的可选参数Evitor，这个可以通过调用evitor(...)方法来实现。这个驱逐器(evitor)可以在触发器触发之前或者之后清理窗口中的元素。为了达到这个目的，Evitor接口有两个方法:

```text
void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
```

注：指定一个Evitor要防止预聚合，因为窗口中的所有元素必须得在计算之前传递到驱逐器中