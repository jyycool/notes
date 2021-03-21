# 【Kafka】事务特性

在说 Kafka 的事务之前，先要说一下 Kafka 中幂等的实现。幂等和事务是 Kafka 0.11.0.0 版本引入的两个特性，以此来实现 EOS（exactly once semantics，精确一次处理语义）。

## 1. Kafka 的幂等性

幂等，简单地说就是对接口的多次调用所产生的结果和调用一次是一致的。生产者在进行重试的时候有可能会重复写入消息，而使用 Kafka 的幂等性功能之后就可以避免这种情况。

### 1.1 配置幂等功能

开启幂等性功能的方式很简单，只需要显式地将 KafkaProducer 参数 `enable.idempotence` 设置为true 即可（这个参数的默认值为false）。

```scala
props.put(ProducerConfig.ENABLE_IDEMPOTENCE, "true")
```

不过如果要保证幂等性功能的正常, 还需要确保 KafkaProducer 的 retries、acks、max.in.flight.requests.per.connection 这几个参数不被配置错。实际上在使用幂等性的时候, 用户完全可以不用也不建议配置这几个参数。

如果用户需要显示的配置这几个参数, 那么配置建议如下:

- retries

  默认值: Long.MAX_VALUE, 自定义配置值必须大于 0, 否则会报 ConfigException。

- max.in.flight.requests.per.connection

  默认值: 5, 自定义配置值必须小于等于 5, 否则也会报 ConfigException。

- acks 默认值: 1, 自定义配置值必须等于 -1, 如果用户开启幂等, 且未配置该参数, KafkaProducer 会将其置为 -1。

### 1.2 幂等的实现原理

为了实现 KafkaPrducer 的幂等性。Kafka 为此引入了 producer id（以下简称PID）和序列号（sequence number）这两个概念。每个新的生产者实例在初始化的时候都会被分配一个 PID，这个 PID 对用户而言是完全透明的。对于每个 PID，消息发送到的每一个分区都有对应的序列号，这些序列号从 0 开始单调递增。生产者每发送一条消息就会将对应的序列号的值加 1。

broker 端会在内存中为每一对 <PID, 分区> 维护一个序列号。对于收到的每一条消息，只有当它的序列号的值（SN_new）比 broker 端中维护的对应的序列号的值（SN_old）大1（即SN_new = SN_old + 1）时，broker才会接收它。

> 这里很聪明: 用 pid 来表示全局唯一的 KafkaProducer, 然后在内存使用组合 key=(pid, partition_id), 来维护一个单调递增的 seq_num, 每条由 pid 的 KafkaProducer 发送到 partition_id 的消息, 该消息的 seq_num 只有比内存里维护的 (pid, partition_id) 的旧 seq_num 大 1 的时候, broker 才会保存该数据。

如果 SN_new < SN_old + 1，那么说明消息被重复写入，broker可以直接将其丢弃。如果SN_new > SN_old + 1，那么说明中间有数据尚未写入，出现了乱序，暗示可能有消息丢失，这个异常是一个严重的异常。

引入序列号来实现幂等也只是针对每一对 <PID, 分区> 而言的，也就是说，Kafka的幂等只能保证单个生产者会话（session）中单分区的幂等。

```java
val record = new ProducerRecord[String, String]("tpc", null, "msg1")
producer.send(record)
producer.send(record)
```

上面示例中发送了两条相同的消息, 不过这仅仅是消息的内容相同, 对 Kafka 而言是两条不同的消息, 因为 broker 会为这两条消息分配不同的序列号(seq num)。Kafka 并不会保证这两条消息内容的幂等。

## 2. Kafka 的事务

幂等性不能跨 Topic 分区运作，而事务可以弥补这个缺陷。事务可以保证写入操作的原子性。操作的原子性是指多个操作要么全部成功, 要么全部失败, 不存在部分成功、部分失败的可能。

对于流式应用(Stream Processing Application, 如 Flink, Spark)而言, 一个典型的应用模式为 "consume -> transforms -> produce"。在这种模式下, 消费和生产并存: 应用程序从某个 topic 中消费消息, 然后经过一系列转换后写入另一个 topic, 消费者可能在提交位移的过程中出现问题而导致重复消费, 也有可能生产者重复产生消息。

kafka 中的事务可以使应用程序将消费消息、生产消息、提交位移当做原子操作来处理, 同时成功或同时失败即使该生产或消费会跨多个分区。

