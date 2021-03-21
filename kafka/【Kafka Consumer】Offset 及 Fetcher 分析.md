## 【Kafka Consumer】Offset 及 Fetcher 分析

## 一、前言

代码及测试基于kafka的版本为0.10.2.1 部分方法(例如`KafkaConsumer.beginningOffsets`)在低版本中可能不存在.

## 二、消费者和消费者组

Topic的每一个分区只能被一个消费者组内的一个消费者所消费

Topic分区分配给消费者消费的分配逻辑都是基于默认的分区分配策略进行分析的, 可以通过消费者客户端参数`partition.assignment.strategy`来设置消费者和订阅主题之间的分区分配策略

### 2.1 分区分配策略

- RangeAssignor(org.apache.kafka.clients.consumer.RangeAssignor)默认分配策略
- RoundRobinAssigner
- StickyAassignor

消费者客户端参数`partition.assignment.strategy`可以配置多个分配策略, 之间用`,`隔开.

#### 2.1.1 RangeAssignor

原理是按照消费者总数和分区总数进行整除运算来获取一个跨度, 然后将分区按照跨度进行平均分配, 以保证分区尽可能均匀分配给所有的消费者. RangeAssignor分配策略对应的`partition.assignment.strategy`参数值为`org.apache.kafka.clients.consumer.RangeAssignor`

假设 m=分区数%消费者数, n=分数区/消费者数, 那么前m个消费者每个分配n+1个分区, 后面的(消费者总数-m)个消费者每个分配n个分区.

例如Topic: testA分区数5个分别为test0, test1, test2, test3, test4, Consumer Group: cg消费者数3个分比为cg0, cg1, cg2, 然后按照上述算法最后分配结果:

```
消费者cg0: test0, test1
消费者cg1: test2, test3
消费者cg2: test4
```

如果这个消费者组又订阅了同样5个分区的Topic: testB, testC, ..... , 经过以上分配策略后可以发现消费cg0, cg1的消费压力非常大,会出现过载的情况

#### 2.1.2 RoundRobinAssigner

原理是将消费者组内的所有消费者以及订阅的所有主题分区按照字典排序, 然后通过轮询的方式逐个将分区依次分配给每个消费者. RoundRobinAssigner分配策略对应的`partition.assignment.strategy`参数值为`org.apache.kafka.clients.consumer.RoundRobinAssigner`

例如消费者组中有2个消费者c0和c1都订阅了主题t0和t1, 并且每个主题都是3个分区, 那么订阅的所有分区可以标识为t0p0, t0p1, t0p2, t1p0, t1p1, t1p2最终分配结果为:

```
消费者c0: t0p0, t0p2, t1p1
消费者c1: t0p1, t1p0, t1p2
```

***如果同一个消费者组内的消费者订阅的信息是不同的***, 那么在执行分区分配策略的时候就不是完全的轮询分配, 有可能能会导致分配的不均匀. 

例如例如消费者组中有3个消费者c0, c1和c2都订阅了主题t0, t1和t2, 这3个主题的分区数依次是1, 2, 3个分区, 那么订阅的所有分区可以标识为t0p0, t1p0, t1p1, t2p0, t2p1, t2p2. 具体而言c0订阅的是主题t0, c1订阅的是主题t0和t1, c2订阅的是主题t0, t1和t2,最终分配结果为:

```
消费者c0: t0p0
消费者c1: t1p0
消费者c2: t1p1, t2p0, t2p1, t2p2
```

#### 2.1.3 StickyAassignor

'stick'这个词的意思是粘性的, Kafka自从0.11.x版本开始引入了这种分配策略 ,它主要有两个目的:

1. 分区的分配要尽可能的均匀
2. 分区的分配要尽可能的和上次分配的保持相同

当两者发生冲突时, 第一目标优先于第二目标. 鉴于这两个目标, StickyAssignor分配策略的具体实现要比以上两个分配策略复杂得多, 我们看一下StickyAssignor分配策略的实际效果.

假设消费者组内有3个消费者c0, c1和c2, 它们都订阅了4个主题(t0~t3), 并且每个主题都有2个分区, 也就是说消费者组订阅了t0p0, t0p1, t1p0, t1p1, t2p0, t2p1, t3p0, t3p1这8个分区. 最终的分配结果如下:

```
消费者C0: t0p0, t1p1, t3p0 
消费者C1: t0p1, t2p0, t3p1
消费者C2: t1p0, t2p1
```

加入此时消费者c0脱离了消费者组, 那么消费者组就会执行在均衡操作, 进而消费分区会重新分配. 如果采用RoundRobinAssigner分配策略, 那么此时的分配结果如下:

```
消费者C1: t0p0, t1p0, t2p0, t3p0
消费者C2: t0p1, t1p1, t2p1, t3p1
```

分配结果显示RoundRobinAssigner会按照消费者c1,c2进行重新轮询分配. 但如果使用的是StickyAassignor分配策略, 那么分配的结果为:

```
消费者C1: t0p1, t2p0, t3p1, t1p1
消费者C2: t1p0, t2p1, t0p0, t3p0
```

可以看到分配结果中保留了上次分配中对消费者c1, c2的所有分配结果, 并将原来c0负担的分区分配给了c1,c2而且最终c1和c2的分配还保持了均衡

那么我们在RoundRobinAssigner中举例的那个例子如果采用StickyAassignor分配策略,最终的分配结果是这样的:

