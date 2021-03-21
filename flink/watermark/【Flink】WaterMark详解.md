# 【Flink】WaterMark详解

## 背景

![img](https:////upload-images.jianshu.io/upload_images/3205503-384a6ac8f7ec926b.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1093)

实时计算中，数据时间比较敏感。有eventTime和processTime区分，一般来说eventTime是从原始的消息中提取过来的，processTime是Flink自己提供的，Flink中一个亮点就是可以基于eventTime计算，这个功能很有用，因为实时数据可能会经过比较长的链路，多少会有延时，并且有很大的不确定性，对于一些需要精确体现事件变化趋势的场景中，单纯使用processTime显然是不合理的。

## 概念

watermark是一种衡量Event Time进展的机制，它是数据本身的一个隐藏属性。通常基于Event Time的数据，自身都包含一个timestamp.watermark是用于处理乱序事件的，而正确的处理乱序事件，通常用watermark机制结合window来实现。

流处理从事件产生，到流经source，再到operator，中间是有一个过程和时间的。虽然大部分情况下，流到operator的数据都是按照事件产生的时间顺序来的，但是也不排除由于网络、背压等原因，导致乱序的产生（out-of-order或者说late element）。

***但是对于late element，我们又不能无限期的等下去，必须要有个机制来保证一个特定的时间后，必须触发window去进行计算了。这个特别的机制，就是watermark***。

> 1. watermark 是在 source 中注入数据中的一个隐藏的的属性, 数据自身都包含一个timestamp.watermark 用于处理乱序事件
> 2. watermark 的主要用于触发 window 计算

## window划分

window的设定无关数据本身，而是系统定义好了的。

window是flink中划分数据一个基本单位，window的划分方式是固定的，默认会根据自然时间划分window，并且划分方式是前闭后开。

| window划分 | w1                  | w2                  | w3                  |
| ---------- | ------------------- | ------------------- | ------------------- |
| 3s         | [00:00:00~00:00:03) | [00:00:03~00:00:06) | [00:00:06~00:00:09) |
| 5s         | [00:00:00~00:00:05) | [00:00:05~00:00:10) | [00:00:10~00:00:15) |
| 10s        | [00:00:00~00:00:10) | [00:00:10~00:00:20) | [00:00:20~00:00:30) |
| 1min       | [00:00:00~00:01:00) | [00:01:00~00:02:00) | [00:02:00~00:03:00) |

### 示例

如果设置最大允许的乱序时间是10s，滚动时间窗口为5s

```json
{"datetime":"2019-03-26 16:25:24","name":"zhangsan"}
//currentThreadId:38,key:zhangsan, eventTime:[2019-03-26 16:25:24], currentMaxTimestamp:[2019-03-26 16:25:24], watermark:[2019-03-26 16:25:14]
```

触达改记录的时间窗口应该为`2019-03-26 16:25:20~2019-03-26 16:25:25`
即当有数据eventTime >= watermark(2019-03-26 16:25:35) 时

> 这里说明了触发窗口计算的条件是, 该条数据的 WaterMark 已经到达窗口结束时间 
>
> `WaterMark(MaxEventTime - MaxOutOfOrderness) >= WindowEndTime` 
>
> 

```json
{"datetime":"2019-03-26 16:25:35","name":"zhangsan"}
//currentThreadId:38,key:zhangsan,eventTime:[2019-03-26 16:25:35],currentMaxTimestamp:[2019-03-26 16:25:35],watermark:[2019-03-26 16:25:25]
//(zhangsan,1,2019-03-26 16:25:24,2019-03-26 16:25:24,2019-03-26 16:25:20,2019-03-26 16:25:25)
```

## 提取watermark

watermark的提取工作在taskManager中完成，意味着这项工作是并行进行的的，而watermark是一个全局的概念，就是一个整个Flink作业之后一个warkermark。

### AssignerWithPeriodicWatermarks

定时提取watermark，这种方式会定时提取更新wartermark。

```java
//默认200ms
public void setStreamTimeCharacteristic(TimeCharacteristic characteristic) {
    this.timeCharacteristic = Preconditions.checkNotNull(characteristic);
    if (characteristic == TimeCharacteristic.ProcessingTime) {
        getConfig().setAutoWatermarkInterval(0);
    } else {
        getConfig().setAutoWatermarkInterval(200);
    }
}
```

### AssignerWithPunctuatedWatermarks

伴随event的到来就提取watermark，就是每一个event到来的时候，就会提取一次Watermark。

这样的方式当然设置watermark更为精准，但是当数据量大的时候，频繁的更新wartermark会比较影响性能。

通常情况下采用定时提取就足够了。

## 使用

### 设置数据流时间特征

```java
//设置为事件时间
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
```

默认为`TimeCharacteristic.ProcessingTime`

使用 `EventTime` 时, 默认 `watermark` 更新每隔 200ms

入口文件

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment

//便于测试，并行度设置为1
env.setParallelism(1)

//env.getConfig.setAutoWatermarkInterval(9000)

//设置为事件时间
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

//设置source 本地socket
val text: DataStream[String] = env.socketTextStream("localhost", 9000)


val lateText = new OutputTag[(String, String, Long, Long)]("late_data")

val value = text.filter(new MyFilterNullOrWhitespace)
	.flatMap(new MyFlatMap)
	.assignTimestampsAndWatermarks(new MyWaterMark)
	.map(x => (x.name, x.datetime, x.timestamp, 1L))
	.keyBy(_._1)
	.window(TumblingEventTimeWindows.of(Time.seconds(5)))
	.sideOutputLateData(lateText)
	//.sum(2)
	.apply(new MyWindow)
	//.window(TumblingEventTimeWindows.of(Time.seconds(3)))
	//.apply(new MyWindow)
value.getSideOutput(lateText).map(x => {
	"延迟数据|name:" + x._1 + "|datetime:" + x._2
}).print()

value.print()

env.execute("watermark test")
```

```scala
class MyWaterMark extends AssignerWithPeriodicWatermarks[EventObj] {

  val maxOutOfOrderness = 10000L // 3.0 seconds
  var currentMaxTimestamp = 0L

  val sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")

  /**
    * 用于生成新的水位线，新的水位线只有大于当前水位线才是有效的
    *
    * 通过生成水印的间隔（每n毫秒）定义 ExecutionConfig.setAutoWatermarkInterval(...)。
    * getCurrentWatermark()每次调用分配器的方法，如果返回的水印非空并且大于先前的水印，则将发出新的水印。
    *
    * @return
    */
  override def getCurrentWatermark: Watermark = {
    new Watermark(this.currentMaxTimestamp - this.maxOutOfOrderness)
  }

  /**
    * 用于从消息中提取事件时间
    *
    * @param element                  EventObj
    * @param previousElementTimestamp Long
    * @return
    */
  override def extractTimestamp(element: EventObj, previousElementTimestamp: Long): Long = {

    currentMaxTimestamp = Math.max(element.timestamp, currentMaxTimestamp)

    val id = Thread.currentThread().getId
    println("currentThreadId:" + id + ",key:" + element.name + ",eventTime:[" + element.datetime + "],currentMaxTimestamp:[" + sdf.format(currentMaxTimestamp) + "],watermark:[" + sdf.format(getCurrentWatermark().getTimestamp) + "]")

    element.timestamp
  }
}	
```

### 代码详解

1. 设置为事件时间
2. 接受本地socket数据
3. 抽取timestamp生成watermark，打印（线程id,key,eventTime,currentMaxTimestamp,watermark）
4. event time每隔3秒触发一次窗口，打印（key,窗口内元素个数，窗口内最早元素的时间，窗口内最晚元素的时间，窗口自身开始时间，窗口自身结束时间）

### 试验

#### 第一次

数据

```json
{"datetime":"2019-03-26 16:25:24","name":"zhangsan"}
```

输出

```json
|currentThreadId:38,key:zhangsan,eventTime:[2019-03-26 16:25:24],currentMaxTimestamp:[2019-03-26 16:25:24],watermark:[2019-03-26 16:25:14]
```

汇总

| Key      | EventTime           | currentMaxTimestamp | Watermark           |
| -------- | ------------------- | ------------------- | ------------------- |
| zhangsan | 2019-03-26 16:25:24 | 2019-03-26 16:25:24 | 2019-03-26 16:25:14 |



#### 第二次

数据

```json
{"datetime":"2019-03-26 16:25:27","name":"zhangsan"}
```

输出

```css
currentThreadId:38,key:zhangsan,eventTime:[2019-03-26 16:25:27],currentMaxTimestamp:[2019-03-26 16:25:27],watermark:[2019-03-26 16:25:17]
```

汇总

| Key      | EventTime           | currentMaxTimestamp | Watermark           |
| -------- | ------------------- | ------------------- | ------------------- |
| zhangsan | 2019-03-26 16:25:24 | 2019-03-26 16:25:24 | 2019-03-26 16:25:14 |
| zhangsan | 2019-03-26 16:25:27 | 2019-03-26 16:25:27 | 2019-03-26 16:25:17 |

随着EventTime的升高，Watermark升高。



#### 第三次

数据

```json
{"datetime":"2019-03-26 16:25:34","name":"zhangsan"}
```

输出

```css
currentThreadId:38,key:zhangsan,eventTime:[2019-03-26 16:25:34],currentMaxTimestamp:[2019-03-26 16:25:34],watermark:[2019-03-26 16:25:24]
```

汇总

| Key      | EventTime               | currentMaxTimestamp | Watermark               |
| -------- | ----------------------- | ------------------- | ----------------------- |
| zhangsan | **2019-03-26 16:25:24** | 2019-03-26 16:25:24 | 2019-03-26 16:25:14     |
| zhangsan | 2019-03-26 16:25:27     | 2019-03-26 16:25:27 | 2019-03-26 16:25:17     |
| zhangsan | 2019-03-26 16:25:34     | 2019-03-26 16:25:34 | **2019-03-26 16:25:24** |

到这里，window仍然没有被触发，此时 watermark 的时间已经等于了第一条数据的 EventTime了。

#### 第四次

数据

```json
{"datetime":"2019-03-26 16:25:35","name":"zhangsan"}
```

![](https://upload-images.jianshu.io/upload_images/3205503-82f99e867b4fff81.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/423)

输出

```css
currentThreadId:38,key:zhangsan,eventTime:[2019-03-26 16:25:35],currentMaxTimestamp:[2019-03-26 16:25:35],watermark:[2019-03-26 16:25:25]
(zhangsan,1,2019-03-26 16:25:24,2019-03-26 16:25:24,2019-03-26 16:25:20,2019-03-26 16:25:25)
```

![](https://upload-images.jianshu.io/upload_images/3205503-0929aa523c106b68.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/979)

汇总

| Key      | EventTime               | currentMaxTimestamp | Watermark               | WindowStartTime          | WindowEndTime            |
| -------- | ----------------------- | ------------------- | ----------------------- | ------------------------ | ------------------------ |
| zhangsan | **2019-03-26 16:25:24** | 2019-03-26 16:25:24 | 2019-03-26 16:25:14     |                          |                          |
| zhangsan | 2019-03-26 16:25:27     | 2019-03-26 16:25:27 | 2019-03-26 16:25:17     |                          |                          |
| zhangsan | 2019-03-26 16:25:34     | 2019-03-26 16:25:34 | 2019-03-26 16:25:24     |                          |                          |
| zhangsan | 2019-03-26 16:25:35     | 2019-03-26 16:25:35 | **2019-03-26 16:25:25** | **[2019-03-26 16:25:20** | **2019-03-26 16:25:25)** |

直接证明了window的设定无关数据本身，而是系统定义好了的。
 输入的数据中，根据自身的Event Time，将数据划分到不同的window中，如果window中有数据，则当watermark时间>=Event Time时，就符合了window触发的条件了，最终决定window触发，还是由数据本身的Event Time所属的window中的window_end_time决定。

当最后一条数据16:25:35到达是，Watermark提升到16:25:25，此时窗口16:25:20~16:25:25中有数据，Window被触发。

#### 第五次

数据

```json
{"datetime":"2019-03-26 16:25:37","name":"zhangsan"}
```

输出

```css
currentThreadId:38,key:zhangsan,eventTime:[2019-03-26 16:25:37],currentMaxTimestamp:[2019-03-26 16:25:37],watermark:[2019-03-26 16:25:27]
```

汇总

| Key      | EventTime               | currentMaxTimestamp | Watermark               | WindowStartTime          | WindowEndTime            |
| -------- | ----------------------- | ------------------- | ----------------------- | ------------------------ | ------------------------ |
| zhangsan | **2019-03-26 16:25:24** | 2019-03-26 16:25:24 | 2019-03-26 16:25:14     |                          |                          |
| zhangsan | 2019-03-26 16:25:27     | 2019-03-26 16:25:27 | 2019-03-26 16:25:17     |                          |                          |
| zhangsan | 2019-03-26 16:25:34     | 2019-03-26 16:25:34 | 2019-03-26 16:25:24     |                          |                          |
| zhangsan | 2019-03-26 16:25:35     | 2019-03-26 16:25:35 | **2019-03-26 16:25:25** | **[2019-03-26 16:25:20** | **2019-03-26 16:25:25)** |
| zhangsan | 2019-03-26 16:25:37     | 2019-03-26 16:25:37 | 2019-03-26 16:25:27     |                          |                          |

此时，watermark时间虽然已经达到了第二条数据的时间，但是由于其没有达到第二条数据所在window的结束时间，所以window并没有被触发。

第二条数据所在的window时间是：`[2019-03-26 16:25:25,2019-03-26 16:25:30)`

#### 第六次

数据

```json
{"datetime":"2019-03-26 16:25:40","name":"zhangsan"}
```

输出

```css
currentThreadId:38,key:zhangsan,eventTime:[2019-03-26 16:25:40],currentMaxTimestamp:[2019-03-26 16:25:40],watermark:[2019-03-26 16:25:30]
(zhangsan,1,2019-03-26 16:25:27,2019-03-26 16:25:27,2019-03-26 16:25:25,2019-03-26 16:25:30)
```

汇总

| Key      | EventTime               | currentMaxTimestamp | Watermark               | WindowStartTime          | WindowEndTime            |
| -------- | ----------------------- | ------------------- | ----------------------- | ------------------------ | ------------------------ |
| zhangsan | **2019-03-26 16:25:24** | 2019-03-26 16:25:24 | 2019-03-26 16:25:14     |                          |                          |
| zhangsan | 2019-03-26 16:25:27     | 2019-03-26 16:25:27 | 2019-03-26 16:25:17     |                          |                          |
| zhangsan | 2019-03-26 16:25:34     | 2019-03-26 16:25:34 | 2019-03-26 16:25:24     |                          |                          |
| zhangsan | 2019-03-26 16:25:35     | 2019-03-26 16:25:35 | **2019-03-26 16:25:25** | **[2019-03-26 16:25:20** | **2019-03-26 16:25:25)** |
| zhangsan | 2019-03-26 16:25:37     | 2019-03-26 16:25:37 | 2019-03-26 16:25:27     |                          |                          |
| zhangsan | 2019-03-26 16:25:40     | 2019-03-26 16:25:40 | **2019-03-26 16:25:30** | **[2019-03-26 16:25:25** | **2019-03-26 16:25:30)** |

### 结论

window的触发要符合以下几个条件：

1. watermark时间 >= window_end_time
2. 在[window_start_time,window_end_time)中有数据存在

同时满足了以上2个条件，window才会触发。

> so, 这里有个问题, 对于半夜活跃用户低的情况下, 两条数据间的间隔时长特别长, 这时 window 会无线等待下去么?
>
> 了解 FLINK idle

 watermark是一个全局的值，不是某一个key下的值，所以即使不是同一个key的数据，其warmark也会增加.

## 迟到数据的处理

侧输出（side output）是Flink的分流机制。迟到数据本身可以当做特殊的流，我们通过调用WindowedStream.sideOutputLateData()方法将迟到数据发送到指定OutputTag的侧输出流里去，再进行下一步处理（比如存到外部存储或[消息队列](https://cloud.tencent.com/product/cmq?from=10680)）。代码如下。

```javascript
      // 侧输出的OutputTag
      OutputTag<UserActionRecord> lateOutputTag = new OutputTag<>("late_data_output_tag");

      sourceStream.assignTimestampsAndWatermarks(
        new BoundedOutOfOrdernessTimestampExtractor<UserActionRecord>(Time.seconds(30)) {
          private static final long serialVersionUID = 1L;
          @Override
          public long extractTimestamp(UserActionRecord record) {
            return record.getTimestamp();
          }
        }
      )
      .keyBy("platform")
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .allowedLateness(Time.seconds(30))
      .sideOutputLateData(lateOutputTag)   // 侧输出
      .aggregate(new ViewAggregateFunc(), new ViewSumWindowFunc())
      // ......

      // 获取迟到数据并写入对应Sink
      stream.getSideOutput(lateOutputTag).addSink(lateDataSink);
```



## 多并行度

![](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/parallel_kafka_watermarks.svg)



上面的示例只有一个并行度，在有多个并行度的情况下，就会有多个流产生水印，窗口触发时该采用哪个水印呢？**答案是所有流入水印中时间戳最小的那个。**

如果一个算子中所有流入水印中时间戳最小的那个都已经达到或超过了窗口的结束时间，那么所有流的数据肯定已经全部收齐，就可以安全地触发窗口计算了。

那么这里说的对齐(aligned)指的是什么?

### isWatermarkAligned

在Watermark相关源码中经常可以看到Watermark对齐的判断，如果各个子任务中的Watermark都对齐的话，那是最理想的情况。然而实际生产中，总会很多意外。这里有两个极端的例子。

#### 一条数据流长时间没有数据，其他流数据正常。

出现这种情况的原因有很多，比如数据本身是波峰状，数据被上游算子过滤掉等。刚刚提到，下游算子的Watermark取决于多个上游算子Watermark的最小值，那么如果一条数据流长时间没有数据，岂不是会造成Watermark一直停留在原地不动的情况？

![](http://www.liaojiayi.com/assets/flink-idletimeout.png)

当然不会，针对这种情况，Flink在数据流中实现了一个Idle的概念。用户首先需要设置`timeout(IdleTimeout)`，这个timeout表示数据源在多久没有收到数据后，数据流要判断为Idle，下游Window算子在接收到Idle状态后，将不再使用这个Channel之前留下的Watermark，而用其他Active Channel发送的Watermark数据来计算。

如上图所示，Source(1)在接收到数据后，会触发生成一个timeout之后调用的callback，如果在timeout时间长度中，没有再接收新的数据，便会向下游发送Idle状态，将自己标识为Idle。之后下游的 window 就会使用 source(2, 3) 中小的 watermark 来触发计算。

#### 数据倾斜。

假设单条数据流倾斜，那么该数据流中处理的数据所带的时间戳，是远低于其他数据流中事件的时间戳。

![](http://www.liaojiayi.com/assets/flink-dataskew.png)

如图所示，假设Watermark设置为收到的时间戳-1，那么Window的Watermark始终都保持在0，这会导致Window存储大量的窗口，并且窗口状态无法释放，极有可能出现OOM。这个问题目前没有好的解决办法，需要具体情况具体分析了。

### State Of WatermarkOperator

```
stream.assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarks[PatternWrapper] {
    override def getCurrentWatermark: Watermark = new Watermark(System.currentTimeMillis() - 15000)

    override def extractTimestamp(element: PatternWrapper, previousElementTimestamp: Long): Long = {
      System.currentTimeMillis()
    }
})
```

我们都知道，用户可以通过显式调用这种方法来实现Watermark，那么对于这种方法，本质上是生成了一个WatermarkOperator，并且在Operator中计算Watermark传递给下游的算子。  

此时问题来了，刚刚提到Watermark是由已有数据产生的，那么在程序刚恢复还没有发送数据的时候怎么办？当然不能选择当前时间，这样的话，稍晚一点的事件就被过滤掉了。   

这就引出一个问题，Watermark是不会存储在Checkpoint中的，也就是说WatermarkOperator是一个无状态算子，那么程序启动时，Flink自身的逻辑是初始化Watermark为Long.MIN_VALUE。由于我们架构设计的原因，对脏数据非常敏感，不允许发送过于久的历史数据，于是我们将WatermarkOperator的算子改成了有状态的算子，其中为了兼容并行度scale的情况，我们将Watermark设置为所有数据流中Watermark的最小值。具体的JIRA可以看[FLINK-5601](https://issues.apache.org/jira/browse/FLINK-5601)中提出的一些方案和代码改动。

## 总结

### Flink如何处理乱序？

watermark+window机制。window中可以对input进行按照Event Time排序，使得完全按照Event Time发生的顺序去处理数据，以达到处理乱序数据的目的。

### Flink何时触发window？

- 对于late element太多的数据而言
  1. Event Time < watermark时间

- 对于out-of-order以及正常的数据而言
  1. watermark时间 >= window_end_time
  2. 在[window_start_time,window_end_time)中有数据存在

### Flink应该如何设置最大乱序时间？

结合自己的业务以及数据情况去设置。

![img](https:////upload-images.jianshu.io/upload_images/3205503-9903be8fdd296d1b.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

## 参考

[Flink WaterMark（水位线）分布式执行理解](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fu013560925%2Farticle%2Fdetails%2F82499612)

[Flink流计算编程--watermark（水位线）简介](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Flmalds%2Farticle%2Fdetails%2F52704170)

[The Dataflow Model](https://links.jianshu.com/go?to=https%3A%2F%2Fstatic.googleusercontent.com%2Fmedia%2Fresearch.google.com%2Fen%2F%2Fpubs%2Farchive%2F43864.pdf)

