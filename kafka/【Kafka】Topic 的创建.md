# 【Kafka】Topic 的创建

在使用 kafka 发送消息和消费消息之前，必须先要创建 topic，在 kafka 中创建 topic 的方式有以下2种：

1. 如果 kafka broker 中的 config/server.properties 配置文件中配置了auto.create.topics.enable 参数为 true（默认值就是true），那么当生产者向一个尚未创建的 topic 发送消息时，会自动创建一个 num.partitions（默认值为1）个分区和default.replication.factor（默认值为1）个副本的对应topic。不过我们一般不建议将auto.create.topics.enable 参数设置为 true，因为这个参数会影响 topic 的管理与维护。
2. 通过 kafka 提供的 kafka-topics.sh 脚本来创建，并且我们也建议通过这种方式（或者相关的变种方式）来创建 topic。



举个demo：通过 kafka-topics.sh 脚本来创建一个名为 topic-test1 并且副本数为 2、分区数为 4的 topic。（如无特殊说明，本文所述都是基于1.0.0版本。）

```sh
bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1 --replication-factor 2 --partitions 4
```

打开kafka-topics.sh脚本一探究竟，其内容只有一行，具体如下:

```sh
exec $(dirname $0)/kafka-run-class.sh kafka.admin.TopicCommand "$@"
```

这个脚本的主要作用就是运行 kafka.admin.TopicCommand。在 main 方法中判断参数列表中是否包含有”create“，如果有，那么就实施创建 topic 的任务。创建 topic 时除了需要zookeeper 的地址参数外，还需要指定 topic 的名称、副本因子 replication-factor 以及分区个数 partitions 等必选参数 ，还可以包括 disable-rack-aware、config、if-not-exists 等可选参数。

真正的创建过程是由 createTopic 这个方法中执行的，这个方法具体内容如下：

```java
def createTopic(zkUtils: ZkUtils, opts: TopicCommandOptions) {
  val topic = opts.options.valueOf(opts.topicOpt)//获取topic参数所对应的值，也就是Demo中的topic名称——topic-test
    val configs = parseTopicConfigsToBeAdded(opts)//将参数解析成Properties参数，config所指定的参数集
    val ifNotExists = opts.options.has(opts.ifNotExistsOpt)//对应if-not-exists
    if (Topic.hasCollisionChars(topic))
      println("WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.")
      try {
        if (opts.options.has(opts.replicaAssignmentOpt)) {//检测是否有replica-assignment参数
          val assignment = parseReplicaAssignment(
            opts.options.valueOf(opts.replicaAssignmentOpt))
            AdminUtils.createOrUpdateTopicPartitionAssignmentPathInZK(
            zkUtils, topic, assignment, configs, update = false)
        } else {
          CommandLineUtils.checkRequiredArgs(
            opts.parser, 
            opts.options, 
            opts.partitionsOpt, 
            opts.replicationFactorOpt
          )
            val partitions = opts.options
            .valueOf(opts.partitionsOpt)
            .intValue
            val replicas = opts.options
            .valueOf(opts.replicationFactorOpt)
            .intValue
            val rackAwareMode = if (opts.options.has(opts.disableRackAware)) RackAwareMode.Disabled
              else RackAwareMode.Enforced
                AdminUtils.createTopic(zkUtils, 
                                       topic, 
                                       partitions, 
                                       replicas, 
                                       configs, 
                                       rackAwareMode)
              }
        println("Created topic \"%s\".".format(topic))
      } catch  {
        case e: TopicExistsException => 
          if (!ifNotExists) throw e
          }
}
```

createTopic 方法中首先获取 topic 的名称，config 参数集以及判断是否有 if-not-exists 参数。config 参数集可以用来设置 topic 级别的配置以覆盖默认配置。如果创建的topic 再现有的集群中存在，那么会报出异常：TopicExistsException，如果创建的时候带了if-not-exists 参数，那么发现 topic 冲突的时候可以不做任何处理；如果 topic 不冲突，那么和不带 if-not-exists 参数的行为一样正常 topic，下面 demo 中接着上面的 demo 继续创建同一个 topic，不带有 if-not-exists 参数和带有 if-not-exists 参数的效果如下：

