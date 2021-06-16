![image-20210608215400269](4Kafka源码.assets/image-20210608215400269-3160442.png)

![image-20210608215556847](4Kafka源码.assets/image-20210608215556847-3160558.png)

## 1 日志模块

日志在磁盘的组成图

<img src="4Kafka源码.assets/image-20210610124553630-3300355.png" alt="image-20210610124553630" style="zoom: 33%;" />

#### 1.1 保存消息文件的对象是怎么实现的？LogSegment

- 为什么？为什么不单独用一个partition对应一个Log Segment？TODO: 为什么要切分日志。
- 是什么？每个partition目录下，都有多个log segment，用来存储日志和索引。
- 做什么？适合做日志存储和查询。
- 优缺点？offset在文件名中，有利于快速查找某一条消息。缺点是如果LogSegment太多，内存的对象就会很大，占用内存。
- 技术关键点？
  - 成员：log and index
  - 方法：append、read、recover



1. log 是保存实际的消息，FileRecords
2. index 是逻辑的offset到物理的position的映射，OffsetIndex

##### append: 日志写入

传入参数：写入日志的消息体、包括它的2类时间戳

- 先看log是否为空，空的话，更新rolling（日志切分）的时间戳（TODO: 日志切分需要搞清楚什么情况下会发生）
- 判断消息的位移值是否合法，合法就写入消息
- 更新最大的时间戳和它对应的offset
- 最后判断是否需要新增索引 ? 当前写入的字节数大于4K。

```scala
 @nonthreadsafe
  def append(largestOffset: Long,
             largestTimestamp: Long,
             shallowOffsetOfMaxTimestamp: Long,
             records: MemoryRecords): Unit = {
    if (records.sizeInBytes > 0) {
      val physicalPosition = log.sizeInBytes()
      if (physicalPosition == 0) //1 先看log是否为空，空的话，更新rolling（日志切分）的时间戳
        rollingBasedTimestamp = Some(largestTimestamp)

      ensureOffsetInRange(largestOffset) // 2 判断消息的位移值是否合法，合法就写入消息

      // append the messages
      val appendedBytes = log.append(records) // 写入消息
      // Update the in memory max timestamp and corresponding offset.
      if (largestTimestamp > maxTimestampSoFar) {
        maxTimestampSoFar = largestTimestamp
        offsetOfMaxTimestampSoFar = shallowOffsetOfMaxTimestamp
      }
      // append an entry to the index (if needed)
      if (bytesSinceLastIndexEntry > indexIntervalBytes) {
        offsetIndex.append(largestOffset, physicalPosition)
        timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestampSoFar)
        bytesSinceLastIndexEntry = 0
      }
      bytesSinceLastIndexEntry += records.sizeInBytes
    }
  }
```

#####  read：读取日志

传入参数：读取日志的开始的offset，最大能读的size条数和文件的最大position，是否允许至少读取一条日志

- 翻译起始的offset为实际的文件position

- 计算这次能够读取的最大条数，取min(输入参数最大读取条数 , 文件的最大能读取的条数)

- 直接读取，返回一个FetchDataInfo

  ```scala
  @threadsafe
    def read(startOffset: Long,
             maxSize: Int,
             maxPosition: Long = size,
             minOneMessage: Boolean = false): FetchDataInfo = {
  
      val startOffsetAndSize = translateOffset(startOffset) // 1 翻译起始的offset为实际的文件position
  
      // if the start position is already off the end of the log, return null
      if (startOffsetAndSize == null)
        return null
  
      val startPosition = startOffsetAndSize.position
      val offsetMetadata = LogOffsetMetadata(startOffset, this.baseOffset, startPosition)
  
      val adjustedMaxSize =
        if (minOneMessage) math.max(maxSize, startOffsetAndSize.size)
        else maxSize
  
      // return a log segment but with zero size in the case below
      if (adjustedMaxSize == 0)
        return FetchDataInfo(offsetMetadata, MemoryRecords.EMPTY)
  
      // calculate the length of the message set to read based on whether or not they gave us a maxOffset
      val fetchSize: Int = min((maxPosition - startPosition).toInt, adjustedMaxSize) // 2 计算这次能够读取的最大条数，取min(输入参数最大读取条数 , 文件的最大能读取的条数)
  
      FetchDataInfo(offsetMetadata, log.slice(startPosition, fetchSize), // 3 直接读取，返回一个FileRecords
        firstEntryIncomplete = adjustedMaxSize < startOffsetAndSize.size)
    }
  ```

  

