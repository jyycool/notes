# 【Kafka】Lag

## 一、前言

消息堆积是消息中间件的一大特色，消息中间件的流量削峰、冗余存储等功能正是得益于消息中间件的消息堆积能力。然而消息堆积其实是一把亦正亦邪的双刃剑，如果应用场合不恰当反而会对上下游的业务造成不必要的麻烦，比如消息堆积势必会影响上下游整个调用链的时效性，有些中间件如 RabbitMQ 在发生消息堆积时在某些情况下还会影响自身的性能。

对于 Kafka 而言，虽然消息堆积不会对其自身性能带来多大的困扰，但难免不会影响上下游的业务，堆积过多有可能会造成磁盘爆满，或者触发日志清除策略而造成消息丢失的情况。如何利用好消息堆积这把双刃剑，监控是最为关键的一步。

## 正文

消息堆积是消费滞后 (Lag) 的一种表现形式，消息中间件服务端中所留存的消息与消费掉的消息之间的差值即为消息堆积量，也称之为消费滞后(Lag)量。对于 Kafka 而言，消息被发送至 Topic中，而 Topic 又分成了多个分区(Partition)，每一个 Partition 都有一个预写式的日志文件，虽然 Partition 可以继续细分为若干个段文件(Segment)，但是对于上层应用来说可以将Partition 看成最小的存储单元(一个由多个Segment文件拼接的“巨型文件”)。每个 Partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到 Partition 中。我们来看下图，其就是 Partition 的一个真实写照：