```sh
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1 --replication-factor 2 --partitions 4
Error while executing topic command : Topic 'topic-test1' already exists.
[2018-01-30 17:52:32,425] ERROR org.apache.kafka.common.errors.TopicExistsException: Topic 'topic-test1' already exists.
 (kafka.admin.TopicCommand$)
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1 --replication-factor 2 --partitions 4 --if-not-exists
[root@node1 kafka_2.12-1.0.0]#
```

接下去还会进一步检测 topic 名称中是否包含有“.”或者“_”字符的，这一个步骤在`AdminUtils#createOrUpdateTopicPartitionAssignmentPathInZK()`中调用validateCreateOrUpdateTopic() 方法实现的。为什么要检测这两个字符呢？因为在 Kafka 的内部做埋点时会根据 topic 的名称来命名 metrics 的名称，并且会将句点号“.”改成下划线”_”。假设遇到一个 topic 的名称为“topic.1_2”，还有一个topic的名称为“topic_1.2”，那么最后的metrics的名称都为“topic_1_2”，所以就会发生名称冲突。举例如下，首先创建一个以”topic.1_2”名称的topic，提示 WARNING 警告，之后在创建一个“topic.1_2”时发生InvalidTopicException 异常。

```sh
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic.1_2 --replication-factor 2 --partitions 4
WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Created topic "topic.1_2".
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic_1.2 --replication-factor 2 --partitions 4
WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Error while executing topic command : Topic 'topic_1.2' collides with existing topics: topic.1_2
[2018-01-31 20:27:28,449] ERROR org.apache.kafka.common.errors.InvalidTopicException: Topic 'topic_1.2' collides with existing topics: topic.1_2
 (kafka.admin.TopicCommand$)
```

> 补充：topic的命名同样不推荐（虽然可以这样做）使用双下划线“\_\__”开头，因为以双下划线开头的 topic 一般看作是 kafka 的内部 topic，比如\_\_consumer_offsets和__transaction_state。topic 的名称必须由大小写字母、数字、“.”、“-”、“_”组成，不能为空、不能为“.”、不能为“..”，且长度不能超过249。

接下去 createTopic 方法的主体就分为两个部分了，如果检测出有 replica-assignment 参数，那么就是制定了副本的分配方式。这个在前面都没有提及，那么这个又指的是什么呢？如果包含了replica-assignment 参数，那么就可以通过指定的分区副本分配方式创建 topic，这个有点绕，不妨再来一个 demo 开拓下思路：

```sh
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1
Topic:topic-test1	PartitionCount:4	ReplicationFactor:2	Configs:
	Topic: topic-test	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test	Partition: 1	Leader: 1	Replicas: 1,0	Isr: 1,0
	Topic: topic-test	Partition: 2	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test	Partition: 3	Leader: 1	Replicas: 1,0	Isr: 1,0
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1 --replication-factor 2 --partitions 4 --if-not-exists
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1
Topic:topic-test	PartitionCount:4	ReplicationFactor:2	Configs:
	Topic: topic-test	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test	Partition: 1	Leader: 1	Replicas: 1,0	Isr: 1,0
	Topic: topic-test	Partition: 2	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test	Partition: 3	Leader: 1	Replicas: 1,0	Isr: 1,0
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test2 --replica-assignment 0:1,1:0,0:1,1:0
Created topic "topic-test2".
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test2
Topic:topic-test2	PartitionCount:4	ReplicationFactor:2	Configs:
	Topic: topic-test2	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test2	Partition: 1	Leader: 1	Replicas: 1,0	Isr: 1,0
	Topic: topic-test2	Partition: 2	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test2	Partition: 3	Leader: 1	Replicas: 1,0	Isr: 1,0
```