### 2.1 配置事务功能

为了实现事务, 应用程序必须提供唯一的 transactionId, 这个 transactionId 通过客户端参数 transactional.id 来显示设置, 参考如下:

```scala
val props: Properties = new Properties
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "trans_id_0001")
```

开启事务特性会要求 KafkaProducer 开启幂等, 因此同时还要配置 enable.idempotence 设置为 true(如果未显示配置, KafkaProducer 默认会将它的值设置为 true)。

transactionId 与 PID 一一对应, 两者之间所不同的是 transactionId 由用户显示设置, 而 PID 是由 Kafka 内部配置。另外为了保证新的生产者启动后具有相同的 transactionId 的旧生产者能够立即失效, 每个生产者通过 transactionId 获取 PID 的同时, 还会获取一个单调递增的 producer_epoch (对应下面要讲的 KafkaProducer.initTransactions() 方法)。如果使用同一个 transactionId 开启两个 KafkaProducer 实例, 那么前一个开启的生产者汇报如下错误:

```java
org.apache.kafka.common.errors.ProducerFetchException: Producer attemtped an operation with an old epoch. Either there is a newer producer with the same transactionalId, or the producer's transaction hab been expired by the broker.
```

从生产者的角度分析, 通过事务kafka 可以保证跨生产者会话的消息幂等性的发送, 以及跨生产者会话的事务恢复。前者表示具有相同 transactionId 的新生产者实例被创建且工作的时候, 旧的拥有相同 transactionId 的生产者实例将不在工作。后者指当在某个生产者实例宕机后, 新的生产者实例可以保证任何未完成的旧事物要么被提交(commit), 要么被终止(Abort), 如此可以使新的生产者实例从一个正常的状态开始工作。

而从消费者的角度分析, 事务能保证的语义相对较弱。出于以下原因, kafka 并不能保证已提交的事务中的所有消息都能够被消费。

- 对采用日志压缩策略的 Topic 而言, 事务中的某些消息有可能被清理(相同 key 的消息, 后写入的消息会覆盖前面写入的消息)
- 事务的消息有可能分部在同个分区内的多个日志段(LogSegment)中, 当老的日志段被删除后, 对应的消息就会丢失
- 消费者可以通过 seek() 方法访问任意 offset 的消息, 从而可能会遗漏事务中的部分消息
- 消费者在消费时可能没有分配到事务内的所有分区, 如此的话它也就不能读取事务中的所有消息

KafkaProducer 提供了以下 5 个接口用于事务操作：

```cpp
// 初始化事务, 能够执行的前提是配置了 transactionId, 否则会报错: IllegalStateException
public void initTransactions();

// 开启事务
public void beginTransaction() throws ProducerFencedException;

// 为 KafkaConsumer 提供在事务内提交已经消费的偏移量的操作
public void sendOffsetsToTransaction(
  					Map<TopicPartition, 
  					OffsetAndMetadata> offsets, 
            String consumerGroupId) throws ProducerFencedException;

// 提交事务
public void commitTransaction() throws ProducerFencedException;

// 终止事务, 类似于事务的回滚
public void abortTransaction() throws ProducerFencedException ;
```

典型的事务消息发送的操作代码:

```scala
val props: Properties = .....
val producer: KafkaProducer[String, String] = 
														new KafkaProducer(props)
producer.initTransactions()
producer.beginTransaction()
try {
  producer.send(new ProducerRecord(topic, "msg1"))
  producer.send(new ProducerRecord(topic, "msg2"))
  producer.send(new ProducerRecord(topic, "msg3"))
  producer.commitTransaction()
} catch {
  case e: Exception => producer.abortTransaction()
}
producer.close()
```







Kafka 中的事务特性主要用于以下两种场景：

- 生产者发送多条消息可以封装在一个事务中，形成一个原子操作。多条消息要么都发送成功，要么都发送失败。
- read-process-write 模式：将消息消费和生产封装在一个事务中，形成一个原子操作。在一个流式处理的应用中，常常一个服务需要从上游接收消息，然后经过处理后送达到下游，这就对应着消息的消费和生成。

> 当事务中仅仅存在 Consumer 消费消息的操作时，它和 Consumer 手动提交 Offset并没有区别。 因此单纯的消费消息并不是 Kafka 引入事务机制的原因，单纯的消费消息也没有必要存在于一个事务中。

KafkaProducer 提供了以下 5 个接口用于事务操作：

