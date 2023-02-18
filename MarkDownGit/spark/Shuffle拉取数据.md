### maptask元数据获取

梳理清楚拉去数据的流程

1. 当前task对应需要读取上游哪些分区的数据
2. 上游每一个maptask都会产生n个分区的数据
3. 上游多个maptask可能会存在于同一个excutor上



几个概念需要明白

- BlockId：一个maptask，的每一个分区都会生成一个BlockId
- BlockManagerId：一个excutor会有一个BlockManagerId
- mapOutputTracker：存放map信息的结构，通过shuffleId能取到当前shuffle阶段所有maptask的元数据信息，一般会搭配起始的分区来使用



所以通过mapOutputTracker和BlockManagerId就可以取到，当前shuffle阶段的所有上游maptask的元数据信息





### 请求封装

当我们获取到所有BlockId，spark将这些数据划分为一个个的请求

遍历每一个BlockManagerId（节点，excutor），将这个节点的BlockId数据大小累加，如果超过了单个请求的数据最大size或者超过了单个请求最多的Block数据量，那么就封装为一个请求



### 请求数据

将上述封装的请求，当前task一次最多允许五个请求正在拉取数据；当map端接收到rpc请求后，会依据shuffleId，mapId定位到文件，然后根据reduceId定位到这个文件需要读取的数据。将数据返回会给当前task



IndexShuffleBlockResolver



关键代码

```scala
override def getBlockData(blockId: ShuffleBlockId): ManagedBuffer = {
  // The block is actually going to be a range of a single map output file for this map, so
  // find out the consolidated file, then the offset within that from our index
  val indexFile = getIndexFile(blockId.shuffleId, blockId.mapId)

  // SPARK-22982: if this FileInputStream's position is seeked forward by another piece of code
  // which is incorrectly using our file descriptor then this code will fetch the wrong offsets
  // (which may cause a reducer to be sent a different reducer's data). The explicit position
  // checks added here were a useful debugging aid during SPARK-22982 and may help prevent this
  // class of issue from re-occurring in the future which is why they are left here even though
  // SPARK-22982 is fixed.
  val channel = Files.newByteChannel(indexFile.toPath)
  channel.position(blockId.reduceId * 8L)
  val in = new DataInputStream(Channels.newInputStream(channel))
  try {
    val offset = in.readLong()
    val nextOffset = in.readLong()
    val actualPosition = channel.position()
    val expectedPosition = blockId.reduceId * 8L + 16
    if (actualPosition != expectedPosition) {
      throw new Exception(s"SPARK-22982: Incorrect channel position after index file reads: " +
        s"expected $expectedPosition but actual position was $actualPosition.")
    }
    new FileSegmentManagedBuffer(
      transportConf,
      getDataFile(blockId.shuffleId, blockId.mapId),
      offset,
      nextOffset - offset)
  } finally {
    in.close()
  }
}
```



```scala
override def getMapSizesByExecutorId(shuffleId: Int, startPartition: Int, endPartition: Int)
    : Seq[(BlockManagerId, Seq[(BlockId, Long)])] = {
  logDebug(s"Fetching outputs for shuffle $shuffleId, partitions $startPartition-$endPartition")
  val statuses = getStatuses(shuffleId)
  try {
    MapOutputTracker.convertMapStatuses(shuffleId, startPartition, endPartition, statuses)
  } catch {
    case e: MetadataFetchFailedException =>
      // We experienced a fetch failure so our mapStatuses cache is outdated; clear it:
      mapStatuses.clear()
      throw e
  }
}

def convertMapStatuses(
      shuffleId: Int,
      startPartition: Int,
      endPartition: Int,
      statuses: Array[MapStatus]): Seq[(BlockManagerId, Seq[(BlockId, Long)])] = {
    assert (statuses != null)
    val splitsByAddress = new HashMap[BlockManagerId, ArrayBuffer[(BlockId, Long)]]
    for ((status, mapId) <- statuses.zipWithIndex) {
      if (status == null) {
        val errorMessage = s"Missing an output location for shuffle $shuffleId"
        logError(errorMessage)
        throw new MetadataFetchFailedException(shuffleId, startPartition, errorMessage)
      } else {
        for (part <- startPartition until endPartition) {
          splitsByAddress.getOrElseUpdate(status.location, ArrayBuffer()) +=
            ((ShuffleBlockId(shuffleId, mapId, part), status.getSizeForBlock(part)))
        }
      }
    }

    splitsByAddress.toSeq
  }
```



关键代码

```scala
private[this] def splitLocalRemoteBlocks(): ArrayBuffer[FetchRequest] = {
  // Make remote requests at most maxBytesInFlight / 5 in length; the reason to keep them
  // smaller than maxBytesInFlight is to allow multiple, parallel fetches from up to 5
  // nodes, rather than blocking on reading output from one node.
  val targetRequestSize = math.max(maxBytesInFlight / 5, 1L)
  logDebug("maxBytesInFlight: " + maxBytesInFlight + ", targetRequestSize: " + targetRequestSize
    + ", maxBlocksInFlightPerAddress: " + maxBlocksInFlightPerAddress)

  // Split local and remote blocks. Remote blocks are further split into FetchRequests of size
  // at most maxBytesInFlight in order to limit the amount of data in flight.
  val remoteRequests = new ArrayBuffer[FetchRequest]

  // Tracks total number of blocks (including zero sized blocks)
  var totalBlocks = 0
  for ((address, blockInfos) <- blocksByAddress) {
    totalBlocks += blockInfos.size
    if (address.executorId == blockManager.blockManagerId.executorId) {
      // Filter out zero-sized blocks
      localBlocks ++= blockInfos.filter(_._2 != 0).map(_._1)
      numBlocksToFetch += localBlocks.size
    } else {
      val iterator = blockInfos.iterator
      var curRequestSize = 0L
      var curBlocks = new ArrayBuffer[(BlockId, Long)]
      while (iterator.hasNext) {
        val (blockId, size) = iterator.next()
        // Skip empty blocks
        if (size > 0) {
          curBlocks += ((blockId, size))
          remoteBlocks += blockId
          numBlocksToFetch += 1
          curRequestSize += size
        } else if (size < 0) {
          throw new BlockException(blockId, "Negative block size " + size)
        }
        if (curRequestSize >= targetRequestSize ||
            curBlocks.size >= maxBlocksInFlightPerAddress) {
          // Add this FetchRequest
          remoteRequests += new FetchRequest(address, curBlocks)
          logDebug(s"Creating fetch request of $curRequestSize at $address "
            + s"with ${curBlocks.size} blocks")
          curBlocks = new ArrayBuffer[(BlockId, Long)]
          curRequestSize = 0
        }
      }
      // Add in the final request
      if (curBlocks.nonEmpty) {
        remoteRequests += new FetchRequest(address, curBlocks)
      }
    }
  }
  logInfo(s"Getting $numBlocksToFetch non-empty blocks out of $totalBlocks blocks")
  remoteRequests
}
```