可以看到手动指定 “–replica-assignment 0:1,1:0,0:1,1:0” 后副本的分配方式和自动分配的效果一样。createTopic方法中如果判断opts.options.has(opts.replicaAssignmentOpt) 满足条件，那么接下去的工作就是解析并验证指定的副本是否有重复、每个分区的副本个数是否相同等等。如果指定 0:0,1:1 这种（副本重复）就会报出 AdminCommandFailedException 异常。详细 demo 如下：

```sh
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replica-assignment 0:0,1:1
Error while executing topic command : Partition replica lists may not contain duplicate entries: 0
[2018-02-01 20:23:40,435] ERROR kafka.common.AdminCommandFailedException: Partition replica lists may not contain duplicate entries: 0
	at kafka.admin.TopicCommand$.$anonfun$parseReplicaAssignment$1(TopicCommand.scala:286)
	at scala.collection.immutable.Range.foreach$mVc$sp(Range.scala:156)
	at kafka.admin.TopicCommand$.parseReplicaAssignment(TopicCommand.scala:282)
	at kafka.admin.TopicCommand$.createTopic(TopicCommand.scala:102)
	at kafka.admin.TopicCommand$.main(TopicCommand.scala:63)
	at kafka.admin.TopicCommand.main(TopicCommand.scala)
 (kafka.admin.TopicCommand$)
```

如果指定0:1, 0, 1:0这种（分区副本个数不同）就会报出 AdminOperationException 异常。详细demo如下：

```sh
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replica-assignment 0:1,1,0:1,1:0
Error while executing topic command : Partition 1 has different replication factor: [I@159f197
[2018-01-31 20:37:49,136] ERROR kafka.admin.AdminOperationException: Partition 1 has different replication factor: [I@159f197
	at kafka.admin.TopicCommand$.$anonfun$parseReplicaAssignment$1(TopicCommand.scala:289)
	at scala.collection.immutable.Range.foreach$mVc$sp(Range.scala:156)
	at kafka.admin.TopicCommand$.parseReplicaAssignment(TopicCommand.scala:282)
	at kafka.admin.TopicCommand$.createTopic(TopicCommand.scala:102)
	at kafka.admin.TopicCommand$.main(TopicCommand.scala:63)
	at kafka.admin.TopicCommand.main(TopicCommand.scala)
 (kafka.admin.TopicCommand$)
```

当然，像0:1,,0:1,1:0这种企图跳过一个partition连续序号的行为也是不被允许的，详细demo如下：

```sh
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replica-assignment 0:1,,0:1,1:0
Error while executing topic command : For input string: ""
[2018-02-04 22:14:26,948] ERROR java.lang.NumberFormatException: For input string: ""
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.lang.Integer.parseInt(Integer.java:592)
	at java.lang.Integer.parseInt(Integer.java:615)
	at scala.collection.immutable.StringLike.toInt(StringLike.scala:301)
	at scala.collection.immutable.StringLike.toInt$(StringLike.scala:301)
	at scala.collection.immutable.StringOps.toInt(StringOps.scala:29)
	at kafka.admin.TopicCommand$.$anonfun$parseReplicaAssignment$2(TopicCommand.scala:283)
	......
 (kafka.admin.TopicCommand$)
```

验证之后在zookeeper中创建/brokers/topics/topic-test持久化节点，对应节点的数据就是以json格式呈现的分区分配的结果集，格式参考：{“version”:1,”partitions”:{“2”:[0,1],”1”:[1,0],”3”:[1,0],”0”:[0,1]}}。如果配置了config参数的话，同样先进行验证，如若无误就创建/config/topics/topic-test节点，并写入config对应的数据，格式参考：{“version”:1,”config”:{“max.message.bytes”:”1000013”}}。详细demo如下：

```sh
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replication-factor 2 --partitions 4 --config key=value
Error while executing topic command : Unknown topic config name: key
[2018-01-31 20:43:23,208] ERROR org.apache.kafka.common.errors.InvalidConfigurationException: Unknown topic config name: key
 (kafka.admin.TopicCommand$)
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test3 --replication-factor 2 --partitions 4 --config max.message.bytes=1000013
Created topic "topic-test3".
```

如果在创建topic的时候并没有指定replica-assignment参数，那么就需要采用kafka默认的分区副本分配策略来创建topic。主要的是以下这6行代码：