```cpp
    /**
     * 初始化事务
     */
    public void initTransactions();
 
    /**
     * 开启事务
     */
    public void beginTransaction() throws ProducerFencedException ;
 
    /**
     * 在事务内提交已经消费的偏移量
     */
    public void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets, 
                                         String consumerGroupId) throws ProducerFencedException ;
 
    /**
     * 提交事务
     */
    public void commitTransaction() throws ProducerFencedException;
 
    /**
     * 丢弃事务
     */
    public void abortTransaction() throws ProducerFencedException ;
```

下面是使用 Kafka 事务特性的例子，这段代码 Producer 开启了一个事务，然后在这个事务中发送了两条消息。这两条消息要么都发送成功，要么都失败。

```bash
KafkaProducer producer = createKafkaProducer(
  "bootstrap.servers", "localhost:9092",
  "transactional.id”, “my-transactional-id");

producer.initTransactions();
producer.beginTransaction();
producer.send("outputTopic", "message1");
producer.send("outputTopic", "message2");
producer.commitTransaction();
```

下面这段代码即为 read-process-write 模式，在一个 Kafka 事务中，同时涉及到了生产消息和消费消息。

```csharp
KafkaProducer producer = createKafkaProducer(
  "bootstrap.servers", "localhost:9092",
  "transactional.id", "my-transactional-id");

KafkaConsumer consumer = createKafkaConsumer(
  "bootstrap.servers", "localhost:9092",
  "group.id", "my-group-id",
  "isolation.level", "read_committed");

consumer.subscribe(singleton("inputTopic"));

producer.initTransactions();

while (true) {
  ConsumerRecords records = consumer.poll(Long.MAX_VALUE);
  producer.beginTransaction();
  for (ConsumerRecord record : records)
    producer.send(producerRecord(“outputTopic”, record));
  producer.sendOffsetsToTransaction(currentOffsets(consumer), group);  
  producer.commitTransaction();
}
```

注意：在理解消息的事务时，一直处于一个错误理解是，把操作 db 的业务逻辑跟操作消息当成是一个事务，如下所示：

```cpp
void  kakfa_in_tranction(){
  // 1.kafa的操作：读取消息或生产消息
  kafkaOperation();
  // 2.db操作
  dbOperation();
}
```

其实这个是有问题的。操作 DB 数据库的数据源是 DB，消息数据源是 kafka，这是完全不同两个数据。**一种数据源（如mysql，kafka）对应一个事务，所以它们是两个独立的事务。**kafka事务指kafka一系列 生产、消费消息等操作组成一个原子操作，db 事务是指操作数据库的一系列增删改操作组成一个原子操作。

## 2. Kafka 事务配置

- 对于 Producer，需要设置 `transactional.id`属性，这个属性的作用下文会提到。设置了 `transactional.id` 属性后，`enable.idempotence` 属性会自动设置为true。
- 对于 Consumer，需要设置 `isolation.level = read_committed`，这样 Consumer 只会读取已经提交了事务的消息。另外，需要设置 `enable.auto.commit = false` 来关闭自动提交Offset功能。

