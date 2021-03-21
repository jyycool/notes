# 【Kafka】失效副本

## 一、简介

Kafka 从 0.8.x 版本开始引入副本机制，这样可以极大的提高集群的可靠性和稳定性。不过这也使得 Kafka 变得更加复杂起来，失效副本就是所要面临的一个难题。

通常情况下，Kafka 中的每个分区（partition）都会分配多个副本（replica），具体的副本数量由 Broker 级别参数 default.replication.factor（默认大小为1）指定，也可以在创建topic 的时候通过 –replication-factor ${num} 显式指定副本的数量（副本因子）。一般情况下，将前者 default.replication.factor 设置为大于1的值，这样在参数auto.create.topic.enable 为 true的时候，自动创建的 topic 会根据default.replication.factor 的值来创建副本数；或者更加通用的做法是使用后者而指定大于 1 的副本数。

每个分区的多个副本称之为 AR（assigned replicas），包含至多一个 leader 副本和多个follower 副本。与 AR 对应的另一个重要的概念就是 ISR（in-sync replicas），ISR 是指与 leader 副本保持同步状态的副本集合，当然 leader 副本本身也是这个集合中的一员。而 ISR 之外，也就是处于同步失败或失效状态的副本，副本对应的分区也就称之为同步失效分区，即under-replicated分区。

## 二、失效副本的判定

怎么样判定一个分区是否有副本是处于同步失效状态的呢？从 Kafka 0.9.x 版本开始通过唯一的一个参数 replica.lag.time.max.ms（默认大小为10,000）来控制，当 ISR 中的一个 follower 副本滞后 leader 副本的时间超过参数 replica.lag.time.max.ms 指定的值时即判定为副本失效，需要将此 follower 副本剔除出 ISR, 将其加入 OSR。

具体实现原理很简单，当 follower 副本将 leader 副本的 LEO（Log End Offset，每个分区最后一条消息的位置）之前的日志全部同步时，则认为该 follower 副本已经追赶上 leader 副本，此时更新该 follower 副本的 lastCaughtUpTimeMs 标识。Kafka 的副本管理器（ReplicaManager）启动时会启动一个副本过期检测的定时任务，而这个定时任务会定时检查当前时间与副本的 lastCaughtUpTimeMs 差值是否大于参数replica.lag.time.max.ms 指定的值。

千万不要错误的认为 follower 副本只要拉取 leader 副本的数据就会更新lastCaughtUpTimeMs，试想当 leader 副本的消息流入速度大于 follower 副本的拉取速度时，follower 副本一直不断的拉取 leader 副本的消息也不能与 leader 副本同步，如果还将此 follower 副本置于 ISR 中，那么当 leader 副本失效，而选取此 follower 副本为新的leader 副本，那么就会有严重的消息丢失。

Kafka 源码注释中说明了一般有两种情况会导致副本失效：

1. follower 副本进程卡住，在一段时间内根本没有向 leader 副本发起同步请求，比如频繁的Full GC。
2. follower 副本进程同步过慢，在一段时间内都无法追赶上 leader 副本，比如IO开销过大。

这里笔者补充一点，如果通过工具增加了副本因子，那么新增加的副本在赶上 leader 副本之前也都是处于失效状态的。如果一个 follower 副本由于某些原因（比如宕机）而下线，之后又上线，在追赶上 leader 副本之前也是出于失效状态。

在 Kafka 0.9.x 版本之前还有另一个 Broker 级别的参数replica.lag.max.messages（默认大小为4000）也是用来判定失效副本的，当一个follower副本滞后leader副本的消息数超过replica.lag.max.messages 的大小时则判定此follower副本为失效副本。它与replica.lag.time.max.ms 参数判定出的失败副本去并集组成一个失效副本的集合，从而进一步剥离出 ISR。下面给出 0.8.2.2 版本的相关核心代码以供参考：

