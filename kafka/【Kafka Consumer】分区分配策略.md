# 【Kafka Consumer】分区分配策略

## 引言

按照 Kafka 默认的消费逻辑设定，一个分区只能被同一个消费组（ConsumerGroup）内的一个消费者消费。假设目前某消费组内只有一个消费者C0，订阅了一个topic，这个topic包含7个分区，也就是说这个消费者C0订阅了7个分区，参考下图（1）。



[![img](http://image.honeypps.com/images/papers/2018/138.png)](http://image.honeypps.com/images/papers/2018/138.png)

此时消费组内又加入了一个新的消费者 C1，按照既定的逻辑需要将原来消费者 C0 的部分分区分配给消费者 C1 消费，情形上图（2），消费者 C0 和 C1 各自负责消费所分配到的分区，相互之间并无实质性的干扰。

接着消费组内又加入了一个新的消费者 C2，如此消费者 C0、C1 和 C2 按照上图（3）中的方式各自负责消费所分配到的分区。

如果消费者过多，出现了消费者的数量大于分区的数量的情况，就会有消费者分配不到任何分区。参考下图，一共有8个消费者，7个分区，那么最后的消费者 C7 由于分配不到任何分区进而就无法消费任何消息。

[![img](http://image.honeypps.com/images/papers/2018/139.png)](http://image.honeypps.com/images/papers/2018/139.png)

上面各个示例中的整套逻辑是按照 Kafka 中默认的分区分配策略来实施的。Kafka 提供了 KafkaConsumer 参数 partition.assignment.strategy 用来设置消费者与订阅主题之间的分区分配策略。kafka 对该参数提供了 3 种策略值:

- RangeAssignor 
- RoundRobinAssignor 
- StickyAssignor

默认情况下，此参数的值为：org.apache.kafka.clients.consumer.RangeAssignor，即采用 RangeAssignor 分配策略。KafkaConsumer 参数 partition.asssignment.strategy 可以配置多个分配策略，彼此之间以逗号分隔。

## 一、RangeAssignor

RangeAssignor 策略的原理是按照消费者总数和分区总数进行整除运算来获得一个跨度，然后将分区按照跨度进行平均分配，以保证分区尽可能均匀地分配给所有的消费者。对于每一个 topic，RangeAssignor 策略会将消费组内所有订阅这个 topic 的消费者按照名称的字典序排序，然后为每个消费者划分固定的分区范围，如果不够平均分配，那么字典序靠前的消费者会被多分配一个分区。

假设 n=parts(分区数)/csms(消费者数量)，m=parts%csms，那么前 m 个消费者每个分配 n+1个分区，后面的（消费者数量-m）个消费者每个分配 n个分区。

为了更加通俗的讲解 RangeAssignor 策略，我们不妨再举一些示例。假设消费组内有 2 个消费者C0 和 C1，都订阅了主题 t0 和 t1，并且每个主题都有 4 个分区，那么所订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t0p3、t1p0、t1p1、t1p2、t1p3。最终的分配结果为：

```
消费者C0：t0p0、t0p1、t1p0、t1p1
消费者C1：t0p2、t0p3、t1p2、t1p3
```

这样分配的很均匀，那么此种分配策略能够一直保持这种良好的特性呢？我们再来看下另外一种情况。假设上面例子中 2 个主题都只有 3 个分区，那么所订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t1p0、t1p1、t1p2。最终的分配结果为：

```
消费者C0：t0p0、t0p1、t1p0、t1p1
消费者C1：t0p2、t1p2
```

可以明显的看到这样的分配并不均匀，如果将类似的情形扩大，有可能会出现部分消费者过载的情况。对此我们再来看下另一种 RoundRobinAssignor 策略的分配效果如何。

## 二、RoundRobinAssignor

RoundRobinAssignor 策略的原理是将消费组内所有消费者以及消费者所订阅的所有 topic 的partition 按照字典序排序，然后通过轮询方式逐个将分区以此分配给每个消费者。RoundRobinAssignor 策略对应的 partition.assignment.strategy 参数值为：org.apache.kafka.clients.consumer.RoundRobinAssignor。

如果同一个消费组内所有的消费者的订阅信息都是相同的，那么 RoundRobinAssignor策略的分区分配会是均匀的。举例，假设消费组中有2个消费者C0和C1，都订阅了主题t0和t1，并且每个主题都有3个分区，那么所订阅的所有分区可以标识为：t0p0、t0p1、t0p2、t1p0、t1p1、t1p2。最终的分配结果为：

```
消费者C0：t0p0、t0p2、t1p1
消费者C1：t0p1、t1p0、t1p2
```

如果同一个消费组内的消费者所订阅的信息是不相同的，那么在执行分区分配的时候就不是完全的轮询分配，有可能会导致分区分配的不均匀。如果某个消费者没有订阅消费组内的某个topic，那么在分配分区的时候此消费者将分配不到这个topic的任何分区。

举例，假设消费组内有3个消费者C0、C1和C2，它们共订阅了3个主题：t0、t1、t2，这3个主题分别有1、2、3个分区，即整个消费组订阅了t0p0、t1p0、t1p1、t2p0、t2p1、t2p2这6个分区。具体而言，消费者C0订阅的是主题t0，消费者C1订阅的是主题t0和t1，消费者C2订阅的是主题t0、t1和t2，那么最终的分配结果为：

```
消费者C0：t0p0
消费者C1：t1p0
消费者C2：t1p1、t2p0、t2p1、t2p2
```

可以看到 RoundRobinAssignor 策略也不是十分完美，这样分配其实并不是最优解，因为完全可以将分区 t1p1 分配给消费者C1。

------

## 三、StickyAssignor

我们再来看一下StickyAssignor策略，“sticky”这个单词可以翻译为“粘性的”，Kafka从0.11.x版本开始引入这种分配策略，它主要有两个目的：

1. 分区的分配要尽可能的均匀；
2. 分区的分配尽可能的与上次分配的保持相同。
   当两者发生冲突时，第一个目标优先于第二个目标。鉴于这两个目标，StickyAssignor 策略的具体实现要比 RangeAssignor 和 RoundRobinAssignor 这两种分配策略要复杂很多。我们举例来看一下 StickyAssignor 策略的实际效果。

假设消费组内有3个消费者：C0、C1和C2，它们都订阅了4个主题：t0、t1、t2、t3，并且每个主题有2个分区，也就是说整个消费组订阅了t0p0、t0p1、t1p0、t1p1、t2p0、t2p1、t3p0、t3p1这8个分区。最终的分配结果如下：

```
消费者C0：t0p0、t1p1、t3p0
消费者C1：t0p1、t2p0、t3p1
消费者C2：t1p0、t2p1
```

这样初看上去似乎与采用 RoundRobinAssignor 策略所分配的结果相同，但事实是否真的如此呢？再假设此时消费者 C1 脱离了消费组，那么消费组就会执行再平衡操作，进而消费分区会重新分配。如果采用 RoundRobinAssignor 策略，那么此时的分配结果如下：

```
消费者C0：t0p0、t1p0、t2p0、t3p0
消费者C2：t0p1、t1p1、t2p1、t3p1
```

如分配结果所示，RoundRobinAssignor策略会按照消费者C0和C2进行重新轮询分配。而如果此时使用的是 StickyAssignor 策略，那么分配结果为：

```
消费者C0：t0p0、t1p1、t3p0、t2p0
消费者C2：t1p0、t2p1、t0p1、t3p1
```

可以看到分配结果中保留了上一次分配中对于消费者C0和C2的所有分配结果，并将原来消费者C1的“负担”分配给了剩余的两个消费者C0和C2，最终C0和C2的分配还保持了均衡。

如果发生分区重分配，那么对于同一个分区而言有可能之前的消费者和新指派的消费者不是同一个，对于之前消费者进行到一半的处理还要在新指派的消费者中再次复现一遍，这显然很浪费系统资源。StickyAssignor策略如同其名称中的“sticky”一样，让分配策略具备一定的“粘性”，尽可能地让前后两次分配相同，进而减少系统资源的损耗以及其它异常情况的发生。

到目前为止所分析的都是消费者的订阅信息都是相同的情况，我们来看一下订阅信息不同的情况下的处理。

举例，同样消费组内有3个消费者：C0、C1和C2，集群中有3个主题：t0、t1和t2，这3个主题分别有1、2个分区，也就是说集群中有t0p0、t1p0、t1p1、t2p0、t2p1、t2p2这6个分区。消费者C0订阅了主题t0，消费者C1订阅了主题t0和t1，消费者C2订阅了主题t0、t1和t2。

如果此时采用RoundRobinAssignor策略，那么最终的分配结果如下所示（和讲述RoundRobinAssignor策略时的一样，这样不妨赘述一下）：

```
【分配结果集1】
消费者C0：t0p0
消费者C1：t1p0
消费者C2：t1p1、t2p0、t2p1、t2p2
```

如果此时采用的是 StickyAssignor 策略，那么最终的分配结果为：

```
【分配结果集2】
消费者C0：t0p0
消费者C1：t1p0、t1p1
消费者C2：t2p0、t2p1、t2p2
```

可以看到这是一个最优解（消费者C0没有订阅主题t1和t2，所以不能分配主题t1和t2中的任何分区给它，对于消费者 C1 也可同理推断）。
假如此时消费者 C0 脱离了消费组，那么 RoundRobinAssignor 策略的分配结果为：

```
消费者C1：t0p0、t1p1
消费者C2：t1p0、t2p0、t2p1、t2p2
```

可以看到 RoundRobinAssignor 策略保留了消费者C1和C2中原有的3个分区的分配：t2p0、t2p1和t2p2（针对结果集1）。而如果采用的是 StickyAssignor 策略，那么分配结果为：

```
消费者C1：t1p0、t1p1、t0p0
消费者C2：t2p0、t2p1、t2p2
```

可以看到 StickyAssignor 策略保留了消费者C1和C2中原有的5个分区的分配：t1p0、t1p1、t2p0、t2p1、t2p2。

从结果上看 StickyAssignor 策略比另外两者分配策略而言显得更加的优异，这个策略的代码实现也是异常复杂，如果读者没有接触过这种分配策略，不妨使用一下来尝尝鲜。

## 四、自定义分区分配策略

读者不仅可以任意选用 Kafka 所提供的3种分配策略，还可以自定义分配策略来实现更多可选的功能。自定义的分配策略必须要实现org.apache.kafka.clients.consumer.internals.PartitionAssignor接口。PartitionAssignor 接口的定义如下：



```java
Subscription subscription(Set<String> topics);
String name();
Map<String, Assignment> assign(Cluster metadata, 
                               Map<String, Subscription> subscriptions);
void onAssignment(Assignment assignment);
class Subscription {
  private final List<String> topics;
  private final ByteBuffer userData;
  （省略若干方法……）
}
class Assignment {
  private final List<TopicPartition> partitions;
  private final ByteBuffer userData;
  （省略若干方法……）
}
```

PartitionAssignor 接口中定义了两个内部类：Subscription 和 Assignment。

Subscription 类用来表示消费者的订阅信息，类中有两个属性：topics 和 userData，分别表示消费者所订阅topic列表和用户自定义信息。PartitionAssignor 接口通过 subscription()方法来设置消费者自身相关的 Subscription 信息，注意到此方法中只有一个参数 topics，与Subscription 类中的 topics 的相互呼应，但是并没有有关 userData 的参数体现。为了增强用户对分配结果的控制，可以在 subscription()方法内部添加一些影响分配的用户自定义信息赋予userData，比如：权重、ip地址、host或者机架（rack）等等。
[![img](http://image.honeypps.com/images/papers/2018/140.png)](http://image.honeypps.com/images/papers/2018/140.png)

举例，在 subscription() 这个方法中提供机架信息，标识此消费者所部署的机架位置，在分区分配时可以根据分区的 leader 副本所在的机架位置来实施具体的分配，这样可以让消费者与所需拉取消息的broker节点处于同一机架。参考下图，消费者 consumer1 和 broker1 都部署在机架 rack1 上，消费者 consumer2 和 broker2 都部署在机架 rack2 上。如果分区的分配不是机架感知的，那么有可能与图（上部分）中的分配结果一样，consumer1消费broker2中的分区，而consumer2消费broker1中的分区；如果分区的分配是机架感知的，那么就会出现图（下部分）的分配结果，consumer1消费broker1中的分区，而consumer2消费broker2中的分区，这样相比于前一种情形而言，既可以减少消费延迟又可以减少跨机架带宽的占用。

再来说一下Assignment类，它是用来表示分配结果信息的，类中也有两个属性：partitions和userData，分别表示所分配到的分区集合和用户自定义的数据。可以通过PartitionAssignor接口中的onAssignment()方法是在每个消费者收到消费组leader分配结果时的回调函数，例如在StickyAssignor策略中就是通过这个方法保存当前的分配方案，以备在下次消费组再平衡（rebalance）时可以提供分配参考依据。

接口中的name()方法用来提供分配策略的名称，对于Kafka提供的3种分配策略而言，RangeAssignor对应的protocol_name为“range”，RoundRobinAssignor对应的protocol_name为“roundrobin”，StickyAssignor对应的protocol_name为“sticky”，所以自定义的分配策略中要注意命名的时候不要与已存在的分配策略发生冲突。这个命名用来标识分配策略的名称，在后面所描述的加入消费组以及选举消费组leader的时候会有涉及。

真正的分区分配方案的实现是在assign()方法中，方法中的参数metadata表示集群的元数据信息，而subscriptions表示消费组内各个消费者成员的订阅信息，最终方法返回各个消费者的分配信息。

Kafka 中还提供了一个抽象类org.apache.kafka.clients.consumer.internals.AbstractPartitionAssignor，它可以简化 PartitionAssignor 接口的实现，对 assign() 方法进行了实现，其中会将Subscription 中的 userData 信息去掉后，在进行分配。Kafka 提供的3种分配策略都是继承自这个抽象类。如果开发人员在自定义分区分配策略时需要使用userData信息来控制分区分配的结果，那么就不能直接继承AbstractPartitionAssignor这个抽象类，而需要直接实现PartitionAssignor接口。

下面笔者参考 Kafka 中的 RangeAssignor 策略来自定义一个随机的分配策略，这里笔者称之为RandomAssignor，具体代码实现如下：

```java
package org.apache.kafka.clients.consumer;

import org.apache.kafka.clients.consumer.internals.AbstractPartitionAssignor;
import org.apache.kafka.common.TopicPartition;
import java.util.*;

public class RandomAssignor extends AbstractPartitionAssignor {
    @Override
    public String name() {
        return "random";
    }

    @Override
    public Map<String, List<TopicPartition>> assign(
            Map<String, Integer> partitionsPerTopic,
            Map<String, Subscription> subscriptions) {
        Map<String, List<String>> consumersPerTopic = 
consumersPerTopic(subscriptions);
        Map<String, List<TopicPartition>> assignment = new HashMap<>();
        for (String memberId : subscriptions.keySet()) {
            assignment.put(memberId, new ArrayList<>());
        }

        // 针对每一个topic进行分区分配
        for (Map.Entry<String, List<String>> topicEntry : 
consumersPerTopic.entrySet()) {
            String topic = topicEntry.getKey();
            List<String> consumersForTopic = topicEntry.getValue();
            int consumerSize = consumersForTopic.size();

            Integer numPartitionsForTopic = partitionsPerTopic.get(topic);
            if (numPartitionsForTopic == null) {
                continue;
            }

            // 当前topic下的所有分区
            List<TopicPartition> partitions = 
AbstractPartitionAssignor.partitions(topic, 
numPartitionsForTopic);
            // 将每个分区随机分配给一个消费者
            for (TopicPartition partition : partitions) {
                int rand = new Random().nextInt(consumerSize);
                String randomConsumer = consumersForTopic.get(rand);
                assignment.get(randomConsumer).add(partition);
            }
        }
        return assignment;
    }

    // 获取每个topic所对应的消费者列表，即：[topic, List[consumer]]
    private Map<String, List<String>> consumersPerTopic(
Map<String, Subscription> consumerMetadata) {
        Map<String, List<String>> res = new HashMap<>();
        for (Map.Entry<String, Subscription> subscriptionEntry : 
consumerMetadata.entrySet()) {
            String consumerId = subscriptionEntry.getKey();
            for (String topic : subscriptionEntry.getValue().topics())
                put(res, topic, consumerId);
        }
        return res;
    }
}
```

在使用时，消费者客户端需要添加相应的Properties参数，示例如下：

```java
properties.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG, 
RandomAssignor.class.getName());
```

这里只是演示如何自定义实现一个分区分配策略，RandomAssignor 的实现并不是特别的理想，并不见得会比 Kafka 自身所提供的 RangeAssignor 策略之类的要好。

## 五、分配的实施

我们了解了 Kafka 中消费者的分区分配策略之后是否会有这样的疑问：如果消费者客户端中配置了两个分配策略，那么以哪个为准？如果有多个消费者，彼此所配置的分配策略并不完全相同，那么以哪个为准？多个消费者之间的分区分配是需要协同的，那么这个协同的过程又是怎样？

在 kafka 中有一个组协调器（GroupCoordinator）负责来协调消费组内各个消费者的分区分配，对于每一个消费组而言，在 kafka 服务端都会有其对应的一个组协调器。具体的协调分区分配的过程如下：

1. 首先各个消费者向 GroupCoordinator 提案各自的分配策略。如下图所示，各个消费者提案的分配策略和订阅信息都包含在 JoinGroupRequest 请求中。

[![img](http://image.honeypps.com/images/papers/2018/141.png)](http://image.honeypps.com/images/papers/2018/141.png)

2. GroupCoordinator 收集各个消费者的提案，然后执行以下两个步骤：一、选举消费组的leader；二、选举消费组的分区分配策略。

   选举消费组的分区分配策略比较好理解，为什么这里还要选举消费组的leader，因为最终的分区分配策略的实施需要有一个成员来执行，而这个leader消费者正好扮演了这一个角色。在 Kafka中把具体的分区分配策略的具体执行权交给了消费者客户端，这样可以提供更高的灵活性。比如需要变更分配策略，那么只需修改消费者客户端就醒来，而不必要修改并重启Kafka服务端。

   怎么选举消费组的leader? 这个分两种情况分析：如果消费组内还没有leader，那么第一个加入消费组的消费者即为消费组的leader；如果某一时刻leader消费者由于某些原因退出了消费组，那么就会重新选举一个新的leader，这个重新选举leader的过程又更为“随意”了，相关代码如下：

   ```scala
   //scala code.
   private val members = new mutable.HashMap[String, MemberMetadata]
   var leaderId = members.keys.head
   ```

   在GroupCoordinator 中消费者的信息是以 HashMap 的形式存储的，其中 key 为消费者的名称，而 value 是消费者相关的元数据信息。leaderId 表示 leader 消费者的名称，它的取值为 HashMap 中的第一个键值对的 key，这种选举的方式基本上和随机挑选无异。

   总体上来说，消费组的leader选举过程是很随意的。

   怎么选举消费组的分配策略？投票决定。每个消费者都可以设置自己的分区分配策略，对于消费组而言需要从各个消费者所呈报上来的各个分配策略中选举一个彼此都“信服”的策略来进行整体上的分区分配。这个分区分配的选举并非由 leader 消费者来决定，而是根据消费组内的各个消费者投票来决定。这里所说的“根据组内的各个消费者投票来决定”不是指 GroupCoordinator 还要与各个消费者进行进一步交互来实施，而是根据各个消费者所呈报的分配策略来实施。最终所选举的分配策略基本上可以看做是被各个消费者所支持的最多的策略，具体的选举过程如下：

   1. 收集各个消费者所支持的所有分配策略，组成候选集 candidates。

   2. 每个消费者从候选集 candidates 中找出第一个自身所支持的策略，为这个策略投上一票。

   3. 计算候选集中各个策略的选票数，选票数最多的策略即为当前消费组的分配策略。
      如果某个消费者并不支持所选举出的分配策略，那么就会报错。

3. GroupCoordinator 发送回执给各个消费者，并交由 leader 消费者执行具体的分区分配。

   [![img](http://image.honeypps.com/images/papers/2018/142.png)](http://image.honeypps.com/images/papers/2018/142.png)
   如上图所示，JoinGroupResponse 回执中包含有 GroupCoordinator 中投票选举出的分配策略的信息。并且，只有 leader 消费者的回执中包含各个消费者的订阅信息，因为只需要leader 消费者根据订阅信息来执行具体的分配，其余的消费并不需要。

4. leader 消费者在整理出具体的分区分配方案后通过 SyncGroupRequest 请求提交给GroupCoordinator，然后 GroupCoordinator 为每个消费者挑选出各自的分配结果并通过SyncGroupResponse 回执以告知它们。

[![img](http://image.honeypps.com/images/papers/2018/143.png)](http://image.honeypps.com/images/papers/2018/143.png)

这里涉及到了消费者再平衡时的一些内容，不过本文只对分区分配策略做相关陈述，故省去了与此无关的一些细节内容：比如 GroupCoordinator 是什么？怎么定义和查找 GroupCoodinator？为什么要有这个东西？JoinGroupRequest 和 SyncGroupRequest 除了与分区分配相关之外还有什么作用？