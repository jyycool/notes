# 【Spark Streaming】Kafka 连接器源码分析

## 序言

![img](https:////upload-images.jianshu.io/upload_images/7547741-6f6746315a4a0773.png?imageMogr2/auto-orient/strip|imageView2/2/w/583/format/webp)

本文会讲解Spark Stream是如何与Kafka进行对接的，包括DirectInputStream和KafkaRDD是如何与KafkaConsumer交互的

理解这个的核心，在于以DirectKafkaInputDStream和KafkaRDD的compute、KafkaRDDIterator的next为中心向外延伸阅读。但本文会以顺序方式讲解

## 入口

在编写程序时，我们创建了一个DirectSream。ConsumerStrategy::Subscribe返回一个Subscribe类。



```scala
// 编写的程序
KafkaUtils.createDirectStream(
        streamingContext,
        LocationStrategies.PreferConsistent(),
        ConsumerStrategies.<String, String>Subscribe(topics, kafkaParams)
);

// ConsumerStrategies.scala
@Experimental
def Subscribe[K, V](
    topics: ju.Collection[jl.String],
    kafkaParams: ju.Map[String, Object]): ConsumerStrategy[K, V] = {
  new Subscribe[K, V](topics, kafkaParams, ju.Collections.emptyMap[TopicPartition, jl.Long]())
}
```

我们顺着方法看Subscribe类

## Subscribe的创建与启动

该方法在创建时，接受了三个参数。在本文背景下，前两个参数是开发者传入的，第三个是参数为空:

- topics 用于KafkaConsumer订阅的topic
- kafkaParams  用于创建KafkaConsumer的配置
- offsets 用于设置KafkaConsumer的offset，此处传入为emptyMap

该类的关键在于onStart方法，在启动阶段，该方法被调用于创建KafkaConsumer，执行订阅，设置offset并返回给上层。



```dart
private case class Subscribe[K, V](
    topics: ju.Collection[jl.String],
    kafkaParams: ju.Map[String, Object],
    offsets: ju.Map[TopicPartition, jl.Long]
  ) extends ConsumerStrategy[K, V] with Logging {

  def executorKafkaParams: ju.Map[String, Object] = kafkaParams

  def onStart(currentOffsets: ju.Map[TopicPartition, jl.Long]): Consumer[K, V] = {
    val consumer = new KafkaConsumer[K, V](kafkaParams)
    consumer.subscribe(topics)
    val toSeek = if (currentOffsets.isEmpty) {
      offsets
    } else {
      currentOffsets
    }

    // 3. 如果设置了`currentOffsets`或`offsets`，则为`consumer`设置offset。
    // 实际上这一段不会执行，因为到时候currentOffsets为空，且传入的offsets也是空
    if (!toSeek.isEmpty) {
      // work around KAFKA-3370 when reset is none
      // poll will throw if no position, i.e. auto offset reset none and no explicit position
      // but cant seek to a position before poll, because poll is what gets subscription partitions
      // So, poll, suppress the first exception, then seek
      val aor = kafkaParams.get(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG)
      val shouldSuppress =
        aor != null && aor.asInstanceOf[String].toUpperCase(Locale.ROOT) == "NONE"
      try {
        consumer.poll(0)
      } catch {
        case x: NoOffsetForPartitionException if shouldSuppress =>
          logWarning("Catching NoOffsetForPartitionException since " +
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG + " is none.  See KAFKA-3370")
      }
      toSeek.asScala.foreach { case (topicPartition, offset) =>
          consumer.seek(topicPartition, offset)
      }
      // we've called poll, we must pause or next poll may consume messages and set position
      consumer.pause(consumer.assignment())
    }

    consumer
  }
}
```

onStart分为以下几步:

1. 根据开发者传入的构造参数`kafkaParams`创建一个KafkaConsumer

2. 调用subscribe订阅topic

3. 如果设置了

   ```
   currentOffsets
   ```

   或

   ```
   offsets
   ```

   ，则为

   ```
   consumer
   ```

   设置offset。核心操作在于为每个

   ```
   toSeek
   ```

   调用seek方法，设置offset。最后调用pause方法，防止其设置offset。

   - 这段代码之所以冗长，主要是因为注释中提到的"KAFKA-3370"，此处不赘述。
   - 在实际运行中，这段代码并不会执行，因为传入的`currentOffsets`和`offsets`都为空

当该方法返回KafkaConsumer时，已经订阅了用户需要的topic。

## DirectKafkaInputDStream

KafkaUtils.createDirectStream最终创建一个DirectKafkaInputDStream。我们需要分析该类。



```scala
val stream = KafkaUtils.createDirectStream[String, String](
  ssc,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)

@Experimental
def createDirectStream[K, V](
    ssc: StreamingContext,
    locationStrategy: LocationStrategy,
    consumerStrategy: ConsumerStrategy[K, V]
  ): InputDStream[ConsumerRecord[K, V]] = {
  val ppc = new DefaultPerPartitionConfig(ssc.sparkContext.getConf)
  createDirectStream[K, V](ssc, locationStrategy, consumerStrategy, ppc)
}

@Experimental
def createDirectStream[K, V](
    ssc: StreamingContext,
    locationStrategy: LocationStrategy,
    consumerStrategy: ConsumerStrategy[K, V],
    perPartitionConfig: PerPartitionConfig
  ): InputDStream[ConsumerRecord[K, V]] = {
  new DirectKafkaInputDStream[K, V](ssc, locationStrategy, consumerStrategy, perPartitionConfig)
}
```

DirectKafkaInputDStream会维护当前的offset，用于划分OffsetRange，并交给Executor拉取数据。
 它有以下几个阶段：

1. 创建。接受构造参数，初始化executorKafkaParams
2. 启动, start。刷新offset
3. 运行, compute。为每个TopicPartition划分OffsetRange，生成KafkaRDD，供Executor执行

## 创建

当程序的main函数被driver执行时，DirectKafkaInputDStream被构造出来。其中初始化了executorKafkaParams，该参数是用来初始化在Executor端执行的KafkaConsumer的参数，而不是用于初始化driver端的。
 复制代码块中调用了KafkaUtils.fixKafkaParams修改参数。

另外还初始化了currentOffsets，该变量是用于维护当前offset的核心变量。



![img](https:////upload-images.jianshu.io/upload_images/7547741-114bb4a75d6b47b4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



```kotlin
private[kafka010] def fixKafkaParams(kafkaParams: ju.HashMap[String, Object]): Unit = {
  logWarning(s"overriding ${ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG} to false for executor")
  kafkaParams.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false: java.lang.Boolean)

  logWarning(s"overriding ${ConsumerConfig.AUTO_OFFSET_RESET_CONFIG} to none for executor")
  kafkaParams.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "none")

  // driver and executor should be in different consumer groups
  val originalGroupId = kafkaParams.get(ConsumerConfig.GROUP_ID_CONFIG)
  if (null == originalGroupId) {
    logError(s"${ConsumerConfig.GROUP_ID_CONFIG} is null, you should probably set it")
  }
  val groupId = "spark-executor-" + originalGroupId
  logWarning(s"overriding executor ${ConsumerConfig.GROUP_ID_CONFIG} to ${groupId}")
  kafkaParams.put(ConsumerConfig.GROUP_ID_CONFIG, groupId)

  // possible workaround for KAFKA-3135
  val rbb = kafkaParams.get(ConsumerConfig.RECEIVE_BUFFER_CONFIG)
  if (null == rbb || rbb.asInstanceOf[java.lang.Integer] < 65536) {
    logWarning(s"overriding ${ConsumerConfig.RECEIVE_BUFFER_CONFIG} to 65536 see KAFKA-3135")
    kafkaParams.put(ConsumerConfig.RECEIVE_BUFFER_CONFIG, 65536: java.lang.Integer)
  }
}
```

以上代码修改了几项参数，以便executor端执行。这就是yarn日志中的几条warn日志:



![img](https:////upload-images.jianshu.io/upload_images/7547741-217dc422f482c752.png?imageMogr2/auto-orient/strip|imageView2/2/w/1055/format/webp)



从代码可知，executor端执行的KafkaConsumer有以下修改:

1. enable.auto.commit被设置为false，因为executor端的KafkaConsumer只是为了拉取数据，不需要额外的行为

2. auto.offset.reset被设置为None，因为offset是由driver端提供的(下文分析)

3. groupId被添加了"spark-executor-"前缀, 我们结合注释可知，

   driver端和executor端都会维护KafkaConsumer，并且处于不同的消费组，他们的作用是不同的

   [[1\]](#fn1)

   。

   > // driver and executor should be in different consumer groups

## 启动

启动时，start方法被调用，有以下行为:

1. 创建consumer。调用consumer的getter，后者调用consumerStrategy.onStart，上文分析过，会返回一个订阅过topic的KafkaConsumer
2. 试探执行一次poll。调用paranoidPoll，后者会调用poll，poll的一个副作用是会恢复之前commit的进度。
3. 将currentOffsets。由于进度被恢复，调用c.position设置进度。

paranoidPoll的行为如下:

1. 调用poll(0)，恢复上次commit的进度
2. 如果收到msgs，则应找到每个分区收到的消息的最小offset，调用c.seek来设置。笔者认为，这样做的原因是:
   1. driver端并不消费信息，只为了查看offset，如果在poll中收到了信息，那一个分区中收到的offset最小的消息，就是上次commit的进度。因此刻意调用seek来设置进度。
   2. 有利于方法返回后，上层调用c.position设置currentOffsets



```tsx
@transient private var kc: Consumer[K, V] = null
def consumer(): Consumer[K, V] = this.synchronized {
  if (null == kc) {
    kc = consumerStrategy.onStart(currentOffsets.mapValues(l => new java.lang.Long(l)).asJava)
  }
  kc
}

override def start(): Unit = {
    val c = consumer
    paranoidPoll(c)
    if (currentOffsets.isEmpty) {
      currentOffsets = c.assignment().asScala.map { tp =>
        tp -> c.position(tp)
      }.toMap
    }

    // don't actually want to consume any messages, so pause all partitions
    c.pause(currentOffsets.keySet.asJava)
}

/**
 * The concern here is that poll might consume messages despite being paused,
 * which would throw off consumer position.  Fix position if this happens.
 */
private def paranoidPoll(c: Consumer[K, V]): Unit = {
  val msgs = c.poll(0)
  if (!msgs.isEmpty) {
    // position should be minimum offset per topicpartition
    msgs.asScala.foldLeft(Map[TopicPartition, Long]()) { (acc, m) =>
      val tp = new TopicPartition(m.topic, m.partition)
      val off = acc.get(tp).map(o => Math.min(o, m.offset)).getOrElse(m.offset)
      acc + (tp -> off)
    }.foreach { case (tp, off) =>
        logInfo(s"poll(0) returned messages, seeking $tp to $off to compensate")
        c.seek(tp, off)
    }
  }
}
```

## 执行

该函数是核心步骤，会被周期性地执行，用于划分OffsetRange，交给executor端根据OffsetRange拉取数据。

> 由于本文只分析spark stream，不分析spark的机制，因此略过compute方法是如何被调用的。了解rdd机制的同学应该明白，getPartitions用来定义数据的划分，compute用来定义数据的计算。我们此处了解compute的行为即可。
>  compute方法有以下几步(已用注释标出):

1. 获取untilOffsets。调用latestOffsets获取当前的最大offset，调用clamp进行速率控制
2. 设置OffsetRange。取到untilOffsets后，结合currentOffsets，生成OffsetRange。由此可知，当前消费进度是以currentOffsets为主
3. 设置KafkaRDD。传入了executor端的参数executorKafkaParams和offset的消费范围offsetRanges。下文分析该类。
4. 异步提交offset。处理已经完成的offset，将它们提交
5. 把第3步设置的KafkaRDD返回



```kotlin
override def compute(validTime: Time): Option[KafkaRDD[K, V]] = {
    // 1. 获取untilOffsets 
    val untilOffsets = clamp(latestOffsets())
    // 2. 设置OffsetRange
    val offsetRanges = untilOffsets.map { case (tp, uo) =>
      val fo = currentOffsets(tp)
      OffsetRange(tp.topic, tp.partition, fo, uo)
    }
    val useConsumerCache = context.conf.getBoolean("spark.streaming.kafka.consumer.cache.enabled",
      true)
    // 3. 设置KafkaRDD
    val rdd = new KafkaRDD[K, V](context.sparkContext, executorKafkaParams, offsetRanges.toArray,
      getPreferredHosts, useConsumerCache)

    // Report the record number and metadata of this batch interval to InputInfoTracker.
    val description = offsetRanges.filter { offsetRange =>
      // Don't display empty ranges.
      offsetRange.fromOffset != offsetRange.untilOffset
    }.map { offsetRange =>
      s"topic: ${offsetRange.topic}\tpartition: ${offsetRange.partition}\t" +
        s"offsets: ${offsetRange.fromOffset} to ${offsetRange.untilOffset}"
    }.mkString("\n")
    // Copy offsetRanges to immutable.List to prevent from being modified by the user
    val metadata = Map(
      "offsets" -> offsetRanges.toList,
      StreamInputInfo.METADATA_KEY_DESCRIPTION -> description)
    val inputInfo = StreamInputInfo(id, rdd.count, metadata)
    ssc.scheduler.inputInfoTracker.reportInfo(validTime, inputInfo)

    currentOffsets = untilOffsets
    // 4. 异步提交offset
    commitAll()
    // 5. 返回RDD
    Some(rdd)
  }
```

**获取untilOffsets**
 我们分析compute的第一步，首先调用latestOffsets，然后调用clamp限制消费速率。



```scala
/**
 * Returns the latest (highest) available offsets, taking new partitions into account.
 */
protected def latestOffsets(): Map[TopicPartition, Long] = {
  val c = consumer
  paranoidPoll(c)
  val parts = c.assignment().asScala

  // make sure new partitions are reflected in currentOffsets
  val newPartitions = parts.diff(currentOffsets.keySet)
  // position for new partitions determined by auto.offset.reset if no commit
  currentOffsets = currentOffsets ++ newPartitions.map(tp => tp -> c.position(tp)).toMap
  // don't want to consume messages, so pause
  c.pause(newPartitions.asJava)
  // find latest available offsets
  c.seekToEnd(currentOffsets.keySet.asJava)
  parts.map(tp => tp -> c.position(tp)).toMap
}
```

从注释可知，latestOffsets是为了返回当前可用的最大offset，并考虑新加入的Partition。

- 如何理解"考虑新加入的Partition"? 代码首先调用了paranoidPoll刷新分区视野，再调用parts.diff得到新出现的分区，最后调用currentOffsets ++...把新分区的offset加入currentOffsets中。
- 如何理解“返回当前可用的最大offset”? 当新分区的offset都被加入考虑后，调用c.seekToEnd设置最大offset[[2\]](#fn2)。最后为每个分区调用c.position，返回这些最大offset。

> 有人可能疑惑，调用c.seekToEnd后并调用c.position确实能取得最大offset，但这也修改了offset。在平时的开发中,c.position往往充当"当前消费进度"的语义，那在此处，c.seekToEnd势必会跳过没消费的消息，直接定位到最新进度，这会导致消息漏处理吗？
>  答案是不会。因为currentOffsets就代表了"当前消费进度"，而由于多次调用c.seekToEnd，c.position的语义变成了"当前最大offset"。这两者之间的offset就是OffsetRange。
>  图示如下:
>
> ![img](https:////upload-images.jianshu.io/upload_images/7547741-6ef62bb0ffa5718f.png?imageMogr2/auto-orient/strip|imageView2/2/w/787/format/webp)

**异步提交offset**
 完成消费的offset会被放入commitQueue。
 本方法从commitQueue中循环取出提交了的offset，并放入变量m，最后调用consumer.commitAsync异步提交。



```scala
/**
 * Queue up offset ranges for commit to Kafka at a future time.  Threadsafe.
 * @param offsetRanges The maximum untilOffset for a given partition will be used at commit.
 * @param callback Only the most recently provided callback will be used at commit.
 */
def commitAsync(offsetRanges: Array[OffsetRange], callback: OffsetCommitCallback): Unit = {
  commitCallback.set(callback)
  commitQueue.addAll(ju.Arrays.asList(offsetRanges: _*))
}

protected def commitAll(): Unit = {
    val m = new ju.HashMap[TopicPartition, OffsetAndMetadata]()
    var osr = commitQueue.poll()
    while (null != osr) {
      val tp = osr.topicPartition
      val x = m.get(tp)
      val offset = if (null == x) { osr.untilOffset } else { Math.max(x.offset, osr.untilOffset) }
      m.put(tp, new OffsetAndMetadata(offset))
      osr = commitQueue.poll()
    }
    if (!m.isEmpty) {
      consumer.commitAsync(m, commitCallback.get)
    }
  }
```

## KafkaRDD

构造时会传入offsetRanges, 并检查两项KafkaConsumer的参数，这是与上文KafkaUtils.fixKafkaParams相应的。



![img](https:////upload-images.jianshu.io/upload_images/7547741-def766c9515bc201.png?imageMogr2/auto-orient/strip|imageView2/2/w/1001/format/webp)

1. getPartitions根据offsetRange建立KafkaRDDPartition。由代码可知，**KafkaRDDPartition的数量与TopicPartition的数量相等**，也就是每个TopicPartition都由一个KafkaConsumer读取。
2. compute根据传入的Partition(由第一步计算得到)，生成并返回一个KafkaRDDIterator



```sacla
override def getPartitions: Array[Partition] = {
  offsetRanges.zipWithIndex.map { case (o, i) =>
      new KafkaRDDPartition(i, o.topic, o.partition, o.fromOffset, o.untilOffset)
  }.toArray
}

override def compute(thePart: Partition, context: TaskContext): Iterator[ConsumerRecord[K, V]] = {
    val part = thePart.asInstanceOf[KafkaRDDPartition]
    assert(part.fromOffset <= part.untilOffset, errBeginAfterEnd(part))
    if (part.fromOffset == part.untilOffset) {
      logInfo(s"Beginning offset ${part.fromOffset} is the same as ending offset " +
        s"skipping ${part.topic} ${part.partition}")
      Iterator.empty
    } else {
      new KafkaRDDIterator(part, context)
    }
  }
```

## KafkaRDDIterator

> 虽然没摸透Spark源码，但笔者推测KafkaRDDIterator会被分配给某个Executor执行，该类方法是由Executor执行的。



```scala
/**
 * An iterator that fetches messages directly from Kafka for the offsets in partition.
 * Uses a cached consumer where possible to take advantage of prefetching
 */
private class KafkaRDDIterator(
    part: KafkaRDDPartition,
    context: TaskContext) extends Iterator[ConsumerRecord[K, V]] {

  logInfo(s"Computing topic ${part.topic}, partition ${part.partition} " +
    s"offsets ${part.fromOffset} -> ${part.untilOffset}")

  val groupId = kafkaParams.get(ConsumerConfig.GROUP_ID_CONFIG).asInstanceOf[String]

  context.addTaskCompletionListener{ context => closeIfNeeded() }

  val consumer = if (useConsumerCache) {
    CachedKafkaConsumer.init(cacheInitialCapacity, cacheMaxCapacity, cacheLoadFactor)
    if (context.attemptNumber > 1) {
      // just in case the prior attempt failures were cache related
      CachedKafkaConsumer.remove(groupId, part.topic, part.partition)
    }
    CachedKafkaConsumer.get[K, V](groupId, part.topic, part.partition, kafkaParams)
  } else {
    CachedKafkaConsumer.getUncached[K, V](groupId, part.topic, part.partition, kafkaParams)
  }

  var requestOffset = part.fromOffset

  def closeIfNeeded(): Unit = {
    if (!useConsumerCache && consumer != null) {
      consumer.close
    }
  }

  override def hasNext(): Boolean = requestOffset < part.untilOffset

  override def next(): ConsumerRecord[K, V] = {
    assert(hasNext(), "Can't call getNext() once untilOffset has been reached")
    val r = consumer.get(requestOffset, pollTimeout)
    requestOffset += 1
    r
  }
}
```

1. 看到构造代码，假设useConsumerCache为真，则consumer会取自CachedKafkaConsumer.get。
2. 在构造代码中设置**requestOffset**为part.fromOffset。hasNext判断是否小于part.untilOffset。每次调用next，读取一条消息，requestOffset加1。
3. 在next中调用consumer.get，读取一条消息。

我们有必要再查看CachedKafkaConsumer

## CachedKafkaConsumer

先看CachedKafkaConsumer的静态类定义



```tsx
private[kafka010]
object CachedKafkaConsumer extends Logging {

  private case class CacheKey(groupId: String, topic: String, partition: Int)

  // Don't want to depend on guava, don't want a cleanup thread, use a simple LinkedHashMap
  private var cache: ju.LinkedHashMap[CacheKey, CachedKafkaConsumer[_, _]] = null
  ...
```

以CacheKey为key，维护了一个名为cache的Map。
 再看到CacheKey有三个属性，可知CachedKafkaConsumer的作用是以groupId, topic, partition为key，存储CachedKafkaConsumer对象

在看CachedKafkaConsumer的类构造:

1. 会判断groupId是否与参数相等，这与上文KafkaUtils.fixKafkaParams相应
2. 会存储一个topicPartition，代表它负责的分区
3. consumer变量创建并维护一个Kafkaconsumer，调用c.assign完成分配
4. buffer是该类功能的核心，它一个List的迭代器
5. nextOffset指当前读取的offset，注意初始化为-2



```kotlin
/**
 * Consumer of single topicpartition, intended for cached reuse.
 * Underlying consumer is not threadsafe, so neither is this,
 * but processing the same topicpartition and group id in multiple threads is usually bad anyway.
 */
private[kafka010]
class CachedKafkaConsumer[K, V] private(
  val groupId: String,
  val topic: String,
  val partition: Int,
  val kafkaParams: ju.Map[String, Object]) extends Logging {

  assert(groupId == kafkaParams.get(ConsumerConfig.GROUP_ID_CONFIG),
    "groupId used for cache key must match the groupId in kafkaParams")

  val topicPartition = new TopicPartition(topic, partition)

  protected val consumer = {
    val c = new KafkaConsumer[K, V](kafkaParams)
    val tps = new ju.ArrayList[TopicPartition]()
    tps.add(topicPartition)
    c.assign(tps)
    c
  }

  // TODO if the buffer was kept around as a random-access structure,
  // could possibly optimize re-calculating of an RDD in the same batch
  protected var buffer = ju.Collections.emptyList[ConsumerRecord[K, V]]().iterator
  protected var nextOffset = -2L
```

看到get的方法和注释: **每次读取对应offset的一条消息，在顺序访问时会使用缓存，随机访问时效率极低**。



```scala
/**
 * Get the record for the given offset, waiting up to timeout ms if IO is necessary.
 * Sequential forward access will use buffers, but random access will be horribly inefficient.
 */
def get(offset: Long, timeout: Long): ConsumerRecord[K, V] = {
  logDebug(s"Get $groupId $topic $partition nextOffset $nextOffset requested $offset")
  // 第一次调用时nextOffset为初始化的-2L，肯定不等。如果是随机访问，也会进入此处判断。
  if (offset != nextOffset) {
    logInfo(s"Initial fetch for $groupId $topic $partition $offset")
    seek(offset)
    poll(timeout)
  }

  // 没有下一个时，等待读取
  if (!buffer.hasNext()) { poll(timeout) }
  assert(buffer.hasNext(),
    s"Failed to get records for $groupId $topic $partition $offset after polling for $timeout")
  // 取得缓存的下一条消息
  var record = buffer.next()

  if (record.offset != offset) {
    // 意外，取得的记录offset与需要的不同，需要重新定位
    logInfo(s"Buffer miss for $groupId $topic $partition $offset")
    seek(offset)
    poll(timeout)
    assert(buffer.hasNext(),
      s"Failed to get records for $groupId $topic $partition $offset after polling for $timeout")
    record = buffer.next()
    assert(record.offset == offset,
      s"Got wrong record for $groupId $topic $partition even after seeking to offset $offset")  // 即使重新定位，还是取得了错误的消息
  }

  nextOffset = offset + 1
  record
}
```

所以**CachedKafkaConsumer的作用是预读取一批消息并缓存，因为每次调用poll可能读取到的消息数不等，因此先缓存起来，而上层每次调用get只读取一条消息**。

## 总结

driver端和每个Executor都会维护KafkaConsumer。
 driver的KafkaConsumer自己处于一个消费组，Executor端的KafkaConsumer们共同属于一个消费组。Executor端的groupId具有"spark-executor"前缀。

- driver端会维护一个KafkaConsumer，poll(0), seekToEnd用来定位offset范围，但并不为了消费。获取到的offset范围会分发给各个executor执行消费。
- 每个TopicPartition会对应一个KafkaConsumer，一个Executor可能会被分配消费多个TopicPartition。

如图所示。

![img](https:////upload-images.jianshu.io/upload_images/7547741-6f6746315a4a0773.png?imageMogr2/auto-orient/strip|imageView2/2/w/583/format/webp)

但尚未搞清楚getPartitions和compute在spark内是如何被调用的，以及任务是如何从driver发送给executor的。

------

1. 后文会分析，其实driver端维护的KafkaConsumer是用来维护offset的，每个executor端都会维护一个KafkaConsumer，单纯为了拉取数据。
2. seekToEnd会向服务端发出一个请求，获取当前最大的offset并设置