```
CommandLineUtils.checkRequiredArgs(opts.parser, opts.options, opts.partitionsOpt, opts.replicationFactorOpt)
val partitions = opts.options.valueOf(opts.partitionsOpt).intValue
val replicas = opts.options.valueOf(opts.replicationFactorOpt).intValue
val rackAwareMode = if (opts.options.has(opts.disableRackAware)) RackAwareMode.Disabled
                    else RackAwareMode.Enforced
AdminUtils.createTopic(zkUtils, topic, partitions, replicas, configs, rackAwareMode)
```

第一行的作用就是验证一下执行kafka-topics.sh时参数列表中是否包含有partitions和replication-factor这两个参数，如果没有包含则报出：Missing required argument “[partitions]”或者Missing required argument “[replication-factor]”，并给出参数的提示信息列表。

第2-5行的作用是获取paritions、replication-factor参数所对应的值以及验证是否包含disable-rack-aware这个参数。从0.10.x版本开始，kafka可以支持指定broker的机架信息，如果指定了机架信息则在副本分配时会尽可能地让分区的副本分不到不同的机架上。指定机架信息是通过kafka的配置文件config/server.properties中的broker.rack参数来配置的，比如配置当前broker所在的机架为“RACK1”：

```
broker.rack=RACK1
```

最后一行通过AdminUtils.createTopic方法来继续创建，至此代码流程又进入到下一个无底洞，不过暂时不用担心，下面是这个方法的详细内容，看上去只有几行而已：

```
def createTopic(zkUtils: ZkUtils,
                topic: String,
                partitions: Int,
                replicationFactor: Int,
                topicConfig: Properties = new Properties,
                rackAwareMode: RackAwareMode = RackAwareMode.Enforced) {
  val brokerMetadatas = getBrokerMetadatas(zkUtils, rackAwareMode)
  val replicaAssignment = AdminUtils.assignReplicasToBrokers(brokerMetadatas, partitions, replicationFactor)
  AdminUtils.createOrUpdateTopicPartitionAssignmentPathInZK(zkUtils, topic, replicaAssignment, topicConfig)
}
```

总共只有三行，最后一行还是见过的，在使用replica-assignment参数解析验证之后调用的，主要用来在/brokers/topics路径下写入相应的节点。回过头来看第一句，它是用来获取集群中每个broker的brokerId和机架信息（Option[String]类型）信息的列表，为下面的 AdminUtils.assignReplicasToBrokers()方法做分区副本分配前的准备工作。AdminUtils.assignReplicasToBrokers()首先是做一些简单的验证工作：分区个数partitions不能小于等于0、副本个数replicationFactor不能小于等于0以及副本个数replicationFactor不能大于broker的节点个数，其后的步骤就是方法最重要的两大核心：assignReplicasToBrokersRackUnaware和assignReplicasToBrokersRackAware，看这个名字也应该猜出个一二来，前者用来针对不指定机架信息的情况，而后者是用来针对指定机架信息的情况，后者更加复杂一点。

### 未指定机架的分配策略

为了能够循序渐进的说明问题，这里先来讲解assignReplicasToBrokersRackUnaware，对应的代码如下：

```
private def assignReplicasToBrokersRackUnaware(nPartitions: Int,
                                               replicationFactor: Int,
                                               brokerList: Seq[Int],
                                               fixedStartIndex: Int,
                                               startPartitionId: Int): Map[Int, Seq[Int]] = {
  val ret = mutable.Map[Int, Seq[Int]]()
  val brokerArray = brokerList.toArray
  val startIndex = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
  var currentPartitionId = math.max(0, startPartitionId)
  var nextReplicaShift = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
  for (_ <- 0 until nPartitions) {
    if (currentPartitionId > 0 && (currentPartitionId % brokerArray.length == 0))
      nextReplicaShift += 1
    val firstReplicaIndex = (currentPartitionId + startIndex) % brokerArray.length
    val replicaBuffer = mutable.ArrayBuffer(brokerArray(firstReplicaIndex))
    for (j <- 0 until replicationFactor - 1)
      replicaBuffer += brokerArray(replicaIndex(firstReplicaIndex, nextReplicaShift, j, brokerArray.length))
    ret.put(currentPartitionId, replicaBuffer)
    currentPartitionId += 1
  }
  ret
}
```