```java
def getOutOfSyncReplicas(leaderReplica: Replica, keepInSyncTimeMs: Long, keepInSyncMessages: Long): Set[Replica] = {
  val leaderLogEndOffset = leaderReplica.logEndOffset
    val candidateReplicas = inSyncReplicas - leaderReplica
    // Case 1: Stuck followers
    val stuckReplicas = candidateReplicas.filter(r => (time.milliseconds - r.logEndOffsetUpdateTimeMs) > keepInSyncTimeMs)
    if(stuckReplicas.size > 0)
      debug("Stuck replicas for partition [%s,%d] are %s".format(topic, partitionId, stuckReplicas.map(_.brokerId).mkString(",")))
      // Case 2: Slow followers
      val slowReplicas = candidateReplicas.filter(r =>
                                                  r.logEndOffset.messageOffset >= 0 &&
                                                  leaderLogEndOffset.messageOffset - r.logEndOffset.messageOffset > keepInSyncMessages)
      if(slowReplicas.size > 0)
        debug("Slow replicas for partition [%s,%d] are %s".format(topic, partitionId, slowReplicas.map(_.brokerId).mkString(",")))
        stuckReplicas ++ slowReplicas
      }
```

不过这个 replica.lag.max.messages 参数很难给定一个合适的值，若设置的太大则这个参数本身就没有太多意义，若设置的太小则会让 follower 副本反复的处于同步、未同步、同步的死循环中，进而又会造成 ISR 的频繁变动。