```
消费者c0: t0p0
消费者c1: t1p0, t1p1
消费者c2: t2p0, t2p1, t2p2
```

可以看出这才是最优解, 假设此时如果c0脱离了消费者组,那么RoundRobinAssigner分配策略的最终结果是:

```

消费者c1: t0p0, t1p1
消费者c2: t1p0, t2p0, t2p1, t2p2
```

同样如果c0脱离了消费者组,那么StickyAassignor分配策略的最终结果是:

```
消费者c1: t1p0, t1p1, t0p0
消费者c2: t2p0, t2p1, t2p2
```

可以看到StickyAassignor分配策略保留了c1, c2中原有的5各分区.

***<u>综上作述: 使用StickyAassignor分配策略的一个优点是可以使分区重分配具备粘性, 减少不必要的分区移动(即一个分区剥离之前的消费者,转而分配给另一个新的消费者)</u>***

### 2.2 订阅主题、分区

#### 2.2.1 订阅的方式

- 集合订阅的方式 `subscribe(Collection<String>)`

- 正则表达式订阅的方式 `subscribe(Pattern)`

- 指定分区的订阅方式 `assign(Collection<TopicPartition>)`

分别代表了三种不同的订阅状态(SubscriptionState): `AUTO_TOPICS`, `AUTO_PATTERN`和`USER_ASSIGNED`(如果没有订阅, 则SubscriptionState为NONE), 然而这三种状态是互斥的, 在一个消费者中只能使用其中一种, 否则会报 `IllegalStateException`

***通过subscribe()方法订阅主题具有消费者自动再均衡的功能***, 在多个消费者的情况下可以根据分区分配策略来自动分配各个消费者与分区的关系. 当消费者组内的消费者增加或减少时, 分区分配关系会根据策略自动调整, 以实现消费负载均衡及故障自动转移.

***而通过assign()方法订阅分区时是不具备自动再均衡功能的***, 这一点从assign()方法的参数就可以看出端倪, 两种类型的subscribe()都有ConsumerRebalanceListener类型参数的方法, 而assign()方法却没有

##### subscribe(Collection)

一个消费者可以使用`subscribe()`订阅一个或多个主题

如果前后两次订阅了不同的主题, 那么消费者以最后一次的为准

```java
consumer.subscribe(Arrays.asList("topic1"));
consumer.subscribe(Arrays.asList("topic2"));
```

如上示例中, 消费订阅了topic2, 而不是topic1, 也不是topic1和topic2的并集.

##### subscribe(Pattern)

如果消费采用的是正则表达式的方式`subscribe(Pattern)`订阅, 在之后的过程中, 如果有人又创建了新的topic, 并且topic的名字与正则表达式相匹配, 那么这个消费者就可以消费到新添加的topic中的消息

```java
consumer.subscribe(Pattern.compile("test-.*"));
```

##### assign(TopicPartition)

在KafkaConsumer中还提供了一个方法assign()来订阅指定主题分区

```
public void assign(Collection<TopicPartition> partitions);
```

assign方法由用户直接手动consumer实例消费哪些具体分区，根据api上述描述，assign的consumer不会拥有kafka的group management机制，也就是当group内消费者数量变化的时候不会有reblance行为发生。assign的方法不能和subscribe方法同时使用。

### 取消订阅

可以使用 KafkaConusmer 中的 unsubscribe() 方法来取消主题的订阅

1. ***这个方法可以取消通过 subscribe(Collection)方式实现的订阅***
2. ***这个方法也可以取消subscribe(Pattern)方式实现的订阅***
3. ***这个方法还可以取消通过 assign(TopicPartition)方式实现的订阅***

如果将 subscribe(Collection) 和 assign(TopicPartition) 中的集合参数设置为空, 那么作用等同于 unsubscribe() 方法, 下面三种代码效果等同.

```java
consumer.unsubscribe();
consumer.subscribe(new ArrayList<String>());
consumer.subscribe(new ArrayList<TopicPartition>());
```



如果没有订阅任何主题或分区, 那么再继续执行消费, 就会报出 ***IllegalStateException*** 异常.

`java.lang.IllegalStateException: Consumer is not subscribed to any topics or assigned any partitions`

并且如果一个 Consumer 包含了两种或两种以上的消费状态( ***AUTO_TOPICS、AUTO_PATTERN、USER_ASSIGNED***) ,也会报出 ***IllegalStateException*** 异常.

`java.lang.IllegalStateException: Subscription to topics, partitions and pattern are mutually exclusive`



### 消费主题、分区

Kafka 中的消费是一种 ***主动向服务端发出请求来拉取消息*** 的消费方式

Kafka  中的消息消费是一个不断轮询的过程, Consumer 会不断重复调用 poll() 方法, poll() 方法返回的是 所订阅的 主题(分区) 上的一组消息集                                          	

```java
public ConsumerRecords<K, V> poll(final Duration timeout)
```

poll() 方法中有一个超时时间参数 timeout, 用来控制 poll() 的阻塞时间,  在 Consumer 的缓冲区中没有数据时就会发生阻塞.

Kafka中消费的数据类型是 SonsumerRecord<K, V>, 它和Producer发送的 ProducerRecord<K, V>相对应

#### poll() 消费方式

Consumer中关于如何消费有2种策略：		

1. 手动指定
    调用 consumer.seek(TopicPartition, offset), 然后开始poll

2. 自动指定
    poll之前给集群发送请求，让集群告知客户端，当前该TopicPartition的offset是多少