主构造函数参数列表中的fixedStartIndex和startPartitionId的值是从上游AdminUtils.assignReplicasToBrokers()方法调用传下来，都是-1，分别表示第一个副本分配的位置和起始分区编号。assignReplicasToBrokers这个方法的核心是遍历每个分区partition然后从brokerArray（brokerId的列表）中选取replicationFactor个brokerId分配给这个partition。

方法首先创建一个可变的Map用来存放本方法将要返回的结果，即分区partition和分配副本的映射关系。由于fixedStartIndex为-1，所以startIndex是一个随机数，用来计算一个起始分配的brokerId，同时由于startPartitionId为-1，所以currentPartitionId的值为0，可见默认创建topic时总是从编号为0的分区依次轮询进行分配。nextReplicaShift表示下一次副本分配相对于前一次分配的位移量，这个字面上理解有点绕，不如举个例子：假设集群中有3个broker节点，即代码中的brokerArray，创建某topic有3个副本和6个分区，那么首先从partitionId（partition的编号）为0的分区开始进行分配，假设第一次计算（由rand.nextInt(brokerArray.length)随机）到nextReplicaShift为1，第一次随机到的startIndex为2，那么partitionId为0的第一个副本的位置（这里指的是brokerArray的数组下标）firstReplicaIndex = (currentPartitionId + startIndex) % brokerArray.length = （0+2）%3 = 2，第二个副本的位置为replicaIndex(firstReplicaIndex, nextReplicaShift, j, brokerArray.length) = replicaIndex(2, nextReplicaShift+1,0, 3)=？，这里引入了一个新的方法replicaIndex，不过这个方法很简单，具体如下：

```
private def replicaIndex(firstReplicaIndex: Int, secondReplicaShift: Int, replicaIndex: Int, nBrokers: Int): Int = {
  val shift = 1 + (secondReplicaShift + replicaIndex) % (nBrokers - 1)
  (firstReplicaIndex + shift) % nBrokers
}
```

继续计算 replicaIndex(2, nextReplicaShift+1,0, 3) = replicaIndex(2, 2,0, 3)= (2+(1+(2+0)%(3-1)))%3=0。继续计算下一个副本的位置replicaIndex(2, 2,1, 3) = (2+(1+(2+1)%(3-1)))%3=1。所以partitionId为0的副本分配位置列表为[2,0,1]，如果brokerArray正好是从0开始编号，也正好是顺序不间断的，即brokerArray为[0,1,2]的话，那么当前partitionId为0的副本分配策略为[2,0,1]。如果brokerId不是从零开始，也不是顺序的（有可能之前集群的其中broker几个下线了），最终的brokerArray为[2,5,8]，那么partitionId为0的分区的副本分配策略为[8,2,5]。为了便于说明问题，可以简单的假设brokerArray就是[0,1,2]。

同样计算下一个分区，即partitionId为1的副本分配策略。此时nextReplicaShift还是为2，没有满足自增的条件。这个分区的firstReplicaIndex = (1+2)%3=0。第二个副本的位置replicaIndex(0,2,0,3) = (0+(1+(2+0)%(3-1)))%3 = 1，第三个副本的位置replicaIndex(0,2,1,3) = 2，最终partitionId为2的分区分配策略为[0,1,2]。

以此类推，更多的分配细节可以参考下面的demo，topic-test4的分区分配策略和上面陈述的一致：

```
[root@node3 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test4 --replication-factor 3 --partitions 6
Created topic "topic-test4".
[root@node3 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test4
Topic:topic-test4	PartitionCount:6	ReplicationFactor:3	Configs:
	Topic: topic-test4	Partition: 0	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
	Topic: topic-test4	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: topic-test4	Partition: 2	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
	Topic: topic-test4	Partition: 3	Leader: 2	Replicas: 2,1,0	Isr: 2,1,0
	Topic: topic-test4	Partition: 4	Leader: 0	Replicas: 0,2,1	Isr: 0,2,1
	Topic: topic-test4	Partition: 5	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
```

