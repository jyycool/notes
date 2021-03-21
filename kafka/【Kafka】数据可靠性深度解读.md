# 【Kafka】数据可靠性深度解读

## 一、概述

Kakfa 起初是由 LinkedIn 公司开发的一个分布式的消息系统，后成为 Apache 的一部分，它使用 Scala 编写，以可水平扩展和高吞吐率而被广泛使用。目前越来越多的开源分布式处理系统如Cloudera、Apache Storm、Spark 等都支持与 Kafka 集成。

Kafka 凭借着自身的优势，越来越受到互联网企业的青睐，唯品会也采用 Kafka 作为其内部核心消息引擎之一。Kafka 作为一个商业级消息中间件，消息可靠性的重要性可想而知。如何确保消息的精确传输？如何确保消息的准确存储？如何确保消息的正确消费？这些都是需要考虑的问题。本文首先从Kafka 的架构着手，先了解下 Kafka 的基本原理，然后通过对 kakfa 的存储机制、复制原理、同步原理、可靠性和持久性保证等等一步步对其可靠性进行分析，最后通过 benchmark 来增强对Kafka 高可靠性的认知。



------

## 二、Kafka 体系架构

[![img](http://image.honeypps.com/images/papers/2017/213.png)](http://image.honeypps.com/images/papers/2017/213.png)

如上图所示，一个典型的 Kafka 体系架构包括若干 Producer（可以是服务器日志，业务数据，页面前端产生的 page view 等等, producer 是非必要的），若干 broker（Kafka 支持水平扩展，一般 broker 数量越多，集群吞吐率越高），若干 Consumer (Group)，以及一个Zookeeper 集群。Kafka 通过 Zookeeper 管理集群配置，选举 leader，以及在 consumer group 发生变化时进行 rebalance。Producer 使用 push(推)模式将消息发布到 broker，Consumer 使用 pull(拉)模式从 broker 订阅并消费消息。

### 2.1 名词解释

| 名称          | 解释                                                         |
| :------------ | :----------------------------------------------------------- |
| Broker        | 消息中间件处理节点，一个 Kafka 节点就是一个 broker，一个或者多个Broker 可以组成一个 Kafka 集群 |
| Topic         | Kafka 根据 topic 对消息进行归类，发布到 Kafka 集群的每条消息都需要指定一个 topic |
| Producer      | 消息生产者，向 Broker 发送消息的客户端                       |
| Consumer      | 消息消费者，从 Broker 读取消息的客户端                       |
| ConsumerGroup | 每个 Consumer 属于一个特定的 Consumer Group，一条消息可以发送到多个不同的 Consumer Group，但是一个 Consumer Group 中只能有一个 Consumer 能够消费该消息 |
| Partition     | 物理上的概念，一个 topic 可以分为多个 partition，每个 partition 内部是有序的, partition 物理上的表现就是不同文件夹 |

### 2.2 Topic & Partition

一个 topic 可以认为一个一类消息，每个 topic 将被分成多个 partition，每个 partition 在存储层面是 append log 文件。任何发布到此 partition 的消息都会被追加到 log 文件的尾部，每条消息在文件中的位置称为 offset(偏移量)，offset 为一个 long 型的数字，它唯一标记一条消息。每条消息都被 append 到 partition 中，是顺序写磁盘，因此效率非常高（经验证，***顺序写磁盘效率比随机写内存还要高，这是Kafka高吞吐率的一个很重要的保证***）。
[![img](http://image.honeypps.com/images/papers/2017/214.png)](http://image.honeypps.com/images/papers/2017/214.png)

每一条消息被发送到 broker 中，会根据 partition 规则(partitioner 分区器决定)选择被存储到哪一个 partition。如果 partition 规则设置的合理，所有消息可以均匀分布到不同的partition 里，这样就实现了水平扩展。（如果一个 topic 对应一个文件，那这个文件所在的机器I/O将会成为这个topic的性能瓶颈，而partition解决了这个问题）。在创建topic时可以在$KAFKA_HOME/config/server.properties 中指定这个 partition 的数量（如下所示），当然可以在 topic 创建之后去修改 partition 的数量。

```
# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
num.partitions=3
```

在发送一条消息时，可以指定这个消息的 key，producer 根据这个 key 和 partition 机制来判断这个消息发送到哪个 partition。partition 机制可以通过指定 producer 的partition.class 这一参数来指定，该 class 必须实现 kafka.producer.Partitioner 接口。

有关 Topic 与 Partition 的更多细节，可以参考下面的 “3.1 Kafka文件存储机制”这一节。

------

## 三、高可靠性存储分析

Kafka 的高可靠性的保障来源于其健壮的副本（replication）策略。通过调节其副本相关参数，可以使得 Kafka 在性能和可靠性之间运转的游刃有余。Kafka 从 0.8.x 版本开始提供 partition级别的复制,replication 的数量可以在 $KAFKA_HOME/config/server.properties 中配置（default.replication.refactor）。

这里先从 Kafka 文件存储机制入手，从最底层了解 Kafka 的存储细节，进而对其的存储有个微观的认知。之后通过 Kafka 复制原理和同步方式来阐述宏观层面的概念。最后从 ISR，HW，leader选举以及数据可靠性和持久性保证等等各个维度来丰富对 Kafka 相关知识点的认知。

### 3.1 Kafka 文件存储机制

Kafka 中消息是以 topic 进行分类的，生产者通过 topic 向 Kafka broker 发送消息，消费者通过 topic 读取数据。然而 topic 在物理层面又能以 partition 为分组，一个topic可以分成若干个partition，那么topic以及partition又是怎么存储的呢？partition还可以细分为segment，一个partition物理上由多个segment组成，那么这些segment又是什么呢？下面我们来一一揭晓。

为了便于说明问题，假设这里只有一个 Kafka 集群且这个集群只有一个 Kafka broker(即只有一台物理机)。在这个 Kafka broker 中配置（`$KAFKA_HOME/config/server.properties`中）`log.dirs=/tmp/kafka-logs`，以此来设置 Kafka 消息文件存储目录，与此同时创建一个topic：topic_zzh_test，partition的数量为4（`$KAFKA_HOME/bin/kafka-topics.sh --create --zookeeper localhost:2181 --partitions 4 --topic topic_zzh_test --replication-factor 1`）。那么我们此时可以在`/tmp/kafka-logs`目录中可以看到生成了4个目录：

```
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-0
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-1
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-2
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_zzh_test-3
```

在 Kafka 文件存储中，同一个 topic 下有多个不同的 partition，每个 partiton 为一个目录，partition 的名称规则为：topic 名称+有序序号，第一个序号从0开始计，最大的序号为partition 数量减 1，partition 是实际物理上的概念，而 topic 是逻辑上的概念。

上面提到 partition 还可以细分为 segment，这个 segment 又是什么？如果就以 partition 为最小存储单位，我们可以想象当 Kafka producer 不断发送消息，必然会引起 partition文 件的无限扩张，这样对于消息文件的维护以及已经被消费的消息的清理带来严重的影响，所以这里以segment为单位又将 partition 细分。每个 partition(目录)相当于一个巨型文件被平均分配到多个大小相等的 segment(段)数据文件中（每个 segment 文件中消息数量不一定相等）这种特性也方便 old segment 的删除，即方便已被消费的消息的清理，提高磁盘的利用率。每个 partition 只需要支持顺序读写就行，segment 的文件生命周期由服务端配置参数（log.segment.bytes，log.roll.{ms,hours}等若干参数）决定。

segment 文件由三部分组成，分别为:

- .index - segment 索引文件
- .log - segment 数据文件
- .timeindex - segment 时间索引文件(0.8版本及之前的 kafka 没有timeindex文件)

这三个文件的命令规则为：partition 全局的第一个 segment 从 0 开始，后续每个 segment 文件名为上一个 segment 文件最后一条消息的 offset 值，数值大小为 64 位，20 位数字字符长度，没有数字用 0 填充，如下：

```sh
00000000000000000000.index
00000000000000000000.log
00000000000000000000.timeindex
00000000000000170410.index
00000000000000170410.log
00000000000000170410.timeindex
00000000000000239430.index
00000000000000239430.log
00000000000000239430.timeindex
```

以上面的 segment 文件为例，展示出 segment：00000000000000170410的 “.index” 文件和 “.log” 文件的对应的关系，如下图：

[![img](http://image.honeypps.com/images/papers/2017/215.png)](http://image.honeypps.com/images/papers/2017/215.png)

如上图，“.index”索引文件存储大量的元数据，“.log”数据文件存储大量的消息，索引文件中的元数据指向对应数据文件中message的物理偏移地址。其中以“.index”索引文件中的元数据[3, 348]为例，在“.log”数据文件表示第3个消息，即在全局partition中表示170410+3=170413个消息，该消息的物理偏移地址为 348(物理偏移地址即为字节数偏移量)。

那么如何从 partition 中通过 offset 查找 message 呢？

以上图为例，读取 offset=170418 的消息，首先查找segment文件，其中00000000000000000000.index 为最开始的文件，第二个文件为00000000000000170410.index（起始偏移为170410+1=170411），而第三个文件为00000000000000239430.index（起始偏移为239430+1=239431），所以这个 offset=170418 就落到了第二个文件之中。其他后续文件可以依次类推，以其实偏移量命名并排列这些文件，然后根据二分查找法就可以快速定位到具体文件位置。其次根据 00000000000000170410.index 文件中的[8,1325]定位到00000000000000170410.log 文件中的 1325 的位置进行读取。

> 要是读取 offset=170418 的消息，从00000000000000170410.log 文件中的 1325 的位置进行读取，那么怎么知道何时读完本条消息，否则就读到下一条消息的内容了？
> 这个就需要联系到消息的物理结构了，消息都具有固定的物理结构，包括：offset（8 Bytes）、消息体的大小（4 Bytes）、crc32（4 Bytes）、magic（1 Byte）、attributes（1 Byte）、key length（4 Bytes）、key（K Bytes）、payload(N Bytes)等等字段，可以确定一条消息的大小，即读取到哪里截止。

### 3.2 复制原理和同步方式

Kafka 中 topic 的每个 partition 有一个预写式的日志文件，虽然 partition 可以继续细分为若干个 segment 文件，但是对于上层应用来说可以将 partition 看成最小的存储单元（一个有多个 segment 文件拼接的“巨型”文件），每个 partition 都由一些列有序的、不可变的消息组成，这些消息被连续的追加到partition中。

[![img](http://image.honeypps.com/images/papers/2017/216.png)](http://image.honeypps.com/images/papers/2017/216.png)

上图中有两个新名词：HW 和 LEO。这里先介绍下 LEO，LogEndOffset 的缩写，表示每个partition 的 log 最后一条 Message 的位置。HW 是 HighWatermark 的缩写，是指consumer 能够看到的此 partition 的位置，这个涉及到多副本的概念，这里先提及一下，下节再详表。

言归正传，为了提高消息的可靠性，Kafka 每个 topic 的 partition 有 N 个副本（replicas），其中 N(大于等于1) 是 topic 的复制因子（replica fator）的个数。Kafka通过多副本机制实现故障自动转移，当 Kafka 集群中一个 broker 失效情况下仍然保证服务可用。在 Kafka 中发生复制时确保 partition 的日志能有序地写到其他节点上，N 个 replicas 中，其中一个 replica 为 leader，其他都为 follower, leader 处理 partition 的所有读写请求，与此同时，follower 会被动定期地去复制 leader 上的数据。

如下图所示，Kafka 集群中有 4 个 broker, 某 topic 有 3 个 partition,且复制因子即副本个数也为 3：

[![img](http://image.honeypps.com/images/papers/2017/217.png)](http://image.honeypps.com/images/papers/2017/217.png)

Kafka 提供了数据复制算法保证，如果 leader 发生故障或挂掉，一个新 leader 被选举并被接受客户端的消息成功写入。Kafka 确保从同步副本列表中选举一个副本为 leader，或者说 follower追赶 leader 数据。leader 负责维护和跟踪ISR(In-Sync Replicas的缩写，表示副本同步队列，具体可参考下节)中所有 follower 滞后的状态。当 producer 发送一条消息到 broker 后，leader 写入消息并复制到所有 follower。消息提交之后才被成功复制到所有的同步副本。消息复制延迟受最慢的 follower 限制，重要的是快速检测慢副本，如果 follower “落后”太多或者失效，leader 将会把它从 ISR 中删除。

### 3.3 ISR

上节我们涉及到 ISR (In-Sync Replicas)，这个是指副本同步队列。副本数对 Kafka 的吞吐率是有一定的影响，但极大的增强了可用性。默认情况下 Kafka 的 replica 数量为1，即每个partition 都有一个且唯一的 leader，为了确保消息的可靠性，通常应用中将其值(由broker的参数 offsets.topic.replication.factor 指定)大小设置为大于1，比如3。 所有的副本（replicas）统称为Assigned Replicas，即 AR。ISR 是 AR中的一个子集，由 leader 维护ISR 列表，follower 从 leader 同步数据有一些延迟（包括延迟时间replica.lag.time.max.ms 和延迟条数 replica.lag.max.messages 两个维度, 当前最新的版本 0.10.x 中只支持 replica.lag.time.max.ms(延迟时间) 这个维度），任意一个超过阈值的 follower 都会被剔除出 ISR, 存入 OSR（Outof-Sync Replicas）列表，新加入的 follower 也会先存放在 OSR 中。AR=ISR+OSR。

Kafka 0.9.0.0 版本后移除了 replica.lag.max.messages 参数，只保留了replica.lag.time.max.ms 作为 ISR 中副本管理的参数。为什么这样做呢？replica.lag.max.messages 表示当前某个副本落后 leader 的消息数量超过了这个参数的值，那么 leader 就会把 follower 从 ISR 中移除将其放入 OSR 列表中。

假设设置 replica.lag.max.messages=4，那么如果 producer 一次传送至 broker 的消息数量都小于 4条时，因为在 leader 接受到 producer 发送的消息之后而 follower 副本开始拉取这些消息之前，follower 落后 leader 的消息数不会超过 4 条消息，故此没有 follower 移出 ISR，所以这时候 replica.lag.max.message 的设置似乎是合理的。但是 producer 发起瞬时高峰流量，producer 一次发送的消息超过 4 条时，也就是超过 replica.lag.max.messages，此时 follower 都会被认为是与 leader 副本不同步了，从而被踢出了 ISR。但实际上这些follower 都是存活状态的且没有性能问题。那么在之后追上 leader,并被重新加入了 ISR。于是就会出现它们不断地剔出 ISR 然后重新回归 ISR，这无疑增加了无谓的性能损耗。而且这个参数是broker 全局的。设置太大了，影响真正“落后”follower的移除；设置的太小了，导致 follower的频繁进出。无法给定一个合适的 replica.lag.max.messages 的值，故此，新版本的 Kafka 移除了这个参数。

> 注：ISR 中包括：leader 和 follower。

上面一节还涉及到一个概念，即 HW。HW 俗称高水位，HighWatermark 的缩写，取一个partition 对应的 ISR 中最小的 LEO 作为HW，consumer最多只能消费到HW所在的位置。另外每个replica都有 HW,leader 和 follower 各自负责更新自己的 HW 的状态。对于 leader 新写入的消息，consumer 不能立刻消费，leader 会等待该消息被所有 ISR 中的 replicas 同步后更新 HW，此时消息才能被 consumer 消费。这样就保证了如果 leader 所在的 broker 失效，该消息仍然可以从新选举的 leader 中获取。对于来自内部 broker 的读取请求，没有 HW 的限制。

下图详细的说明了当 producer 生产消息至 broker 后，ISR 以及 HW 和 LEO 的流转过程：

[![img](http://image.honeypps.com/images/papers/2017/218.png)](http://image.honeypps.com/images/papers/2017/218.png)

由此可见，Kafka 的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，同步复制要求所有能工作的 follower 都复制完，这条消息才会被 commit，这种复制方式极大的影响了吞吐率。而异步复制方式下，follower 异步的从 leader 复制数据，数据只要被 leader 写入 log 就被认为已经 commit，这种情况下如果 follower 都还没有复制完，落后于 leader 时，突然leader 宕机，则会丢失数据。而 Kafka 的这种使用 ISR 的方式则很好的均衡了确保数据不丢失以及吞吐率。

Kafka 的 ISR 的管理最终都会反馈到 Zookeeper 节点上。具体位置为：/brokers/topics/[topic]/partitions/[partition]/state。目前有两个地方会对这个Zookeeper 的节点进行维护：

1. Controller 来维护：Kafka 集群中的其中一个 Broker 会被选举为 Controller，主要负责 Partition 管理和副本状态管理，也会执行类似于重分配 partition 之类的管理任务。

   在符合某些特定条件下，Controller 下的 LeaderSelector 会选举新的 leader，ISR 和新的 leader_epoch 及 controller_epoch 写入 Zookeeper 的相关节点中。同时发起LeaderAndIsrRequest 通知所有的 replicas。

2. leader 来维护：leader 有单独的线程定期检测 ISR 中 follower 是否脱离 ISR, 如果发现 ISR 变化，则会将新的 ISR 的信息返回到 Zookeeper 的相关节点中。

> ISR 机制中一定要搞清楚一个问题: ISR 中副本的 LEO 一定是大于等于 leader 的 HW, 而 leader 每次成功 commit 之后才会更新 HW, 而 HW 的更新又是以 ISR 中最小 LEO 来确定的。
>
> 这样就形成了一个闭环: leader 接收到消息, replica 同步消息, 每个 replica 将自己同步后的 LEO 提交给 leader, leader 根据 LEO 更新 HW, 落后较多的 replica 被移除 ISR, 然后 leader commit, 这样能确保 ISR 中数据的 commit 都完全同步 leader commit。
>
> 即使 leader 宕机, 只要 ISR 队列中 replica >=1 , 就可以选举出新的 leader

### 3.4 数据可靠性和持久性保证

当 producer 向 leader 发送数据时，可以通过 request.required.acks 参数来设置数据可靠性的级别：

- 1（默认）：这意味着 producer 在 ISR 中的 leader 已成功收到数据并得到确认。如果leader 宕机了，则会丢失数据。
- 0：这意味着 producer 无需等待来自 broker 的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
- -1：producer 需要等待 ISR 中的所有 follower 都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当 ISR 中只有leader时（前面ISR那一节讲到，ISR中的成员由于某些情况会增加也会减少，最少就只剩一个leader），这样就变成了acks=1 的情况。

如果要提高数据的可靠性，在设置 request.required.acks=-1 的同时，也要min.insync.replicas 这个参数(可以在 broker 或者 topic 层面进行设置)的配合，这样才能发挥最大的功效。

min.insync.replicas 这个参数设定 ISR 中的最小副本数是多少，默认值为1，当且仅当request.required.acks 参数设置为-1时，此参数才生效。如果 ISR 中的副本数少于min.insync.replicas 配置的数量时，客户端会返回异常：`org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。`

接下来对 `request.required.acks=1/-1` 的两种情况进行详细分析：

#### 3.4.1 acks = 1

producer 发送数据到 leader，leader 写本地日志成功，返回客户端成功；此时 ISR 中的副本还没有来得及拉取该消息，leader 就宕机了，那么此次发送的消息就会丢失。

[![img](http://image.honeypps.com/images/papers/2017/219.png)](http://image.honeypps.com/images/papers/2017/219.png)

#### 3.4.2 acks = -1

同步(Kafka默认为同步，即 producer.type=sync)的发送模式，replication.factor >=2 且 min.insync.replicas>=2 的情况下，不会丢失数据。

有两种典型情况。acks=-1的情况下，数据发送到 leader, ISR 的 follower 全部完成数据同步后，leader 此时挂掉，那么会选举出新的leader，数据不会丢失。

[![img](http://image.honeypps.com/images/papers/2017/220.png)](http://image.honeypps.com/images/papers/2017/220.png)

acks=-1 的情况下，数据发送到 leader 后，部分 ISR 的副本同步，leader 此时挂掉。比如follower1 和 follower2 都有可能变成新的leader, producer端会得到返回异常，producer端会重新发送数据，数据可能会重复。

[![img](http://image.honeypps.com/images/papers/2017/221.png)](http://image.honeypps.com/images/papers/2017/221.png)

当然上图中如果在 leader crash的时候，follower2 还没有同步到任何数据，而且 follower2 被选举为新的 leader 的话，这样消息就不会重复。

> 注：Kafka 只处理 fail/recover 问题,不处理 Byzantine 问题。

### 3.5 关于 HW 的进一步探讨

考虑上图（即acks=-1,部分ISR副本同步）中的另一种情况，如果在Leader挂掉的时候，follower1 同步了消息 4,5，follower2 同步了消息4，与此同时 follower2 被选举为leader，那么此时 follower1 中的多出的消息5该做如何处理呢？

这里就需要 HW 的协同配合了。如前所述，一个 partition 中的 ISR 列表中，leader 的 HW 是所有 ISR 列表里副本中最小的那个的 LEO。类似于木桶原理，水位取决于最低那块短板。

[![img](http://image.honeypps.com/images/papers/2017/222.png)](http://image.honeypps.com/images/papers/2017/222.png)

如上图，某个 topic 的某 partition 有三个副本，分别为A、B、C。A 作为 leader 肯定是LEO 最高，B 紧随其后，C 机器由于配置比较低，网络比较差，故而同步最慢。

这个时候 A 机器宕机，这时候如果 B 成为 leader，假如没有 HW，在 A 重新恢复之后会做同步(makeFollower)操作，在宕机时log文件之后直接做追加操作，而假如 B 的 LEO 已经达到了 A 的 LEO，会产生数据不一致的情况，所以使用 HW 来避免这种情况。

A 在做同步操作的时候，先将 log 文件截断到之前自己宕机之前的 HW 的位置，即3，之后再从 B 中拉取消息进行同步。

如果失败的 follower 恢复过来，它首先将自己的 log 文件截断到上次 checkpointed 时刻的 HW 的位置，之后再从 leader 中同步消息。

leader 挂掉会重新选举，新的 leader 会发送“指令”让其余的 follower 截断至自身的 HW 的位置然后再拉取新的消息。

> 当 ISR 中的个副本的 LEO 不一致时，如果此时 leader 挂掉，选举新的 leader 时并不是按照 LEO 的高低进行选举，而是按照 ISR中 的顺序选举。

### 3.6 Leader选举

一条消息只有被 ISR 中的所有 follower 都从 leader 复制过去才会被认为已提交。这样就避免了部分数据被写进了 leader，还没来得及被任何 follower 复制就宕机了，而造成数据丢失。

而对于 producer 而言，它可以选择是否等待消息 commit，这可以通过request.required.acks 来设置。这种机制确保了只要 ISR 中有一个或者以上的 follower，一条被 commit 的消息就不会丢失。

有一个很重要的问题是当 leader 宕机了，怎样在 follower 中选举出新的 leader，因为follower 可能落后很多或者直接 crash了，所以必须确保选择“最新”的 follower 作为新的leader。一个基本的原则就是，如果 leader 不在了，新的 leader 必须拥有原来的leader commit的所有消息。这就需要做一个折中，如果 leader 在一个消息被 commit 前等待更多的follower 确认，那么在它挂掉之后就有更多的 follower 可以成为新的leader，但这也会造成吞吐率的下降。

一种常用的选举 leader 的方式是“少数服从多数”(Kafka 并不是采用这种方式)。

- 这种模式下，如果我们有 2f+1 个副本，那么在 commit 之前必须保证有 f+1 个 replica 复制完消息，同时为了保证能正确选举出新的 leader，失败的副本数不能超过 f 个, 这样即使在最坏的情况下(失败的 f 个 replica 都是完全同步 leader 的 replica), 依旧可以保证至少有 f+1-f=1 个完全同步的 replica 可用 。

  这种方式有个很大的优势，系统的延迟取决于最快的几台机器，也就是说比如副本数为3，那么延迟就取决于最快的那个 follower 而不是最慢的那个。“少数服从多数”的方式也有一些劣势，为了保证 leader 选举的正常进行，它所能容忍的失败的 follower 数比较少，如果要容忍1个follower 挂掉，那么至少要 3 个以上的副本，如果要容忍 2 个 follower 挂掉，必须要有5 个以上的副本。也就是说，在生产环境下为了保证较高的容错率，必须要有大量的副本，而大量的副本又会在大数据量下导致性能的急剧下降。这种算法更多用在 Zookeeper 这种共享集群配置的系统中而很少在需要大量数据的系统中使用的原因。HDFS 的 HA 功能也是基于“少数服从多数”的方式，但是其数据存储并不是采用这样的方式。

实际上，leader 选举的算法非常多，比如 Zookeeper 的[Zab](http://web.archive.org/web/20140602093727/http://www.stanford.edu/class/cs347/reading/zab.pdf)、[Raft](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf)以及[Viewstamped Replication](http://pmg.csail.mit.edu/papers/vr-revisited.pdf)。而 Kafka 所使用的 leader 选举算法更像是微软的 [PacificA](https://www.microsoft.com/en-us/research/publication/pacifica-replication-in-log-based-distributed-storage-systems/?from=http%3A%2F%2Fresearch.microsoft.com%2Fapps%2Fpubs%2Fdefault.aspx%3Fid%3D66814) 算法。

`Kafka 在 Zookeeper 中为每一个 partition 动态的维护了一个 ISR，这个 ISR 里的所有replica 都跟上了 leader，只有 ISR 里的成员才能有被选为 leader 的可能`（unclean.leader.election.enable=false）。

> 这句话很重要: 这个 ISR 里的所有 replica 都跟上了 leader，只有 ISR 里的成员才能有被选为 leader 的可能。
>
> 如果 ISR 中的 replica 可以落后于 leader, 此时有种最坏的情况: leader crash, 并且此时 ISR 中所有副本的 commit 都未达到 leader 的 commit, 那么就算选举出的新的 leader, 那么还是会有数据丢失
>
> 所以 ISR 中 replica 的 commit 一定要和 leader 同步

在这种模式下，对于 f+1个 副本，一个 Kafka topic 能在保证不丢失已经 commit 消息的前提下容忍 f 个副本的失败，在大多数使用场景下，这种模式是十分有利的。事实上，为了容忍 f 个副本的失败，“少数服从多数”的方式和 ***ISR 在 commit 前需要等待的副本的数量是一样的***，但是 ISR需要的总的副本的个数几乎是“少数服从多数”的方式的一半。

上文提到，在 ISR 中至少有一个 follower 时，Kafka 可以确保已经 commit 的数据不丢失，但如果某一个 partition 的所有 replica 都挂了，就无法保证数据不丢失了。这种情况下有两种可行的方案：

1. 等待 ISR 中任意一个replica“活”过来，并且选它作为 leader
2. 选择第一个“活”过来的 replica（并不一定是在ISR中）作为 leader

这就需要在可用性和一致性当中作出一个简单的抉择。如果一定要等待 ISR 中的 replica “活”过来，那不可用的时间就可能会相对较长。而且如果 ISR 中所有的 replica 都无法“活”过来了，或者数据丢失了，这个 partition 将永远不可用。选择第一个“活”过来的 replica 作为 leader, 而这个 replica 不是 ISR 中的 replica, 那即使它并不保障已经包含了所有已 commit 的消息，它也会成为 leader 而作为 consumer 的数据源。默认情况下，Kafka 采用第二种策略，即unclean.leader.election.enable=true，也可以将此参数设置为 false 来启用第一种策略。

unclean.leader.election.enable 这个参数对于leader的选举、系统的可用性以及数据的可靠性都有至关重要的影响。下面我们来分析下几种典型的场景。

[![img](http://image.honeypps.com/images/papers/2017/223.png)](http://image.honeypps.com/images/papers/2017/223.png)

如果上图所示，假设某个 partition 中的副本数为 3，replica-0, replica-1, replica-2分别存放在 broker0, broker1 和 broker2 中。AR=(0,1,2)，ISR=(0,1)。

设置 request.required.acks=-1, min.insync.replicas=2，unclean.leader.election.enable=false。这里将 broker0 中的副本也称之为 broker0 起初 broker0 为 leader，broker1 为 follower。

- 当ISR中的replica-0出现crash的情况时，broker1选举为新的leader[ISR=(1)]，因为受min.insync.replicas=2影响，write不能服务，但是read能继续正常服务。此种情况恢复方案：
  1. 尝试恢复(重启)replica-0，如果能起来，系统正常；
  2. 如果replica-0不能恢复，需要将min.insync.replicas设置为1，恢复write功能。
- 当ISR中的replica-0出现crash，紧接着replica-1也出现了crash, 此时[ISR=(1),leader=-1],不能对外提供服务，此种情况恢复方案：
  1. 尝试恢复replica-0和replica-1，如果都能起来，则系统恢复正常；
  2. 如果replica-0起来，而replica-1不能起来，这时候仍然不能选出leader，因为当设置unclean.leader.election.enable=false时，leader只能从ISR中选举，当ISR中所有副本都失效之后，需要ISR中最后失效的那个副本能恢复之后才能选举leader, 即replica-0先失效，replica-1后失效，需要replica-1恢复后才能选举leader。保守的方案建议把unclean.leader.election.enable设置为true,但是这样会有丢失数据的情况发生，这样可以恢复read服务。同样需要将min.insync.replicas设置为1，恢复write功能；
  3. replica-1恢复，replica-0不能恢复，这个情况上面遇到过，read服务可用，需要将min.insync.replicas设置为1，恢复write功能；
  4. replica-0和replica-1都不能恢复，这种情况可以参考情形2.
- 当ISR中的replica-0, replica-1同时宕机,此时[ISR=(0,1)],不能对外提供服务，此种情况恢复方案：尝试恢复replica-0和replica-1，当其中任意一个副本恢复正常时，对外可以提供read服务。直到2个副本恢复正常，write功能才能恢复，或者将将min.insync.replicas设置为1。

### 3.7 Kafka的发送模式

Kafka的发送模式由producer端的配置参数producer.type来设置，这个参数指定了在后台线程中消息的发送方式是同步的还是异步的，默认是同步的方式，即producer.type=sync。如果设置成异步的模式，即producer.type=async，可以是producer以batch的形式push数据，这样会极大的提高broker的性能，但是这样会增加丢失数据的风险。如果需要确保消息的可靠性，必须要将producer.type设置为sync。

对于异步模式，还有4个配套的参数，如下：

| Property                     | Description                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| queue.buffering.max.ms       | 默认值：5000。启用异步模式时，producer缓存消息的时间。比如我们设置成1000时，它会缓存1s的数据再一次发送出去，这样可以极大的增加broker吞吐量，但也会造成时效性的降低。 |
| queue.buffering.max.messages | 默认值：10000。启用异步模式时，producer缓存队列里最大缓存的消息数量，如果超过这个值，producer就会阻塞或者丢掉消息。 |
| queue.enqueue.timeout.ms     | 默认值：-1。当达到上面参数时producer会阻塞等待的时间。如果设置为0，buffer队列满时producer不会阻塞，消息直接被丢掉；若设置为-1，producer会被阻塞，不会丢消息。 |
| batch.num.messages           | 默认值：200。启用异步模式时，一个batch缓存的消息数量。达到这个数值时，producer才会发送消息。（每次批量发送的数量） |

> 以batch的方式推送数据可以极大的提高处理效率，kafka producer可以将消息在内存中累计到一定数量后作为一个batch发送请求。batch的数量大小可以通过producer的参数（batch.num.messages）控制。通过增加batch的大小，可以减少网络请求和磁盘IO的次数，当然具体参数设置需要在效率和时效性方面做一个权衡。在比较新的版本中还有batch.size这个参数。

------

## 四、高可靠性使用分析

### 4.1 消息传输保障

前面已经介绍了Kafka如何进行有效的存储，以及了解了producer和consumer如何工作。接下来讨论的是Kafka如何确保消息在producer和consumer之间传输。有以下三种可能的传输保障（delivery guarantee）:

- At most once: 消息可能会丢，但绝不会重复传输
- At least once：消息绝不会丢，但可能会重复传输
- Exactly once：每条消息肯定会被传输一次且仅传输一次

Kafka的消息传输保障机制非常直观。当producer向broker发送消息时，一旦这条消息被commit，由于副本机制（replication）的存在，它就不会丢失。但是如果producer发送数据给broker后，遇到的网络问题而造成通信中断，那producer就无法判断该条消息是否已经提交（commit）。虽然Kafka无法确定网络故障期间发生了什么，但是producer可以retry多次，确保消息已经正确传输到broker中，所以目前Kafka实现的是at least once。

consumer从broker中读取消息后，可以选择commit，该操作会在Zookeeper中存下该consumer在该partition下读取的消息的offset。该consumer下一次再读该partition时会从下一条开始读取。如未commit，下一次读取的开始位置会跟上一次commit之后的开始位置相同。当然也可以将consumer设置为autocommit，即consumer一旦读取到数据立即自动commit。如果只讨论这一读取消息的过程，那Kafka是确保了exactly once, 但是如果由于前面producer与broker之间的某种原因导致消息的重复，那么这里就是at least once。

考虑这样一种情况，当consumer读完消息之后先commit再处理消息，在这种模式下，如果consumer在commit后还没来得及处理消息就crash了，下次重新开始工作后就无法读到刚刚已提交而未处理的消息，这就对应于at most once了。

读完消息先处理再commit。这种模式下，如果处理完了消息在commit之前consumer crash了，下次重新开始工作时还会处理刚刚未commit的消息，实际上该消息已经被处理过了，这就对应于at least once。

要做到exactly once就需要引入消息去重机制。

### 4.2 消息去重

如上一节所述，Kafka在producer端和consumer端都会出现消息的重复，这就需要去重处理。

Kafka文档中提及GUID(Globally Unique Identifier)的概念，通过客户端生成算法得到每个消息的unique id，同时可映射至broker上存储的地址，即通过GUID便可查询提取消息内容，也便于发送方的幂等性保证，需要在broker上提供此去重处理模块，目前版本尚不支持。

针对GUID, 如果从客户端的角度去重，那么需要引入集中式缓存，必然会增加依赖复杂度，另外缓存的大小难以界定。

不只是Kafka, 类似RabbitMQ以及RocketMQ这类商业级中间件也只保障at least once, 且也无法从自身去进行消息去重。所以我们建议业务方根据自身的业务特点进行去重，比如业务消息本身具备幂等性，或者借助Redis等其他产品进行去重处理。

### 4.3 高可靠性配置

Kafka提供了很高的数据冗余弹性，对于需要数据高可靠性的场景，我们可以增加数据冗余备份数（replication.factor），调高最小写入副本数的个数（min.insync.replicas）等等，但是这样会影响性能。反之，性能提高而可靠性则降低，用户需要自身业务特性在彼此之间做一些权衡性选择。

要保证数据写入到Kafka是安全的，高可靠的，需要如下的配置：

- topic的配置：replication.factor>=3,即副本数至少是3个；2<=min.insync.replicas<=replication.factor
- broker的配置：leader的选举条件unclean.leader.election.enable=false
- producer的配置：request.required.acks=-1(all)，producer.type=sync

------

## 五、BenchMark

Kafka 在唯品会有着很深的历史渊源，根据唯品会消息中间件团队（VMS团队）所掌握的资料显示，在VMS 团队运转的 Kafka 集群中所支撑的 topic 数已接近2000，每天的请求量也已达千亿级。这里就以 Kafka 的高可靠性为基准点来探究几种不同场景下的行为表现，以此来加深对 Kafka 的认知，为大家在以后高效的使用 Kafka 时提供一份依据。

### 5.1 测试环境

Kafka broker用到了 4 台机器，分别为broker[0/1/2/3]配置如下：

- CPU: 24core/2.6GHZ
- Memory: 62G
- Network: 4000Mb
- OS/kernel: CentOs release 6.6 (Final)
- Disk: 1089G
- Kafka版本：0.10.1.0

broker 端 JVM 参数设置：

```scala
-Xmx8G 
-Xms8G 
-server 
-XX:+UseParNewGC 
-XX:+UseConcMarkSweepGC 
-XX:+CMSClassUnloadingEnabled 
-XX:+CMSScavengeBeforeRemark 
-XX:+DisableExplicitGC 
-Djava.awt.headless=true 
-Xloggc:/apps/service/kafka/bin/../logs/kafkaServer-gc.log 
-verbose:gc 
-XX:+PrintGCDetails 
-XX:+PrintGCDateStamps 
-XX:+PrintGCTimeStamps 
-Dcom.sun.management.jmxremote 
-Dcom.sun.management.jmxremote.authenticate=false 
-Dcom.sun.management.jmxremote.ssl=false 
-Dcom.sun.management.jmxremote.port=9999
```

客户端机器配置：

- CPU: 24core/2.6GHZ
- Memory: 3G
- Network: 1000Mb
- OS/kernel: CentOs release 6.3 (Final)
- Disk: 240G

### 5.2 不同场景测试

#### 5.2.1 场景1

测试不同的副本数、min.insync.replicas策略以及request.required.acks策略（以下简称acks策略）对于发送速度（TPS）的影响。

具体配置：一个producer；发送方式为 sync；消息体大小为1kB；partition数为12。副本数为：1/2/4；min.insync.replicas分别为1/2/4；acks分别为-1（all）/1/0。

具体测试数据如下表（min.insync.replicas只在acks=-1时有效）：

| acks | replicas | min.insync.replicas | retries | TPS     |
| :--- | :------- | :------------------ | :------ | :------ |
| -1   | 1        | 1                   | 0       | 28511.3 |
| -1   | 2        | 1                   | 0       | 22359.5 |
| -1   | 2        | 2                   | 0       | 22927.4 |
| -1   | 4        | 1                   | 0       | 16193.9 |
| -1   | 4        | 2                   | 0       | 16599.9 |
| -1   | 4        | 4                   | 0       | 16680.3 |
| 0    | 1        | N/A                 | 0       | 45353.8 |
| 0    | 2        | N/A                 | 0       | 46426.5 |
| 0    | 4        | N/A                 | 0       | 46764.2 |
| 1    | 1        | N/A                 | 0       | 33950.3 |
| 1    | 2        | N/A                 | 0       | 32192.2 |
| 1    | 4        | N/A                 | 0       | 32275.9 |

测试结果分析：

- 客户端的 acks 策略对发送的 TPS 有较大的影响，TPS：acks_0 > acks_1 > ack_-1;
- 副本数越高，TPS 越低；副本数一致时，min.insync.replicas 不影响 TPS；
- acks=0/1 时，TPS 与 min.insync.replicas 参数以及副本数无关，仅受 acks 策略的影响。

下面将 partition 的个数设置为 1，来进一步确认下不同的 acks 策略、不同的min.insync.replicas 策略以及不同的副本数对于发送速度的影响，详细请看情景2和情景3。

#### 5.2.2 场景2

在partition个数固定为1，测试不同的副本数和min.insync.replicas策略对发送速度的影响。

具体配置：一个producer；发送方式为sync；消息体大小为1kB；producer端acks=-1(all)。变换副本数：2/3/4； min.insync.replicas设置为：1/2/4。

测试结果如下：

| replicas | min.insync.replicas | TPS    |
| :------- | :------------------ | :----- |
| 2        | 1                   | 9738.8 |
| 2        | 2                   | 9701.6 |
| 3        | 1                   | 8999.7 |
| 3        | 2                   | 9243.1 |
| 4        | 1                   | 9005.8 |
| 4        | 2                   | 8216.9 |
| 4        | 4                   | 9092.4 |

测试结果分析：副本数越高，TPS越低（这点与场景1的测试结论吻合），但是当partition数为1时差距甚微。min.insync.replicas不影响TPS。

#### 5.2.3 场景3

在 partition 个数固定为 1，测试不同的 acks 策略和副本数对发送速度的影响。

具体配置：一个producer；发送方式为sync；消息体大小为1kB；min.insync.replicas=1。topic副本数为：1/2/4；acks： 0/1/-1。

测试结果如下：

| replicas | acks | TPS   |
| :------- | :--- | :---- |
| 1        | 0    | 76696 |
| 2        | 0    | 57503 |
| 4        | 0    | 59367 |
| 1        | 1    | 19489 |
| 2        | 1    | 20404 |
| 4        | 1    | 18365 |
| 1        | -1   | 18641 |
| 2        | -1   | 9739  |
| 4        | -1   | 9006  |

测试结果分析（与情景1一致）：

- 副本数越多，TPS越低；
- 客户端的acks策略对发送的TPS有较大的影响，TPS：acks_0 > acks_1 > ack_-1。

#### 5.2.4 场景4

测试不同 partition 数对发送速率的影响

具体配置：一个producer；消息体大小为1KB；发送方式为sync；topic副本数为2；min.insync.replicas=2；acks=-1。partition数量设置为1/2/4/8/12。

测试结果：

[![img](http://image.honeypps.com/images/papers/2017/224.png)](http://image.honeypps.com/images/papers/2017/224.png)

测试结果分析：partition的不同会影响TPS，随着partition的个数的增长TPS会有所增长，但并不是一直成正比关系，到达一定临界值时，partition数量的增加反而会使TPS略微降低。

#### 5.2.5 场景5

通过将集群中部分broker设置成不可服务状态，测试对客户端以及消息落盘的影响。**

具体配置：一个producer；消息体大小1KB;发送方式为sync；topic副本数为4；min.insync.replicas设置为2；acks=-1；retries=0/100000000；partition数为12。

具体测试数据如下表：

| acks | replicas | min.insync.replicas | retries   | 测试方法                           | TPS   | 数据落盘                         | 出现错误    |
| :--- | :------- | :------------------ | :-------- | :--------------------------------- | :---- | :------------------------------- | :---------- |
| -1   | 4        | 2                   | 0         | 发送过程中kill两台broker           | 12840 | 一致（部分数据可落盘，部分失败） | 错误1       |
| -1   | 4        | 2                   | 100000000 | 发送过程中kill两台broker           | 13870 | 一致（消息有重复落盘）           | 错误2       |
| -1   | 4        | 2                   | 100000000 | 发送过程中kill三台broker，之后重启 | N/A   | 一致（消息有重复落盘）           | 错误2、3、4 |

出错信息：

- 错误1：客户端返回异常，部分数据可落盘，部分失败：org.apache.kafka.common.errors.NetworkException: The server disconnected before a response was received.
- 错误2：[WARN]internals.Sender - Got error produce response with correlation id 19369 on topic-partition default_channel_replicas_4_1-3, retrying (999999999 attempts left). Error: NETWORK_EXCEPTION
- 错误3： [WARN]internals.Sender - Got error produce response with correlation id 77890 on topic-partition default_channel_replicas_4_1-8, retrying (999999859 attempts left). Error: NOT_ENOUGH_REPLICAS
- 错误4： [WARN]internals.Sender - Got error produce response with correlation id 77705 on topic-partition default_channel_replicas_4_1-3, retrying (999999999 attempts left). Error: NOT_ENOUGH_REPLICAS_AFTER_APPEND

测试结果分析：

- kill两台broker后，客户端可以继续发送。broker减少后，partition的leader分布在剩余的两台broker上，造成了TPS的减小；
- kill三台broker后，客户端无法继续发送。Kafka的自动重试功能开始起作用，当大于等于min.insync.replicas数量的broker恢复后，可以继续发送；
- 当retries不为0时，消息有重复落盘；客户端成功返回的消息都成功落盘，异常时部分消息可以落盘。

#### 5.2.6 场景6

测试单个producer的发送延迟，以及端到端的延迟。

具体配置：：一个producer；消息体大小1KB；发送方式为sync；topic副本数为4；min.insync.replicas设置为2；acks=-1；partition数为12。

测试数据及结果（单位为ms）：

| 发送端(avg) | 发送端(min) | 发送端(max) | 发送端(99%) | 发送端(99.99%) | 消费端(avg) | 消费端(min) | 消费端(max) | 消费端(99%) | 消费端(99.99%) |
| :---------- | :---------- | :---------- | :---------- | :------------- | :---------- | :---------- | :---------- | :---------- | :------------- |
| 1.715       | 1           | 157         | 3           | 29             | 1.646       | 1           | 288         | 4           | 72             |

### 5.3 总结

- 当 acks=-1 时，Kafka 发送端的 TPS 受限于 topic 的副本数量（ISR中），副本越多 TPS 越低；
- acks=0 时，TPS 最高，其次为 1，最差为 -1，即TPS：acks_0 > acks_1 > ack_-1；
- min.insync.replicas 参数不影响 TPS；
- partition 的不同会影响 TPS，随着 partition 的个数的增长 TPS 会有所增长，但并不是一直成正比关系，到达一定临界值时，partition 数量的增加反而会使 TPS 略微降低；
- Kafka 在 acks=-1, min.insync.replicas>=1 时，具有高可靠性，所有成功返回的消息都可以落盘。