> 更多关于配置的信息请参考我的文章：[Kafka消息送达语义详解](https://www.jianshu.com/p/0943bbf482e9)

## 3. Kafka 事务特性

Kafka 的事务特性本质上代表了三个功能：

- 原子写操作
- 拒绝僵尸实例（Zombie fencing）
- 读事务消息

### 3.1 原子写

Kafka 的事务特性本质上是支持了 Kafka 跨分区和 Topic 的原子写操作。在同一个事务中的消息要么同时写入成功，要么同时写入失败。我们知道，Kafka 中的 Offset 信息存储在一个名为 \_\_consumed_offsets 的 Topic 中，因此 read-process-write 模式，除了向目标 Topic写入消息，还会向 \_\_consumed_offsets 中写入已经消费的 Offsets 数据。因此 read-process-write 本质上就是跨分区和 Topic 的原子写操作。Kafka 的事务特性就是要确保跨分区的多个写操作的原子性。

### 3.2 拒绝僵尸实例（Zombie fencing）

在分布式系统中，一个 instance 的宕机或失联，集群往往会自动启动一个新的实例来代替它的工作。此时若原实例恢复了，那么集群中就产生了两个具有相同职责的实例，此时宕机后又恢复的 instance 就被称为“僵尸实例（Zombie Instance）”。在 Kafka 中，两个相同的 producer 同时处理消息并生产出重复的消息（read-process-write 模式），这样就严重违反了Exactly Once Processing的语义。这就是僵尸实例问题。

Kafka 事务特性通过 `transaction-id` 属性来解决僵尸实例问题。所有具有相同 `transaction-id` 的 Producer 都会被分配相同的 pid，同时每一个 Producer 还会被分配一个递增的epoch。Kafka 收到事务提交请求时，如果检查当前事务提交者的 epoch 不是最新的，那么就会拒绝该 Producer 的请求。从而达成拒绝僵尸实例的目标。

### 3.3 读事务消息

为了保证事务特性，Consumer如果设置了`isolation.level = read_committed`，那么它只会读取已经提交了的消息。在Producer成功提交事务后，Kafka会将所有该事务中的消息的`Transaction Marker`从`uncommitted`标记为`committed`状态，从而所有的Consumer都能够消费。

## 4. Kafka 事务原理

Kafka 为了支持事务特性，引入一个新的组件：Transaction Coordinator。主要负责分配pid，记录事务状态等操作。下面时 Kafka 开启一个事务到提交一个事务的流程图：

![img](https:////upload-images.jianshu.io/upload_images/448235-24c566fdeb21de7b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

主要分为以下步骤：

1. 查找Tranaction Corordinator

   Producer向任意一个brokers发送 FindCoordinatorRequest 请求来获取Transaction Coordinator的地址。

2. 初始化事务 initTransaction

   Producer发送InitpidRequest给Transaction Coordinator，获取pid。**Transaction Coordinator在Transaciton Log中记录这<TransactionId,pid>的映射关系。**另外，它还会做两件事：

- 恢复（Commit或Abort）之前的Producer未完成的事务
- 对PID对应的epoch进行递增，这样可以保证同一个app的不同实例对应的PID是一样，而epoch是不同的。

> 只要开启了幂等特性即必须执行InitpidRequest，而无须考虑该Producer是否开启了事务特性。

3. 开始事务 beginTransaction

   执行Producer的beginTransacion()，它的作用是Producer在**本地记录下这个transaction的状态为开始状态。**这个操作并没有通知Transaction Coordinator，因为Transaction Coordinator只有在Producer发送第一条消息后才认为事务已经开启。

4. read-process-write 流程

   一旦Producer开始发送消息，**Transaction Coordinator会将该<Transaction, Topic, Partition>存于Transaction Log内，并将其状态置为BEGIN**。另外，如果该<Topic, Partition>为该事务中第一个<Topic, Partition>，Transaction Coordinator还会启动对该事务的计时（每个事务都有自己的超时时间）。

   在注册<Transaction, Topic, Partition>到Transaction Log后，生产者发送数据，虽然没有还没有执行commit或者abort，但是此时消息已经保存到Broker上了。即使后面执行abort，消息也不会删除，只是更改状态字段标识消息为abort状态。

5. 事务提交或终结 commitTransaction/abortTransaction

   在Producer执行commitTransaction/abortTransaction时，Transaction Coordinator会执行一个两阶段提交：

   - **第一阶段，将Transaction Log内的该事务状态设置为`PREPARE_COMMIT`或`PREPARE_ABORT`**

   - **第二阶段，将`Transaction Marker`写入该事务涉及到的所有消息（即将消息标记为`committed`或`aborted`）。**这一步骤Transaction Coordinator会发送给当前事务涉及到的每个<Topic, Partition>的Leader，Broker收到该请求后，会将对应的`Transaction Marker`控制信息写入日志。

   **一旦`Transaction Marker`写入完成，Transaction Coordinator会将最终的`COMPLETE_COMMIT`或`COMPLETE_ABORT`状态写入Transaction Log中以标明该事务结束。**

## 5. 总结

- Transaction Marker与PID提供了识别消息是否应该被读取的能力，从而实现了事务的隔离性。
- Offset的更新标记了消息是否被读取，从而将对读操作的事务处理转换成了对写（Offset）操作的事务处理。
- Kafka事务的本质是，将一组写操作（如果有）对应的消息与一组读操作（如果有）对应的Offset的更新进行同样的标记（Transaction Marker）来实现事务中涉及的所有读写操作同时对外可见或同时对外不可见。
- Kafka只提供对Kafka本身的读写操作的事务性，不提供包含外部系统的事务性。



作者：伊凡的一天
链接：https://www.jianshu.com/p/64c93065473e
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。