我们无法预先获知startIndex和nextReplicaShift的值，因为都是随机产生的。startIndex和nextReplicaShift的值可以通过最终的分区分配方案来反推，比如上面的topic-test4，第一个分区（即partitionId=0的分区）的第一个副本为2，那么可由2 = (0+startIndex)%3推断出startIndex为2。之所以startIndex随机是因为这样可以在多个topic的情况下尽可能的均匀分布分区副本，如果这里固定为一个特定值，那么每次的第一个副本都是在这个broker上，进而就会导致少数几个broker所分配到的分区副本过多而其余broker分配到的过少，最终导致负载不均衡。尤其是某些topic的副本数和分区数都比较少，甚至都为1的情况下，所有的副本都落到了那个指定的broker上。与此同时，在分配时位移量nextReplicaShift也可以更好的使得分区副本分配的更加均匀。

### 指定机架的分配策略

下面我们再来看一下指定机架信息的副本分配情况，即方法assignReplicasToBrokersRackAware，注意assignReplicasToBrokersRackUnaware的执行前提是所有的broker都没有配置机架信息，而assignReplicasToBrokersRackAware的执行前提是所有的broker都配置了机架信息，如果出现部分broker配置了机架信息而另一部分没有配置的话，则会抛出AdminOperationException的异常，如果还想要顺利创建topic的话，此时需加上“–disable-rack-aware”，详细demo如下：

```
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test5 --replication-factor 2 --partitions 4 
Error while executing topic command : Not all brokers have rack information. Add --disable-rack-aware in command line to make replica assignment without rack information.
[2018-02-06 00:19:07,213] ERROR kafka.admin.AdminOperationException: Not all brokers have rack information. Add --disable-rack-aware in command line to make replica assignment without rack information.
	at kafka.admin.AdminUtils$.getBrokerMetadatas(AdminUtils.scala:443)
	at kafka.admin.AdminUtils$.createTopic(AdminUtils.scala:461)
	at kafka.admin.TopicCommand$.createTopic(TopicCommand.scala:110)
	at kafka.admin.TopicCommand$.main(TopicCommand.scala:63)
	at kafka.admin.TopicCommand.main(TopicCommand.scala)
 (kafka.admin.TopicCommand$)
[root@node2 kafka_2.12-1.0.0]# bin/kafka-topics.sh --create --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test5 --replication-factor 2 --partitions 4 --disable-rack-aware
Created topic "topic-test5".
[root@node2 kafka_2.12-1.0.0]#
```

assignReplicasToBrokersRackAware方法的详细内容如下，这段代码内容偏多，仅供参考，看得辣眼睛的小伙伴可以习惯性的忽略，后面会做详细的文字介绍。