##### recover: 恢复日志段

recovery 使用场景：Broker在启动的时候，会把所有的Log Segment文件加载到内存中，生成一系列的LogSegment实例。（应该不是把所有的文件都读进去了，只是读个文件名）

传入参数：ProducerStateManager Producer的状态管理员、LeaderEpochFileCache 这个Partition的LeaderEpoch信息。

- 1 清空所有index的数据，包括offsetindex，timeindex, txnindex
- 2 遍历Log Segment里面，所有的消息集合Batch
  - 2.1 校验
  - 2.2 更新这批Batch消息的最大时间戳和最大位移  
  - 2.3 更新index 
  - 2.4 更新LeaderEpoch缓存和ProducerStateManager
- 3 如果发现读取的日志消息少于LogSegement本身的字节数，就截断日志和index文件

```scala
 @nonthreadsafe
  def recover(producerStateManager: ProducerStateManager, leaderEpochCache: Option[LeaderEpochFileCache] = None): Int = {
    offsetIndex.reset() // 1 清空所有index的数据
    timeIndex.reset()
    txnIndex.reset()
    var validBytes = 0
    var lastIndexEntry = 0
    maxTimestampSoFar = RecordBatch.NO_TIMESTAMP
    try {
      for (batch <- log.batches.asScala) { // 2 遍历所有日志
        batch.ensureValid() // 2.1 校验
        ensureOffsetInRange(batch.lastOffset)

        // The max timestamp is exposed at the batch level, so no need to iterate the records
        // 2.2 更新最大时间戳和最大位移  
        if (batch.maxTimestamp > maxTimestampSoFar) {
          maxTimestampSoFar = batch.maxTimestamp
          offsetOfMaxTimestampSoFar = batch.lastOffset
        }

        // Build offset index // 2.3 更新index 
        if (validBytes - lastIndexEntry > indexIntervalBytes) {
          offsetIndex.append(batch.lastOffset, validBytes)
          timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestampSoFar)
          lastIndexEntry = validBytes
        }
        validBytes += batch.sizeInBytes()
		// 2.4 更新LeaderEpoch缓存和ProducerStateManager
        if (batch.magic >= RecordBatch.MAGIC_VALUE_V2) {
          leaderEpochCache.foreach { cache =>
            if (batch.partitionLeaderEpoch > 0 && cache.latestEpoch.forall(batch.partitionLeaderEpoch > _))
              cache.assign(batch.partitionLeaderEpoch, batch.baseOffset)
          }
          updateProducerState(producerStateManager, batch)
        }
      }
    } catch {
      
    }
    val truncated = log.sizeInBytes - validBytes
    if (truncated > 0)
      debug(s"Truncated $truncated invalid bytes at the end of segment ${log.file.getAbsoluteFile} during recovery")

    log.truncateTo(validBytes) // 3 截断日志和index文件
    offsetIndex.trimToValidSize()
    // A normally closed segment always appends the biggest timestamp ever seen into log segment, we do this as well.
    timeIndex.maybeAppend(maxTimestampSoFar, offsetOfMaxTimestampSoFar, skipFullCheck = true)
    timeIndex.trimToValidSize()
    truncated
  }
```

##### 总结

- append的时候，什么时候更新的索引？（读取的消息累计大于4KB）
- read的时候，minMessage参数的作用是什么？(至少可以读取一条message，防止消费饥饿)
- recovery的时候，要读取LogSegment本地文件，然后构建索引。这会造成Kafka Broker启动很慢，因为要读很多本地磁盘文件的嘛。



#### 1.2 Log是如何加载log segment的？

![image-20210614161716507](4Kafka源码.assets/image-20210614161716507-3658638.png)

Kafka定义的文件类型有哪些？.index .log .timeindex .txnindex , 还有.swap .cleaned .snapshot .deteled

- .swap 和 .clean 是日志做campaction的中间产物
- .deteled 删除日志文件。（异步）
- .snapshot 事务相关的文件

maybeIncrementHighWatermark 有什么作用？当leader收到follower所有提交的(HW,LEO)后，会判断是否更新高水位。