[![img](http://image.honeypps.com/images/papers/2018/112.png)](http://image.honeypps.com/images/papers/2018/112.png)

上图中有四个概念：

1. **LogStartOffset**

   表示一个 Partition 的起始位移，初始为0，虽然消息的增加以及日志清除策略的影响，这个值会阶段性的增大。

2. **ConsumerOffset**

   消费位移，表示 Partition 的某个消费者消费到的位移位置。

3. **HighWatermark**

   简称 HW，代表消费端所能“观察”到的 Partition 的最高日志位移，HW 大于等于ConsumerOffset的值。

4. **LogEndOffset**

   简称 LEO, 代表 Partition 的最高日志位移，其值对消费者不可见。比如在ISR（In-Sync-Replicas）副本数等于 3 的情况下（如下图所示），消息发送到 Leader A 之后会更新 LEO 的值，Follower B 和 Follower C 也会实时拉取 Leader A 中的消息来更新自己，HW 就表示 A、B、C 三者同时达到的日志位移，也就是A、B、C 三者中 LEO 最小的那个值。由于 B、C 拉取 A 消息之间延时问题，所以 HW 必然不会一直与 Leader 的 LEO 相等，即LEO>=HW。

[![img](http://image.honeypps.com/images/papers/2018/113.png)](http://image.honeypps.com/images/papers/2018/113.png)

要计算 Kafka 中某个消费者的滞后量很简单，首先看看其消费了几个Topic，然后针对每个Topic来计算其中每个 Partition 的 Lag，每个 Partition 的 Lag 计算就显得非常的简单了，参考下图：

由图可知消费 Lag=HW - ConsumerOffset。对于这里大家有可能有个误区，就是认为Lag应该是LEO 与 ConsumerOffset 之间的差值，笔者在这之前也犯过这样的错误认知，详细可以参考《[如何使用JMX监控Kafka](https://blog.csdn.net/u013256816/article/details/53524884)》。LEO是对消费者不可见的，既然不可见何来消费滞后一说。

那么这里就引入了一个新的问题，HW 和 ConsumerOffset 的值如何获取呢？

[![img](http://image.honeypps.com/images/papers/2018/114.png)](http://image.honeypps.com/images/papers/2018/114.png)

首先来说说 ConsumerOffset，Kafka 中有两处可以存储，一个是 Zookeeper，而另一个是”__consumer_offsets 这个内部 topic 中，前者是 0.8.x 版本中的使用方式，但是随着版本的迭代更新，现在越来越趋向于后者。就拿 1.0.0 版本来说，虽然默认是存储在”\_\_consumer_offsets”中，但是保不齐用于就将其存储在了 Zookeeper 中了。这个问题倒也不难解决，针对两种方式都去拉取，然后哪个有值的取哪个。不过这里还有一个问题，对于消费位移来说，其一般不会实时的更新，而更多的是定时更新，这样可以提高整体的性能。那么这个定时的时间间隔就是 ConsumerOffset 的误差区间之一。

再来说说 HW，其也是 Kafka 中 Partition 的一个状态。有可能你会察觉到在 Kafka 的 JMX 中可以看到“kafka.log:type=Log,name=LogEndOffset,topic=[topic_name],partition=[partition_num]”这样一个属性，但是这个值不是LEO而是HW。

那么怎样正确的计算消费的 Lag 呢？对Kafka熟悉的同学可能会想到Kafka中自带的kafka-consumer_groups.sh 脚本中就有 Lag 的信息，示例如下：

```sh
[root@node2 kafka_2.12-1.0.0]# bin/kafka-consumer-groups.sh --describe --bootstrap-server localhost:9092 --group CONSUMER_GROUP_ID
TOPIC                PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID                                       HOST                   CLIENT-ID
topic-test1          0          1648            1648            0          CLIENT_ID-e2d41f8d-dbd2-4f0e-9239-efacb55c6261    /192.168.92.1          CLIENT_ID
topic-test1          1          1648            1648            0          CLIENT_ID-e2d41f8d-dbd2-4f0e-9239-efacb55c6261    /192.168.92.1          CLIENT_ID
topic-test1          2          1648            1648            0          CLIENT_ID-e2d41f8d-dbd2-4f0e-9239-efacb55c6261    /192.168.92.1          CLIENT_ID
topic-test1          3          1648            1648            0          CLIENT_ID-e2d41f8d-dbd2-4f0e-9239-efacb55c6261    /192.168.92.1          CLIENT_ID
```

我们深究一下kafka-consumer_groups.sh脚本，发现只有一句代码：

```sh
exec $(dirname $0)/kafka-run-class.sh kafka.admin.ConsumerGroupCommand "$@"
```

其含义就是执行 kafka.admin.ConsumerGroupCommand 而已。进一步深究，在ConsumerGroupCommand 内部抓住了 2 句关键代码：

```sh
val consumerGroupService = new KafkaConsumerGroupService(opts)
val (state, assignments) = consumerGroupService.describeGroup()
```

代码详解：consumerGroupService 的类型是 ConsumerGroupServicesealed trait类型），而 KafkaConsumerGroupService 只是 ConsumerGroupService 的一种实现，还有一种实现是 ZkConsumerGroupService，分别对应新版的消费方式（消费位移存储在__consumer_offsets中）和旧版的消费方式（消费位移存储在 zk 中），详细计算步骤参考下一段落的内容。opt 参数是指 “–describe –bootstrap-server localhost:9092 –group CONSUMER_GROUP_ID” 等参数。第 2 句代码是调用 describeGroup()方 法来获取具体的信息，即二元组中的 assignments，这个 assignments 中保存了上面打印信息中的所有内容。

> **Scala小知识：**
> 在 Scala 中 trait（特征）相当于 Java 的接口，实际上它比接口更大强大。与 Java 中的接口不同的是，它还可以定义属性和方法的实现（JDK8起的接口默认方法）。一般情况下Scala 中的类只能继承单一父类，但是如果是 trait 的话就可以继承多个，从结果来看是实现了多重继承。被 sealed 声明的trait仅能被同一文件的类继承。

ZkConsumerGroupService 中计算消费 lag 的步骤如下：

1. 通过 zk 获取一些基本信息，对应上面打印信息中的：TOPIC、PARTITION、CONSUMER-ID等，不过不会有HOST和CLIENT-ID。
2. 通过 OffsetFetchRequest 请求获取消费位移（offset），如果获取失败则在通过zk获取。
3. 通过 OffsetReuqest 请求获取分区的 LogEndOffset（简称为LEO，可见的LEO）。
4. 计算 LogEndOffset 与消费位移的差值来获取 lag。

KafkaConsumerGroupService 中计算消费lag的步骤如下：

1. 通过 DescibeGroupsRequest 请求获取一些基本信息，不仅包括 TOPIC、PARTITION、CONSUMER-ID，还有 HOST 和 CLIENT-ID。其实还有通过
   FindCoordinatorRequest 请求来获取 coordinator 信息，如果不了解 coordinator 在这里也没影响。
2. 通过 OffsetFetchRequest请 求获取消费位移。
3. 通过 OffsetReuqest 请求获取分区的 LogEndOffset（简称为LEO）。
4. 计算 LogEndOffset 与消费位移的差值来获取 lag。

可以看到 KafkaConsumerGroupService 与 ZkConsumerGroupService 的计算 Lag 的方式都差不多，但是 KafkaConsumerGroupService 能获取更多消费详情，并且ZkConsumerGroupService 也被标注为@Deprecated的了，后面内容都针对KafkaConsumerGroupService 来做说明。既然 Kafka 已经为我们提供了线程的方法来获取Lag，那么我们有何必再重复造轮子，这里笔者写了一个调用的 KafkaConsumerGroupService 的示例（KafkaConsumerGroupService 是使用Scala语言编写的，在 Java的程序里使用类似scala.collection.Seq 这样的全名称以防止混淆）：

```scala
String[] agrs = {"--describe", "--bootstrap-server", brokers, "--group", groupId};
ConsumerGroupCommand.ConsumerGroupCommandOptions opts =
new ConsumerGroupCommand.ConsumerGroupCommandOptions(agrs);
ConsumerGroupCommand.KafkaConsumerGroupService kafkaConsumerGroupService =
new ConsumerGroupCommand.KafkaConsumerGroupService(opts);
scala.Tuple2<scala.Option<String>, scala.Option<scala.collection.Seq<ConsumerGroupCommand
.PartitionAssignmentState>>> res = kafkaConsumerGroupService.describeGroup();
scala.collection.Seq<ConsumerGroupCommand.PartitionAssignmentState> pasSeq = res._2.get();
scala.collection.Iterator<ConsumerGroupCommand.PartitionAssignmentState> iterable = pasSeq.iterator();
while (iterable.hasNext()) {
  ConsumerGroupCommand.PartitionAssignmentState pas = iterable.next();
  System.out.println(String.format("\n%-30s %-10s %-15s %-15s %-10s %-50s%-30s %s",
                                   pas.topic().get(), pas.partition().get(), pas.offset().get(),
                                   pas.logEndOffset().get(), pas.lag().get(), pas.consumerId().get(),
                                   pas.host().get(), pas.clientId().get()));
}
```

在使用时，你可以封装一下这段代码然后返回一个类似List<ConsumerGroupCommand.PartitionAssignmentState> 的东西给上层业务代码做进一步的使用。ConsumerGroupCommand.PartitionAssignmentState的代码如下：

```scala
case class PartitionAssignmentState(
  group: String, 
  coordinator: Option[Node], 
  topic: Option[String],
  partition: Option[Int], 
  offset: Option[Long], 
  lag: Option[Long],
  consumerId: Option[String], 
  host: Option[String],
  clientId: Option[String], 
  logEndOffset: Option[Long]
)
```

> **Scala小知识：**
> 对于 case class, 在这里你可以简单的把它看成是一个JavaBean，但是它远比JavaBean强大，比如它会自动生成equals、hashCode、toString、copy、伴生对象、apply、unapply等等东西。在 scala 中，对保护（Protected）成员的访问比 java 更严格一些。因为它只允许保护成员在定义了该成员的的类的子类中被访问。而在java中，用protected关键字修饰的成员，除了定义了该成员的类的子类可以访问，同一个包里的其他类也可以进行访问。Scala中，如果没有指定任何的修饰符，则默认为 public。这样的成员在任何地方都可以被访问。

如果你正在试着运行上面一段程序，你会发现编译失败，报错：cannot access ‘kafka.admin.ConsumerGroupCommand.PartitionAssignmentState’ in ‘kafka.admin.ConsumerGroupCommand‘。这时候需要将所引入的 kafka.core 包中的kafka.admin.ConsumerGroupCommand 中的 PartitionAssignmentState 类前面的protected 修饰符去掉才能编译通过。

现实情况下你可能并不想变更 kafka-core 的代码然后再重新打包，而是寻求直接能够调用的东西，至于到底怎么实现将会在下一篇文章中介绍，如果你比较猴急，可以先预览一下代码的实现，具体参见：https://github.com/hiddenzzh/kafka/blob/master/src/main/java/com/hidden/custom/kafka/admin/KafkaConsumerGroupCustomService.java。详细的逻辑解析敬请期待….