```
private def assignReplicasToBrokersRackAware(nPartitions: Int,
                                             replicationFactor: Int,
                                             brokerMetadatas: Seq[BrokerMetadata],
                                             fixedStartIndex: Int,
                                             startPartitionId: Int): Map[Int, Seq[Int]] = {
  val brokerRackMap = brokerMetadatas.collect { case BrokerMetadata(id, Some(rack)) =>
    id -> rack
  }.toMap
  val numRacks = brokerRackMap.values.toSet.size//统计机架个数
  val arrangedBrokerList = getRackAlternatedBrokerList(brokerRackMap)//基于机架信息生成一个Broker列表，不同机架上的Broker交替出现

  val numBrokers = arrangedBrokerList.size
  val ret = mutable.Map[Int, Seq[Int]]()
  val startIndex = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(arrangedBrokerList.size)
  var currentPartitionId = math.max(0, startPartitionId)
  var nextReplicaShift = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(arrangedBrokerList.size)
  for (_ <- 0 until nPartitions) {
    if (currentPartitionId > 0 && (currentPartitionId % arrangedBrokerList.size == 0))
      nextReplicaShift += 1
    val firstReplicaIndex = (currentPartitionId + startIndex) % arrangedBrokerList.size
    val leader = arrangedBrokerList(firstReplicaIndex)
    val replicaBuffer = mutable.ArrayBuffer(leader)//每个分区的副本分配列表
    val racksWithReplicas = mutable.Set(brokerRackMap(leader))//每个分区中所分配的机架的列表集
    val brokersWithReplicas = mutable.Set(leader)//每个分区所分配的brokerId的列表集，和racksWithReplicas一起用来做一层筛选处理
    var k = 0
    for (_ <- 0 until replicationFactor - 1) {
      var done = false
      while (!done) {
        val broker = arrangedBrokerList(replicaIndex(firstReplicaIndex, nextReplicaShift * numRacks, k, arrangedBrokerList.size))
        val rack = brokerRackMap(broker)
        if ((!racksWithReplicas.contains(rack) || racksWithReplicas.size == numRacks)
            && (!brokersWithReplicas.contains(broker) || brokersWithReplicas.size == numBrokers)) {
          replicaBuffer += broker
          racksWithReplicas += rack
          brokersWithReplicas += broker
          done = true
        }
        k += 1
      }
    }
    ret.put(currentPartitionId, replicaBuffer)
    currentPartitionId += 1
  }
  ret
}
```

第一步获得brokerId和rack信息的映射关系列表brokerRackMap ，之后调用getRackAlternatedBrokerList()方法对brokerRackMap做进一步的处理生成一个brokerId的列表，这么解释比较拗口，不如举个demo。假设目前有3个机架rack1、rack2和rack3，以及9个broker，分别对应关系如下：

```
rack1: 0, 1, 2
rack2: 3, 4, 5
rack3: 6, 7, 8
```

那么经过getRackAlternatedBrokerList()方法处理过后就变成了[0, 3, 6, 1, 4, 7, 2, 5, 8]这样一个列表，显而易见的这是轮询各个机架上的broker而产生的，之后你可以简单的将这个列表看成是brokerId的列表，对应assignReplicasToBrokersRackUnaware()方法中的brokerArray，但是其中包含了简单的机架分配信息。之后的步骤也和未指定机架信息的算法类似，同样包含startIndex、currentPartiionId, nextReplicaShift的概念，循环为每一个分区分配副本。分配副本时处理第一个副本之外，其余的也调用replicaIndex方法来获得一个broker，但是这里和assignReplicasToBrokersRackUnaware()不同的是，这里不是简单的将这个broker添加到当前分区的副本列表之中，还要经过一层的筛选，满足以下任意一个条件的broker不能被添加到当前分区的副本列表之中：

1. 如果此broker所在的机架中已经存在一个broker拥有该分区的副本，并且还有其他的机架中没有任何一个broker拥有该分区的副本。对应代码中的(!racksWithReplicas.contains(rack) || racksWithReplicas.size == numRacks)
2. 如果此broker中已经拥有该分区的副本，并且还有其他broker中没有该分区的副本。对应代码中的(!brokersWithReplicas.contains(broker) || brokersWithReplicas.size == numBrokers))

无论是带机架信息的策略还是不带机架信息的策略，上层调用方法AdminUtils.assignReplicasToBrokers()最后都是获得一个[Int, Seq[Int]]类型的副本分配列表，其最后作为kafka zookeeper节点/brokers/topics/{topic-name}节点数据。至此kafka的topic创建就讲解完了，有些同学会感到很疑问，全文通篇（包括上一篇）都是在讲述如何分配副本，最后得到的也不过是个分配的方案，并没有真正创建这些副本的环节，其实这个观点没有任何问题，对于通过kafka提供的kafka-topics.sh脚本创建topic的方法来说，它只是提供一个副本的分配方案，并在kafka zookeeper中创建相应的节点而已。kafka broker的服务会注册监听/brokers/topics/目录下是否有节点变化，如果有新节点创建就会监听到，然后根据其节点中的数据（即topic的分区副本分配方案）来创建对应的副本，具体的细节笔者会在后面的副本管理中有详细介绍。