而且这个参数是 Broker 级别的，也就是说对 Broker 中的所有 topic 都生效，就以默认的值4000 来说，对于消息流入速度很低的 topic 来说，比如 TPS=10，这个参数并无用武之地；而对于消息流入速度很高的 topic 来说，比如 TPS=20,000，这个参数的取值又会引入 ISR 的频繁变动，所以从 0.9.x 版本开始就彻底移除了这一参数，相关的资料还可以参考[KIP16](https://cwiki.apache.org/confluence/display/KAFKA/KIP-16+-+Automated+Replica+Lag+Tuning)。

具有失效副本的分区可以从侧面洞悉出 Kafka 集群的很多问题，毫不夸张的说：如果只能用一个指标来衡量 Kafka，那么**失效副本分区的个数**必然是首选。Kafka 本身也提供了一个相关的指标，即UnderReplicatedPartitions，这个可以通过JMX访问:

```java
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
```

来获取其值，取值范围是大于等于0的整数。如果获取的 UnderReplicatedPartitions 值大于0，就需要对其进行告警，并进一步诊断其背后的真正原因，有可能是某个 Broker 的问题，也有可能引申到整个集群的问题，也许还要引入其他一些信息、指标等配合找出问题之所在。注意：如果Kafka 集群正在做分区迁移（kafka-reassign-partitions.sh）的时候，这个值也会大于0。

## 三、优先副本的选举

在诊断失效副本之前，可以先尝试执行一次优先副本的选举操作来看看问题是否迎刃而解，反之也能够将排查的范围缩小。

所谓的优先副本是指在Kafka的AR列表中的第一个副本。理想情况下，优先副本就是该分区的leader副本，所以也可以称之为preferred leader。Kafka要确保所有主题的优先副本在Kafka集群中均匀分布，这样就保证了所有分区的Leader均衡分布。保证Leader在集群中均衡分布很重要，因为所有的读写请求都由分区leader副本进行处理，如果leader分布过于集中，就会造成集群负载不均衡。试想一下，如果某分区的leader副本在某个很空闲的Broker上，而它的follower副本宿主于另一个很繁忙的Broker上，那么此follower副本很可能由于分配不到足够的系统资源而无法完成副本同步的任务，进而造成副本失效。

所谓的优先副本的选举是指通过自动或者手动的方式促使优先副本选举为leader，也就是分区平衡，这样可以促进集群的均衡负载，也就进一步的降低失效副本生存的几率。需要注意的是分区平衡并不意味着Kafka集群的负载均衡，因为这还要考虑到集群中的分区分配是否均衡。更进一步每个分区的leader的负载也是各不相同，有些leader副本的负载很高，比如需要承受TPS为3W的负荷，而有些leader副本只需承载个位数的负荷，也就是说就算集群中的分区分配均衡，leader分配也均衡也并不能确保整个集群的负载就是均衡的，还需要其他一些硬性的指标来做进一步的衡量，这个会在下面的内容中涉及，本小节只探讨优先副本的选举。

随着集群运行时间的推移，可能部分节点的变化导致leader进行了重新选举，若优先副本的宿主Broker在发生故障后由其他副本代替而担任了新的leader，就算优先副本的宿主Broker故障恢复而重新回到集群时若没有自动平衡的功能，该副本也不会成为分区的leader。Kafka具备分区自动平衡的功能，且默认情况下此功能是开启的，与此对应的参数是

auto.leader.rebalance.enable=true。如果开启分区自动平衡，则Kafka的Controller会创建一个分区重分配检查及分区重分配操作（onPartitionReassignment）的定时任务，这个定时任务会轮询所有的Broker，计算每个Broker的分区不平衡率（Broker中的不平衡率=非优先副本的leader个数 / 分区总数）是否超过leader.imbalance.per.broker.percentage配置的比率，默认是10%，如果超过设定的比率则会自动执行优先副本的选举动作以求分区平衡。默认执行周期是leader.imbalance.check.interval.seconds=300，即5分钟。

不过在生产环境中不建议将auto.leader.rebalance.enable设置为默认的true，因为这可能会引起负面的性能问题，也有可能会引起客户端一定时间的阻塞。因为执行的时间无法自主掌控，如果在关键时期（比如电商大促波峰期）执行关键任务的关卡摆上一道优先副本的自动选举操作，势必会有业务阻塞、频繁超时之类的风险。前面也分析过分区的均衡也不能确保集群的均衡，而集群一定程度上的不均衡也是可以忍受的，为防关键时期掉链子的行为，笔者建议还是把这类的掌控权把控在自己的手中，可以针对此类相关的埋点指标设置相应的告警，在合适的时机执行合适的操作。

优先副本的选举是一个安全的（Kafka客户端可以自动感知分区leader的变更）并且也容易执行的一类操作。执行优先副本的选举是通过$KAFKA_HOME/bin/路径下的kafka-preferred-replica-election.sh脚本来实现的。举例某Kafka集群有3个Broker，编号（broker.id）为[0,1,2]，且创建了名称为“topic-1”、副本数为3， 分区数为9的一个topic，细节如下（注意其中的IP地址是虚构的）：

```
[root@zzh kafka_1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181,192.168.0.3:2181,192.168.0.3:2181/kafka --topic topic-1
Topic:topic-1	PartitionCount:9	ReplicationFactor:3	Configs:
	Topic: topic-1	Partition: 0	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
	Topic: topic-1	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: topic-1	Partition: 2	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
	Topic: topic-1	Partition: 3	Leader: 2	Replicas: 2,1,0	Isr: 2,1,0
	Topic: topic-1	Partition: 4	Leader: 0	Replicas: 0,2,1	Isr: 0,2,1
	Topic: topic-1	Partition: 5	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
	Topic: topic-1	Partition: 6	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
	Topic: topic-1	Partition: 7	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: topic-1	Partition: 8	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
```

可以看到初始情况下，所有的leader都是AR中的第一个副本也就是优先副本。此时关闭再开启broker.id=2那台Broker，就可以使得topic-1中存在非优先副本的leader，细节如下：

```
Topic:topic-1	PartitionCount:9	ReplicationFactor:3	Configs:
	Topic: topic-1	Partition: 0	Leader: 0	Replicas: 2,0,1	Isr: 0,1,2
	Topic: topic-1	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: topic-1	Partition: 2	Leader: 1	Replicas: 1,2,0	Isr: 1,0,2
	Topic: topic-1	Partition: 3	Leader: 1	Replicas: 2,1,0	Isr: 1,0,2
	Topic: topic-1	Partition: 4	Leader: 0	Replicas: 0,2,1	Isr: 0,1,2
	Topic: topic-1	Partition: 5	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
	Topic: topic-1	Partition: 6	Leader: 0	Replicas: 2,0,1	Isr: 0,1,2
	Topic: topic-1	Partition: 7	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: topic-1	Partition: 8	Leader: 1	Replicas: 1,2,0	Isr: 1,0,2
```

此时可以执行对应的kafka-preferred-replica-election.sh脚本来进行优先副本的选举操作，相关细节如下：

```
[root@zzh kafka_1.0.0]# bin/kafka-preferred-replica-election.sh --zookeeper 192.168.0.2:2181,192.168.0.3:2181,192.168.0.3:2181/kafka
Created preferred replica election path with {"version":1,"partitions":[{"topic":"topic-1","partition":6},{"topic":"topic-1","partition":0},{"topic":"topic-1","partition":7},{"topic":"topic-1","partition":3},{"topic":"topic-1","partition":8},{"topic":"topic-1","partition":2},{"topic":"topic-1","partition":5},{"topic":"topic-1","partition":4},{"topic":"topic-1","partition":1}]}
Successfully started preferred replica election for partitions Set([topic-1,6]], [topic-1,5], [topic-1,4], [topic-1,3], [topic-1,2], [topic-1,7], [topic-1,1], [topic-1,8], [topic-1,0])
```

最终的leader分配又回到初始情况下的状态。不过上面的执行方法是针对Kafka集群中的所有topic都执行一次优先副本的选举，如果集群中存有大量的分区，这一操作有可能会失效，因为这个请求的内容会写入到Zookeeper的节点之中，如果这个请求的内容体过大而超过节点所能存储的数据（默认为1MB）时请求会失败。Kafka提供了更细粒度的优先副本的选举操作，它可以细化到某个topic的某个分区这个层级，这样在面对一次请求过大的问题时可以选择性的进行细粒度拆分，也可以在实现自定义的个性化优先副本的选举操作。

在实现细粒度的优先副本的选举操作之前，首先要建立一个JSON文件，将所需要的topic以及对应的分区编号记录于其中，比如针对topic-1的编号为0的分区进行优先副本的选举操作，对应的JSON文件内容如下（假设此文件命名为partitions.json）：

```
{
        "partitions":[
                {
                        "partition":0,
                        "topic":"topic-1"
                }
        ]
}
```

之后再执行kafka-preferred-replica-election.sh脚本时通过–path-to-json-file参数来指定此

JSON文件，相关细节如下：

```
[root@zzh kafka_2.12-0.10.2.1]# bin/kafka-preferred-replica-election.sh --zookeeper 192.168.0.2:2181,192.168.0.3:2181,192.168.0.3:2181/kafka --path-to-json-file partitions.json 
Created preferred replica election path with {"version":1,"partitions":[{"topic":"topic-1","partition":0}]}
Successfully started preferred replica election for partitions Set([topic-1,0])
```

## 四、失效副本的诊断及预警

在第2小节“失效副本的判定”中提及了UnderReplicatedPartitions指标，这个UnderReplicatedPartitions是一个Broker级别的指标，指的是leader副本在当前Broker上且具有失效副本的分区的个数，也就是说这个指标可以让我们感知失效副本的存在以及波及的分区数量。这一类分区也就是文中篇头所说的同步失效分区，即under-replicated分区。

如果集群中有多个Broker的UnderReplicatedPartitions保持一个大于0的稳定值时，一般暗示着集群中有Broker已经处于下线状态。这种情况下，这个Broker中的分区个数与集群中的所有UnderReplicatedPartitions（处于下线的Broker是不会上报任何指标值的）之和是相等的。通常这类问题是由于机器硬件原因引起的，但也有可能是由于操作系统或者JVM引起的，可以根据这个方向继续做进一步的深入调查。

如果集群中存在Broker的UnderReplicatedPartitions频繁变动，或者处于一个稳定的大于0的值（这里特指没有Broker下线的情况）时，一般暗示着集群出现了性能问题，通常这类问题很难诊断，不过我们可以一步一步的将问题的范围缩小，比如先尝试确定这个性能问题是否只存在于集群的某个Broker中，还是整个集群之上。如果确定集群中所有的under-replicated分区都是在单个Broker上，那么可以看出这个Broker出现了问题，进而可以针对这单一的Broker做专项调查，比如：操作系统、GC、网络状态或者磁盘状态（比如：iowait、ioutil等指标）。

如果多个Broker中都出现了under-replicated分区，这个一般是整个集群的问题，但也有可能是单个Broker出现了问题，前者可以理解，后者有作何解释？想象这样一种情况，如果某个Broker在同步消息方面出了问题，那么其上的follower副本就无法及时有效与其他Broker上的leader副本上进行同步，这样一来就出现了多个Broker都存在under-replicated分区的现象。有一种方法可以查看是否是单个Broker问题已经是哪个Broker出现了问题，就是通过kafka-topic.sh工具来查看集群中所有的under-replicated分区。

举例说明，假设集群中有4个Broker，编号为[0,1,2,3]，相关的under-replicated分区信息如下：

```
[root@zzh kafka-1.0.0]# bin/kafka-topics.sh --describe --zookeeper 192.168.0.2:2181,192.168.0.3:2181,192.168.0.3:2181/kafka --under-replicated
	Topic: topic-1	Partition: 7	Leader: 0		Replicas: 0,1	Isr: 0
	Topic: topic-1	Partition: 1	Leader: 2		Replicas: 1,2	Isr: 2
	Topic: topic-2	Partition: 3	Leader: 3		Replicas: 1,3	Isr: 3
	Topic: topic-2	Partition: 4	Leader: 0		Replicas: 0,1	Isr: 0
	Topic: topic-3	Partition: 7	Leader: 0		Replicas: 0,1	Isr: 0
	Topic: topic-3	Partition: 5	Leader: 3		Replicas: 1,3	Isr: 3
	Topic: topic-4	Partition: 6	Leader: 2		Replicas: 1,2	Isr: 2
	Topic: topic-4	Partition: 2	Leader: 2		Replicas: 1,2	Isr: 2
```

在这个案例中，我们可以看到所有的ISR列表中都出现编号为1的Broker的缺失，进而可以将调查的中心迁移到这个Broker上。如果通过上面的步骤没有定位到某个独立的Broker，那么就需要针对整个集群层面做进一步的探究。

集群层面的问题一般也就是两个方面：资源瓶颈以及负载不均衡。资源瓶颈指的是Broker在某硬件资源的使用上遇到了瓶颈，比如网络、CPU、IO等层面。就以IO而论，Kafka中的消息都是落日志存盘的，生产者线程将消息写入leader副本的性能和IO有着直接的关联，follower副本的同步线程以及消费者的消费线程又要通过IO从磁盘中拉取消息，如果IO层面出现了瓶颈，那么势必会影响全局的走向，与此同时消息的流入流出又都需要和网络打交道。笔者建议硬件层面的指标可以关注CPU的使用率、网络流入/流出速率、磁盘的读/写速率、iowait、ioutil等，也可以适当的关注下文件句柄数、socket句柄数以及内存等方面。

前面在讲述优先副本的时候就涉及到了负载均衡，负载不均衡会影响leader与follower之间的同步效率，进而引起失效副本的产生。集群层面的负载均衡所要考虑的就远比leader副本的分布均衡要复杂的多，需要考虑负载层面的各个因素，将前面所提及的分区数量（partitions）、leader数量（leaders）、CPU占用率（cpuUsed）、网络流入/流出速率(nwBytesIn/nwBytesOut)、磁盘读写速率（ioRead/ioWrite）、iowait、ioutil、文件句柄数（fd）、内存使用率（memUsed）整合考虑。（这些指标不全是必须的，可以自定义增加或者减少。）在资源瓶颈这一方面我们可以单方面的针对每一个单一资源的使用情况设置一个合理的额定阈值，超过额定阈值可以输出告警，进而作出进一步的响应动作，而这里的集群层面的资源整合负载又作何分析？

首先对每一个负载指标做归一化的处理，归一化是一种无量纲的处理手段，把数据映射到0-1范围之内，这样更加方便处理。就以分区数量为例，这里记为MpartitionsMpartitions，对于拥有n个Broker的Kafka集群来说：Mpartitions(n)Mpartitions(n)代表broker.id=n的Broker中拥有的分区数，那么对应的归一化计算公式为：

[![img](http://image.honeypps.com/images/papers/2017/193.jpeg)](http://image.honeypps.com/images/papers/2017/193.jpeg)
用字母P代表每个指标的权重，那么对应前面的所提及的指标分别有：PpartitionsPpartitions、PleadersPleaders、PcpuUsedPcpuUsed、PnwBytesInPnwBytesIn、PnwBytesOutPnwBytesOut、PioReadPioRead、PioWritePioWrite、PiowaitPiowait、PioutilPioutil、PmemUsedPmemUsed、PfdPfd。由此一个Broker(n)的负载值的计算公式为：

[![img](http://image.honeypps.com/images/papers/2017/194.jpeg)](http://image.honeypps.com/images/papers/2017/194.jpeg)
各个权重的取值就需要根据实践检验去调节，不过也可以简单的将各个指标的权重看的一致，那么计算公式也可以简化为：

[![img](http://image.honeypps.com/images/papers/2017/195.jpeg)](http://image.honeypps.com/images/papers/2017/195.jpeg)
将BnBn进一步的再做归一化处理：

[![img](http://image.honeypps.com/images/papers/2017/196.jpeg)](http://image.honeypps.com/images/papers/2017/196.jpeg)
如果将整个集群的负载量看做是1，那么这个Db(n)Db(n) 代表每个Broker所占的负载比重，如果这里采用“饼图”来做集群负载数据可视化的话，那么这个Db(n)Db(n)就代表作每个扇区的比重值。在发现under-replicated分区的时候，可以按照Db(n)Db(n) 值从大到小的顺序逐一对各个Broker进行排查。

那么如何预警Kafka集群中有Broker负载过高或者过低的情况，这里可以引入均方差的概念，不过在计算均方差之前还需要来计算下Broker负载的平均值，这里用BB 来表示：

[![img](http://image.honeypps.com/images/papers/2017/197.jpeg)](http://image.honeypps.com/images/papers/2017/197.jpeg)
这个BB对应的归一化值为：

[![img](http://image.honeypps.com/images/papers/2017/198.jpeg)](http://image.honeypps.com/images/papers/2017/198.jpeg)
对应的集群负载的均方差方差可表示为：

[![img](http://image.honeypps.com/images/papers/2017/199.jpeg)](http://image.honeypps.com/images/papers/2017/199.jpeg)
如果用rnrn表示某个Broker的负载偏离率，那么很明显的有：

[![img](http://image.honeypps.com/images/papers/2017/200.jpeg)](http://image.honeypps.com/images/papers/2017/200.jpeg)
这个rnrn与前面优先副本的选举中的leader.imbalance.per.broker.percentage参数有异曲同工之妙，而且比这个参数更加的精准，我们同样可以设置Broker的负载偏离率的额定阈值r为10%，超过这个阈值可以发送告警。

假设集群中每个Broker的负载偏离率都无限接近r，那么对应的集群负载均方差也就最大：

[![img](http://image.honeypps.com/images/papers/2017/201.jpeg)](http://image.honeypps.com/images/papers/2017/201.jpeg)
比如对于一个具有4个Broker节点的Kafka集群来说，如果设置Broker的负载偏离率为10%，那么对应的集群负载均方差σ就不能超过0.025。针对集群负载均方差设置合理的告警可以提前预防失效副本的发生。

为了让上面这段陈述变得不那么的生涩，这里举一个简单的示例来演示一下这些公式的具体用法。假设集群中有4（即n=4）个Broker节点，为了简化说明只取MpartitionsMpartitions、MleadersMleaders、McpuUsedMcpuUsed、MnwBytesInMnwBytesIn、MnwBytesOutMnwBytesOut这几个作为负载的考量指标，某一时刻集群中各个Broker的负载情况如下表所示：

[![img](http://image.honeypps.com/images/papers/2017/202.jpeg)](http://image.honeypps.com/images/papers/2017/202.jpeg)
首先计算Broker1的DpartitionsDpartitions如下所示：

[![img](http://image.honeypps.com/images/papers/2017/203.jpeg)](http://image.honeypps.com/images/papers/2017/203.jpeg)

其余各个指标的归一化值可以类推，具体如下表所示：
[![img](http://image.honeypps.com/images/papers/2017/204.jpeg)](http://image.honeypps.com/images/papers/2017/204.jpeg)
由上表看到经过简单的归一化处理就将有单位的各种类型的指标归纳为一个简单的数值。进一步的我们省去各个指标权重的考虑，可以计算出此刻各个Broker的负载值：

[![img](http://image.honeypps.com/images/papers/2017/205.jpeg)](http://image.honeypps.com/images/papers/2017/205.jpeg)
同理可得：
[![img](http://image.honeypps.com/images/papers/2017/206.jpeg)](http://image.honeypps.com/images/papers/2017/206.jpeg)
如果把此刻的集群整体负载看成是1，也就是100%，各个Broker分摊这100%的负载，这样可以将Broker的负载值做进一步的归一化处理：

[![img](http://image.honeypps.com/images/papers/2017/207.jpeg)](http://image.honeypps.com/images/papers/2017/207.jpeg)
同理可得：

[![img](http://image.honeypps.com/images/papers/2017/208.jpeg)](http://image.honeypps.com/images/papers/2017/208.jpeg)
如果设置Broker的额定负载偏离率r为10%，那么我们进一步来计算下各个Broker的负载偏离率是否超过值，首先计算Broker1的负载偏离率：

[![img](http://image.honeypps.com/images/papers/2017/209.jpeg)](http://image.honeypps.com/images/papers/2017/209.jpeg)
同理可得：
[![img](http://image.honeypps.com/images/papers/2017/210.jpeg)](http://image.honeypps.com/images/papers/2017/210.jpeg)
可以看出这4个Broker都是相对均衡的，那么集群的负载均方差也就会在合理范围之内（即小于0.025）：

[![img](http://image.honeypps.com/images/papers/2017/211.jpeg)](http://image.honeypps.com/images/papers/2017/211.jpeg)

随着集群运行时间的推移，某一时刻集群中各个Broker的负载情况发生了变化，具体如下表所示：

[![img](http://image.honeypps.com/images/papers/2017/212.jpeg)](http://image.honeypps.com/images/papers/2017/212.jpeg)
具体的计算过程就留给读者自行验算，最后集群的负载均方差为0.0595，大于0.025，所以可以看出发生了负载不均衡的现象。

## 五、写在最后

失效副本会引起Kafka的多种异常发生，严重降低Kafka的可靠性，所以如何有效的预发以及在出现失效副本时如何精准的定位问题是至关重要的。本文尽量从Kafka本身的角度去剖析失效副本，篇幅限制这里并没有针对操作系统、JVM以及集群硬件本身做更深层次的阐述。引起失效副本的原因也是千变万化，希望这篇文章可以给读者在解决相关问题时提供一定的思路。