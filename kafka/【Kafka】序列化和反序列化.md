# 【Kafka】序列化和反序列化

## 一、Producer 端的序列化

Kafka Producer 在发送消息时必须配置的参数为：bootstrap.servers、key.serializer、value.serializer。

序列化操作是在拦截器（Interceptor）执行之后并且在分配分区(partitions)之前执行的。

首先我们通过一段示例代码来看下普通情况下Kafka Producer如何编写：

```java
public class ProducerJavaDemo {
  public static final String brokerList = "192.168.0.2:9092,192.168.0.3:9092,192.168.0.4:9092";
  public static final String topic = "hidden-topic";

  public static void main(String[] args) {
    Properties properties = new Properties();
    properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    properties.put("client.id", "hidden-producer-client-id-1");
    properties.put("bootstrap.servers", brokerList);

    Producer<String,String> producer = new KafkaProducer<String,String>(properties);

    while (true) {
      String message = "kafka_message-" + new Date().getTime() + "-edited by hidden.zhu";
      ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>(topic,message);
      try {
        Future<RecordMetadata> future =  producer.send(producerRecord, new Callback() {
          public void onCompletion(RecordMetadata metadata, Exception exception) {
            System.out.print(metadata.offset()+"    ");
            System.out.print(metadata.topic()+"    ");
            System.out.println(metadata.partition());
          }
        });
      } catch (Exception e) {
        e.printStackTrace();
      }
      try {
        TimeUnit.MILLISECONDS.sleep(10);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }
}
```

这里采用的客户端不是 0.8.x.x 时代的Scala版本，而是 Java 编写的新 Kafka Producer, 相应的 Maven 依赖如下：

```
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>1.0.0</version>
</dependency>
```

上面的程序中使用的是 Kafka 客户端自带的`org.apache.kafka.common.serialization.StringSerializer`，除了用于 String 类型的序列化器之外还有：ByteArray、ByteBuffer、Bytes、Double、Integer、Long这几种类型，它们都实现了 `org.apache.kafka.common.serialization.Serializer` 接口，此接口有三种方法：

1. public void configure(Map<String, ?> configs, boolean isKey)

   用来配置当前类。

2. public byte[] serialize(String topic, T data)

   用来执行序列化。

3. public void close()

   用来关闭当前序列化器。一般情况下这个方法都是个空方法，如果实现了此方法，必须确保此方法的幂等性，因为这个方法很可能会被 KafkaProducer 调用多次。

下面我们来看看 Kafka 中 org.apache.kafka.common.serialization.StringSerializer 的具体实现，源码如下：

```java
public class StringSerializer implements Serializer<String> {
  private String encoding = "UTF8";

  @Override
  public void configure(Map<String, ?> configs, boolean isKey) {
    String propertyName = isKey ? "key.serializer.encoding" : "value.serializer.encoding";
    Object encodingValue = configs.get(propertyName);
    if (encodingValue == null)
      encodingValue = configs.get("serializer.encoding");
    if (encodingValue != null && encodingValue instanceof String)
      encoding = (String) encodingValue;
  }

  @Override
  public byte[] serialize(String topic, String data) {
    try {
      if (data == null)
        return null;
      else
        return data.getBytes(encoding);
    } catch (UnsupportedEncodingException e) {
      throw new SerializationException("Error when serializing string to byte[] due to unsupported encoding " + encoding);
    }
  }

  @Override
  public void close() {
    // nothing to do
  }
}
```

首先看下 StringSerializer#configure(Map<String, ?> configs, boolean isKey)方法，这个方法的执行是在创建 KafkaProducer 实例的时候调用的，即执行代码Producer<String,String> producer = new KafkaProducer<String,String>(properties)时调用，主要用来确定编码类型，不过一般 key.serializer.encoding 或serializer.encoding 都不会配置，更确切的来说在 Kafka Producer Configs 列表里都没有此项，所以一般情况下 encoding 的值就是 UTF-8。serialize(String topic, String data) 方法非常的直观，就是将 String 类型的数据转为 byte[] 类型即可。