#### 1.3 Log对象的常见操作

![image-20210616083322626](4Kafka源码.assets/image-20210616083322626-3803604.png)

##### 高水位值

```scala
// 高水位值的定义
@volatile private var highWatermarkMetadata: LogOffsetMetadata = LogOffsetMetadata(logStartOffset) // volatile LogOffsetMetadata类型，初始值是logStartOffset

/*
 * A log offset structure, including: LogOffsetMetata包含三个类型：
 *  1. the message offset （消息位移值）
 *  2. the base message offset of the located segment （segment的起始位移值）
 *  3. the physical position on the located segment（该位移值所在的物理磁盘位置）
 */
case class LogOffsetMetadata(messageOffset: Long,
                             segmentBaseOffset: Long = Log.UnknownOffset,
                             relativePositionInSegment: Int = LogOffsetMetadata.UnknownFilePosition)
```

- 高水位值的获取
  - 读取时日志不能被关闭
  - 没有获得完整的高水位值，就通过读取日志，来构造一个高水位；同时更新它

```scala
private def fetchHighWatermarkMetadata: LogOffsetMetadata = {
    checkIfMemoryMappedBufferClosed()

    val offsetMetadata = highWatermarkMetadata 
    if (offsetMetadata.messageOffsetOnly) {
      lock.synchronized {
        val fullOffset = convertToOffsetMetadataOrThrow(highWatermark)
        updateHighWatermarkMetadata(fullOffset)
        fullOffset
      }
    } else {
      offsetMetadata
    }
  }
```

- 更新高水位值
  - 高水位一定在[Log Start Offset, Log End Offset]之间
  - 有2个方法，一个用来Follower更新高水位；一个用来Leader更新高水位。

```scala
def updateHighWatermark(hw: Long): Long = {
    val newHighWatermark = if (hw < logStartOffset)
      logStartOffset
    else if (hw > logEndOffset)
      logEndOffset
    else
      hw
    updateHighWatermarkMetadata(LogOffsetMetadata(newHighWatermark))
    newHighWatermark
  }

def maybeIncrementHighWatermark(newHighWatermark: LogOffsetMetadata): Option[LogOffsetMetadata] = {
    lock.synchronized {
      val oldHighWatermark = fetchHighWatermarkMetadata

      // Ensure that the high watermark increases monotonically. We also update the high watermark when the new
      // offset metadata is on a newer segment, which occurs whenever the log is rolled to a new segment.
      if (oldHighWatermark.messageOffset < newHighWatermark.messageOffset ||
        (oldHighWatermark.messageOffset == newHighWatermark.messageOffset && oldHighWatermark.onOlderSegment(newHighWatermark))) {
        updateHighWatermarkMetadata(newHighWatermark)
        Some(oldHighWatermark)
      } else {
        None
      }
    }
  }
```

##### 日志段管理：ConcurrentSkipListMap

```scala
private val segments: ConcurrentNavigableMap[java.lang.Long, LogSegment] = new ConcurrentSkipListMap[java.lang.Long, LogSegment] // 线程安全，可排序的Map
```

利用跳表的操作来实现增删改查。

1. 添加addSegment
2. 删除：根据留存策略来删除
   - 基于时间
   - 基于空间
   - 基于LogStartOffset
3. 修改：直接利用Map的put特性，进行替换。
4. 查询：利用ConcurrentSkipListMap来实现
   - segments.firstEntry
   - segments.lastEntry
   - segments.highEntry
   - segments.floorEntry

##### LEO的更新时机

1. 日志初始化时候
2. 写入新消息的时候
3. Log对象切分（Log Roll）的时候，一旦当前写入的日志段满了，就创建1个全新的日志段对象。
4. 日志截断的时候（Truncate）

##### 写操作

- appendAsLeader：写leader副本
- appendAsFollower：Follower同步副本的时候

<img src="4Kafka源码.assets/image-20210616092934646-3806976.png" alt="image-20210616092934646" style="zoom:50%;" />



##### 读操作

```scala
def read(startOffset: Long,
           maxLength: Int,
           isolation: FetchIsolation,
           minOneMessage: Boolean): FetchDataInfo 
```

- isolation:读取设置的级别，主要控制能够读取的最大位移值。是幂等的，还是事务的，还是普通的，多用于kafka事务。

