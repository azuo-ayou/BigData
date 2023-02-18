## spark-join的具体实现



我们先回忆一个RDD的构成

1. 依赖
2. 函数
3. 分区



## 1、join的理解

实现的功能



## 2、join算子在源码的实现

源码中join算子是分了三个步骤实现的，也就是经过了三个算子，分别是join、map、flapmap

1. join：生成CoGroupedRDD：经过我们理解的shuffle过程，生成[key,Array[Iterable[_]]]的数据类型
2. mapValues：生成MapPartitionsRDD：将上述的数据结构转换成[key,Tuple[Iterable,Iterable]]的数据类型
3. flatMapValues：生成MapPartitionsRDD：将每一个key对应的迭代器，打散生成[key,(value1,value2)]的数据类型

下边我们分别详细说明三个步骤

### 2.1、CoGroupedRDD

CoGroupedRDD，join算子会直接生成这个RDD，作用就是将上游依赖的RDD，按照key相同收集到一起。我们主要看他的compute方法，在看compute方法之前，先介绍一下数据结构

```scala
// join过程中，对于单条数据的封装结构
// Any ：存上游依赖RDD，key对应的value
// Int ：存的依赖的信息，也就是对应上游哪个RDD
private type CoGroupValue = (Any, Int) 


// 上述的CoGroupValue 是对于一个RDD的一条数据的封装
// 这里join过程中对于一个key，来自同一个RDD，所有的value的封装，所以本质上是一个数组
private type CoGroup = CompactBuffer[Any]


// 上述的CoGroup 是对于一个key，来自一个RDD的所有value的封装
// 这里是一个key，来自不同RDD所有条数据的封装，也是一个数组，数据的元素是CoGroup，索引是RDD的信息
// 比如上游join只有两个RDD，那么CoGroupCombiner的数组长度只有2，0号位置存当前key，在第一个RDD的所有value
private type CoGroupCombiner = Array[CoGroup]
```



这里还是要重点介绍下这个方法，这个方法的作用是生成一个，在join过程中，数据临时保存的一个数据结构，类似我们之前讲的shufflewrite过程的map，或者mr过程中的环形缓冲区，这里涉及需要定义三个方法

1. createCombiner：初始化CoGroupCombiner
2. mergeValue：CoGroupCombiner和value的合并
3. mergeCombiners：CoGroupCombiner和CoGroupCombiner的合并

```scala
private def createExternalMap(numRdds: Int) // 上游依赖的RDD数据
    : ExternalAppendOnlyMap[K, CoGroupValue, CoGroupCombiner] = {
		
      // 初始化一个Combiner结构，当一个key第一次出现的时候调用这个方法，生成一个CoGroupCombiner，并把值当前的value保存
    val createCombiner: (CoGroupValue => CoGroupCombiner) = value => {
      val newCombiner = Array.fill(numRdds)(new CoGroup)
      newCombiner(value._2) += value._1
      newCombiner
    }
      // 合并，当一个key，不是第一次来的时候，就合并到之前创建的CoGroupCombiner上，value._2代表的是第一个RDD的数据
      // 这里是CoGroupCombiner和value的合并
    val mergeValue: (CoGroupCombiner, CoGroupValue) => CoGroupCombiner =
      (combiner, value) => {
      combiner(value._2) += value._1
      combiner
    }
      // 合并CoGroupCombiner，因为会涉及到溢写；合并多次译写的数据时，就会涉及到CoGroupCombiner和CoGroupCombiner的合并
    val mergeCombiners: (CoGroupCombiner, CoGroupCombiner) => CoGroupCombiner =
      (combiner1, combiner2) => {
        var depNum = 0
        while (depNum < numRdds) {
          combiner1(depNum) ++= combiner2(depNum)
          depNum += 1
        }
        combiner1
      }
    new ExternalAppendOnlyMap[K, CoGroupValue, CoGroupCombiner](
      createCombiner, mergeValue, mergeCombiners)
  }
```





spark join的时候在这里做了更高的抽象，数据在map（可以聚合的情况）中存储的不是key-value，而是key-Combiner，当新来key-value的时候，将value聚合到，key对应的Combiner上

```scala
class ExternalAppendOnlyMap[K, V, C](
    createCombiner: V => C,//初始化一个Combiner，新来一个key，将value初始化成一个Combiner
    mergeValue: (C, V) => C, //将value聚合到Combiner，合并重复的key的value
    mergeCombiners: (C, C) => C, //两个Combiner的聚合，这里是会在合并多次溢写文件会用到
    serializer: Serializer = SparkEnv.get.serializer,
    blockManager: BlockManager = SparkEnv.get.blockManager,
    context: TaskContext = TaskContext.get(),
    serializerManager: SerializerManager = SparkEnv.get.serializerManager)
```



用map存存储，每一个key就是join用的键，每个value也就是范型C是一个combiner，他的本质是一个数组，数组的范型是CoGroup，

大小是所有依赖的RDD数量，CoGroup的本质又是列表seq

```scala

currentMap = new SizeTrackingAppendOnlyMap[K, C]
K = key
C = combiner = Array[CoGroup](numRdds)
numRdds 就是依赖的数组的数量
CoGroup = CompactBuffer[Any]
```

所以数据结构是如下：

```scala
map[key,Array[CoGroup[CompactBuffer[Any]]](numRdds)]
// map的value是每一个key，数组长度就是所依赖的RDD数量，也就是，数组的每一个元素，就是这个Key对应RDD的所有数据
```



所以到这里就不用多讲了，接下来的操作就是：读数据--溢写--合并溢写文件；最终生成的CoGroupedRDD的数据结构类型就是一个二元组：(K, Array[Iterable[_]])

Key：join用到的key

Array[Iterable[_]]：长度为2的数据，数组的每一个元素是CompactBuffer[Any]的迭代器，分别对应当前key不同RDD的所有value



### 2.2、mapValues

spark内部又接着进行了处理，把每一个key对应的数组迭代器，变成了二元组迭代器;数据结构变成了 key : (Iterable[V],Iterable[W])

```scala
def cogroup[W](other: RDD[(K, W)], partitioner: Partitioner)
      : RDD[(K, (Iterable[V], Iterable[W]))] = self.withScope {
    if (partitioner.isInstanceOf[HashPartitioner] && keyClass.isArray) {
      throw new SparkException("HashPartitioner cannot partition array keys.")
    }
     // 生成一个CoGroupedRDD
    val cg = new CoGroupedRDD[K](Seq(self, other), partitioner)
      // 只是调用了mapValues，将数组变成二元组
    cg.mapValues { case Array(vs, w1s) =>
      (vs.asInstanceOf[Iterable[V]], w1s.asInstanceOf[Iterable[W]])
    }
  }
```



### 2.3、flatMapValues

在2.2的基础上，spark又接着帮忙处理了数据，将一个key对应的二元组迭代器展开成了多条数据，具体如下：

```scala
def join[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (V, W))] = self.withScope {
    this.cogroup(other, partitioner).flatMapValues( pair =>
      for (v <- pair._1.iterator; w <- pair._2.iterator) yield (v, w)
    )
  }
// 同一个key，双重循环遍历两个来自不同RDD的迭代器
```



到这里就结束了