如果 Kafka 自身提供的诸如 String、ByteArray、ByteBuffer、Bytes、Double、Integer、Long 这些类型的 Serializer 都不能满足需求，读者可以选择使用如 Avro、JSON、Thrift、ProtoBuf 或者 Protostuff 等通用的序列化工具来实现，亦或者是使用自定义类型的Serializer 来实现。下面就以一个简单的例子来介绍下如何自定义类型的使用方法。

假设我们要发送的消息都是 Company 对象，这个 Company 的定义很简单，只有名称 name 和地址 address，具体如下：

```java
public class Company {
  private String name;
  private String address;
  //省略Getter, Setter, Constructor & toString方法
}
```

接下去我们来实现 Company 类型的 Serializer，即下面代码示例中的 DemoSerializer。

```java
package com.hidden.client;
public class DemoSerializer implements Serializer<Company> {
  public void configure(Map<String, ?> configs, boolean isKey) {}
  public byte[] serialize(String topic, Company data) {
    if (data == null) {
      return null;
    }
    byte[] name, address;
    try {
      if (data.getName() != null) {
        name = data.getName().getBytes("UTF-8");
      } else {
        name = new byte[0];
      }
      if (data.getAddress() != null) {
        address = data.getAddress().getBytes("UTF-8");
      } else {
        address = new byte[0];
      }
      ByteBuffer buffer = ByteBuffer.allocate(4+4+name.length + address.length);
      buffer.putInt(name.length);
      buffer.put(name);
      buffer.putInt(address.length);
      buffer.put(address);
      return buffer.array();
    } catch (UnsupportedEncodingException e) {
      e.printStackTrace();
    }
    return new byte[0];
  }
  public void close() {}
}
```

使用时只需要在 Kafka Producer 的 config 中修改 value.serializer 属性即可，示例如下：

```java
properties.put("value.serializer", "com.hidden.client.DemoSerializer");
//记得也要将相应的String类型改为Company类型，如：
// Producer<String,Company> producer = new KafkaProducer<String,Company>(properties);
// Company company = new Company();
// company.setName("hidden.cooperation-" + new Date().getTime());
// company.setAddress("Shanghai, China");
// ProducerRecord<String, Company> producerRecord = new ProducerRecord<String, Company>(topic,company);
```

示例中只修改了 value.serializer,而 key.serializer 和 value.serializer 没有什么区别，如果有真实需要，修改以下也未尝不可。

## 二、Consumer 端的反序列化

有序列化就会有反序列化，反序列化的操作是在 Kafka Consumer 中完成的，使用起来只需要配置一下 key.deserializer 和 value.deseriaizer。对应上面自定义的 Company 类型的Deserializer 就需要实现 org.apache.kafka.common.serialization.Deserializer 接口，这个接口同样有三个方法：

1. `public void configure(Map<String, ?> configs, boolean isKey)`

   用来配置当前类。

2. `public byte[] serialize(String topic, T data)`

   用来执行反序列化。如果data为null建议处理的时候直接返回null而不是抛出一个异常。

3. `public void close()`

   用来关闭当前序列化器。

下面就来看一下 DemoSerializer 对应的反序列化的 DemoDeserializer，详细代码如下：

```java
public class DemoDeserializer implements Deserializer<Company> {
  public void configure(Map<String, ?> configs, boolean isKey) {}
  public Company deserialize(String topic, byte[] data) {
    if (data == null) {
      return null;
    }
    if (data.length < 8) {
      throw new SerializationException("Size of data received by DemoDeserializer is shorter than expected!");
    }
    ByteBuffer buffer = ByteBuffer.wrap(data);
    int nameLen, addressLen;
    String name, address;
    nameLen = buffer.getInt();
    byte[] nameBytes = new byte[nameLen];
    buffer.get(nameBytes);
    addressLen = buffer.getInt();
    byte[] addressBytes = new byte[addressLen];
    buffer.get(addressBytes);
    try {
      name = new String(nameBytes, "UTF-8");
      address = new String(addressBytes, "UTF-8");
    } catch (UnsupportedEncodingException e) {
      throw new SerializationException("Error occur when deserializing!");
    }
    return new Company(name,address);
  }
  public void close() {}
}
```