既然整个kafka-topics.sh脚本的作用就只是创建一个zookeeper的节点，并且写上一些分配的方案数据而已，那么我们直接创建一个zookeeper节点来创建一个topic可不可以呢？答案是可以的。在开启的kafka broker的情况下（如果未开启kafka服务的情况下创建zk节点的话，待kafka启动之后是不会再创建实际副本的，只有watch到当前通知才可以），通过zkCli创建一个与topic-test1副本分配方案相同的topic-test6，详细如下：

```
[zk: localhost:2181(CONNECTED) 8] create /kafka100/brokers/topics/topic-test6 {"version":1,"partitions":{"2":[0,1],"1":[1,0],"3":[1,0],"0":[0,1]}}
Created /kafka100/brokers/topics/topic-test6
```

这里再来进一步check下topic-test1和topic-test6是否完全相同：

```
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test1,topic-test6
Topic:topic-test1	PartitionCount:4	ReplicationFactor:2	Configs:
	Topic: topic-test1	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test1	Partition: 1	Leader: 1	Replicas: 1,0	Isr: 1,0
	Topic: topic-test1	Partition: 2	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test1	Partition: 3	Leader: 1	Replicas: 1,0	Isr: 1,0
Topic:topic-test6	PartitionCount:4	ReplicationFactor:2	Configs:
	Topic: topic-test6	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test6	Partition: 1	Leader: 1	Replicas: 1,0	Isr: 1,0
	Topic: topic-test6	Partition: 2	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test6	Partition: 3	Leader: 1	Replicas: 1,0	Isr: 1,0
```

答案显而易见。前面的篇幅也提到了通过kafka-topics.sh脚本的创建方式会对副本的分配有大堆的合格性的校验，但是直接创建zk节点的方式没有这些校验，比如创建一个topic-test7，这个topic节点的数据为：{“version”:1,”partitions”:{“2”:[0,1],”1”:[1],”3”:[1,0],”0”:[0,1]}}，可以看出paritionId为1的分区只有一个副本，我们来检测下是否创建成功：

```
[root@node1 kafka_2.12-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181/kafka100 --topic topic-test7
Topic:topic-test7	PartitionCount:4	ReplicationFactor:2	Configs:
	Topic: topic-test7	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test7	Partition: 1	Leader: 1	Replicas: 1	Isr: 1
	Topic: topic-test7	Partition: 2	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: topic-test7	Partition: 3	Leader: 1	Replicas: 1,0	Isr: 1,0
```

结果也是显而易见的，不过如果没有特殊需求不宜用这种方式，这种方式把控不好就像断了线的风筝一样难以把控。如果又不想用auto.create.topics.enable=true的这种方式，也不想用kafka-topics.sh的这种方式，就像用类似java的编程语言在代码中内嵌创建topic，以便更好的与公司内部的系统结合怎么办？

我们上篇文章中知道kafka-topics.sh内部就是调用了一下kafka.admin.TopicCommand而已，那么我们也调用一下这个可不可以？Of course，下面举一个简单的demo，创建一个副本数为2，分区数为4的topic-test8：

```
public class CreateTopicDemo {
    public static void main(String[] args) {
        //demo: 创建一个副本数为2，分区数为4的主题：topic-test8
        String[] options = new String[]{
                "--create",
                "--zookeeper","192.168.0.2:2181/kafka100",
                "--replication-factor", "2",
                "--partitions", "4",
                "--topic", "topic-test8"
        };
        kafka.admin.TopicCommand.main(options);
    }
}
```

可以看到这种方式和kafka-topics.sh的方式如出一辙，可以用这种方式继承到自动化系统中以创建topic，当然对于topic的删、改、查等都可以通过这种方法来实现，具体的篇幅限制就不一一细表了。

有关kafka的topic的创建细节其实并没有介绍完全，比如create.topic.policy.class.name参数的具体含义与用法，这个会在后面介绍KafkaApis的时候再做具体的介绍，所以为了不迷路，为了涨知识不如关注一波公众号，然后watch。。。