# 【Kafka】分区器

KafkaProducer 在调用 send 方法发送消息至 broker 的过程中，首先是经过拦截器Inteceptors 处理，然后是经过序列化 Serializer 处理，之后就到了 Partitions 阶段，即分区分配计算阶段。在某些应用场景下，业务逻辑需要控制每条消息落到合适的分区中，有些情形下则只要根据默认的分配规则即可。

在 KafkaProducer 计算分配时，首先根据的是 ProducerRecord 中的 partition 字段指定的序号计算分区。先看段 Kafka 生产者的示例片段：

```java
Producer<String,String> producer = new KafkaProducer<String,String>(properties);
String message = "kafka producer demo";
ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>(topic,message);
try {
  producer.send(producerRecord).get();
} catch (InterruptedException e) {
  e.printStackTrace();
} catch (ExecutionException e) {
  e.printStackTrace();
}
```

没错，ProducerRecord 只是一个封装了消息的对象而已，ProducerRecord 一共有5个成员变量，即：

```java
private final String topic;				// 所要发送的topic
private final Integer partition;	// 指定的partition序号
private final Headers headers;		// 一组键值对，与RabbitMQ中的headers类似，kafka0.11.x版本才引入的一个属性
private final K key;							// 消息的key
private final V value;						// 消息的value,即消息体
private final Long timestamp;			// 消息的时间戳，可以分为Create_Time和LogAppend_Time之分，这个以后的文章中再表。
```

在 KafkaProducer 的源码（1.0.0）中，计算分区时调用的是下面的 partition()方法：

```java
/**
 * computes partition for given record.
 * if the record has partition returns the value otherwise
 * calls configured partitioner class to compute the partition.
 */
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
  Integer partition = record.partition();
  return partition != null ?
    partition :
  partitioner.partition(record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
}
```

可以看出的确是先判断有无指明 ProducerRecord 的 partition字段，如果没有指明，则再进一步计算分区。上面这段代码中的 partitioner 在默认情况下是指 Kafka 默认实现的`org.apache.kafka.clients.producer.DefaultPartitioner`，其 partition() 方法实现如下：

```java
/**
 * Compute the partition for the given record.
 *
 * @param topic The topic name
 * @param key The key to partition on (or null if no key)
 * @param keyBytes serialized key to partition on (or null if no key)
 * @param value The value to partition on or null
 * @param valueBytes serialized value to partition on or null
 * @param cluster The current cluster metadata
 */
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
  List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
  int numPartitions = partitions.size();
  if (keyBytes == null) {
    int nextValue = nextValue(topic);
    List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
    if (availablePartitions.size() > 0) {
      int part = Utils.toPositive(nextValue) % availablePartitions.size();
      return availablePartitions.get(part).partition();
    } else {
      // no partitions are available, give a non-available partition
      return Utils.toPositive(nextValue) % numPartitions;
    }
  } else {
    // hash the keyBytes to choose a partition
    return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
  }
}

private int nextValue(String topic) {
  AtomicInteger counter = topicCounterMap.get(topic);
  if (null == counter) {
    counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
    AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
    if (currentCounter != null) {
      counter = currentCounter;
    }
  }
  return counter.getAndIncrement();
}
```

由上源码可以看出 partition 的计算方式：

1. 如果 key 为 null，则按照一种轮询的方式来计算分区分配
2. 如果 key 不为 null 则使用称之为 murmur 的 Hash算法（非加密型Hash函数，具备高运算性能及低碰撞率）来计算分区分配。

KafkaProducer 中还支持自定义分区分配方式，与org.apache.kafka.clients.producer.internals.DefaultPartitioner 一样首先实现org.apache.kafka.clients.producer.Partitioner 接口，然后在 KafkaProducer 的配置中指定 partitioner.class 为对应的自定义分区器（Partitioners）即可，即：

```java
properties.put("partitioner.class","com.hidden.partitioner.DemoPartitioner");
```

自定义 DemoPartitioner 主要是实现 Partitioner 接口的 `public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster)`的方法。DemoPartitioner 稍微修改了下DefaultPartitioner 的计算方式，详细参考如下：

```java
public class DemoPartitioner implements Partitioner {

  private final AtomicInteger atomicInteger = new AtomicInteger(0);

  @Override
  public void configure(Map<String, ?> configs) {}

  @Override
  public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();
    if (null == keyBytes || keyBytes.length<1) {
      return atomicInteger.getAndIncrement() % numPartitions;
    }
    //借用String的hashCode的计算方式
    int hash = 0;
    for (byte b : keyBytes) {
      hash = 31 * hash + b;
    }
    return hash % numPartitions;
  }

  @Override
  public void close() {}
}
```

这个自定义分区器的实现比较简单，读者可以根据自身业务的需求来灵活实现分配分区的计算方式，比如：一般大型电商都有多个仓库，可以将仓库的名称或者ID作为Key来灵活的记录商品信息。