有些读者可能对新版的 Consumer 不是很熟悉，这里顺带着举一个完整的消费示例，并以DemoDeserializer 作为消息 Value 的反序列化器。

```java
Properties properties = new Properties();
properties.put("bootstrap.servers", brokerList);
properties.put("group.id", consumerGroup);
properties.put("session.timeout.ms", 10000);
properties.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
properties.put("value.deserializer", "com.hidden.client.DemoDeserializer");
properties.put("client.id", "hidden-consumer-client-id-zzh-2");
KafkaConsumer<String, Company> consumer = new KafkaConsumer<String, Company>(properties);
consumer.subscribe(Arrays.asList(topic));
try {
  while (true) {
    ConsumerRecords<String, Company> records = consumer.poll(100);
    for (ConsumerRecord<String, Company> record : records) {
      String info = String.format("topic=%s, partition=%s, offset=%d, consumer=%s, country=%s",
                                  record.topic(), record.partition(), record.offset(), record.key(), record.value());
      System.out.println(info);
    }
    consumer.commitAsync(new OffsetCommitCallback() {
      public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {
        if (exception != null) {
          String error = String.format("Commit failed for offsets {}", offsets, exception);
          System.out.println(error);
        }
      }
    });
  }
} finally {
  consumer.close();
}
```

有些时候自定义的类型还可以和 Avro、ProtoBuf 等联合使用，而且这样更加的方便快捷，比如我们将前面 Company 的 Serializer 和 Deserializer 用 Protostuff包装一下，由于篇幅限制，笔者这里只罗列出对应的 serialize 和 deserialize 方法，详细参考如下：

```java
public byte[] serialize(String topic, Company data) {
  if (data == null) {
    return null;
  }
  Schema schema = (Schema) RuntimeSchema.getSchema(data.getClass());
  LinkedBuffer buffer = LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE);
  byte[] protostuff = null;
  try {
    protostuff = ProtostuffIOUtil.toByteArray(data, schema, buffer);
  } catch (Exception e) {
    throw new IllegalStateException(e.getMessage(), e);
  } finally {
    buffer.clear();
  }
  return protostuff;
}

public Company deserialize(String topic, byte[] data) {
  if (data == null) {
    return null;
  }
  Schema schema = RuntimeSchema.getSchema(Company.class);
  Company ans = new Company();
  ProtostuffIOUtil.mergeFrom(data, ans, schema);
  return ans;
}
```

如果 Company 的字段很多，我们使用 Protostuff 进一步封装一下的方式就显得简洁很多。不过这个不是最主要的，而最主要的是经过 Protostuff 包装之后，这个 Serializer 和Deserializer 可以向前兼容（新加字段采用默认值）和向后兼容（忽略新加字段），这个特性Avro 和 Protobuf 也都具备。

自定义的类型有一个不得不面对的问题就是 Kafka Producer 和 Kafka Consumer 之间的序列化和反序列化的兼容性，试想对于 StringSerializer 来说，Kafka Consumer 可以顺其自然的采用 StringDeserializer，不过对于 Company 这种专用类型，某个服务使用DemoSerializer 进行了序列化之后，那么下游的消费者服务必须也要实现对应的DemoDeserializer。再者，如果上游的 Company 类型改变，下游也需要跟着重新实现一个新的DemoSerializer，这个后面所面临的难题可想而知。所以，如无特殊需要，笔者不建议使用自定义的序列化和反序列化器；如有业务需要，也要使用通用的 Avro、Protobuf、Protostuff 等序列化工具包装，尽可能的实现得更加通用且向前后兼容。

题外话，对于 Kafka 的“深耕者”Confluent来说，还有其自身的一套序列化和反序列化解决方案（io.confluent.kafka.serializer.KafkaAvroSerializer），GitHub 上有相关资料，读者如有兴趣可以自行扩展学习。