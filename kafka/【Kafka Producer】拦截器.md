# 【Kafka Producer】拦截器

Kafka 中的拦截器（Interceptor）是 0.10.x.x 版本引入的一个功能，一共有两种：

- Kafka Producer 端的拦截器
- Kafka Consumer 端的拦截器

本篇主要讲述的是 Kafka Producer 端的拦截器，它主要用来对消息进行拦截或者修改，也可以用于 Producer 的 Callback 回调之前进行相应的预处理。

使用 Kafka Producer 端的拦截器非常简单，主要是实现 ProducerInterceptor 接口，此接口包含 4 个方法：

1. `ProducerRecord<K, V> onSend(ProducerRecord<K, V> record)`

   Producer在将消息序列化和分配分区之前会调用拦截器的这个方法来对消息进行相应的操作。一般来说最好不要修改消息 ProducerRecord 的 topic、key 以及 partition 等信息，如果要修改，也需确保对其有准确的判断，否则会与预想的效果出现偏差。比如修改 key 不仅会影响分区的计算，同样也会影响 Broker 端日志压缩（Log Compaction）的功能。

2. `void onAcknowledgement(RecordMetadata metadata, Exception exception)`

   在消息被应答（Acknowledgement）之前或者消息发送失败时调用，优先于用户设定的Callback 之前执行。这个方法运行在Producer的IO线程中，所以这个方法里实现的代码逻辑越简单越好，否则会影响消息的发送速率。

3. `void close()`

   关闭当前的拦截器，此方法主要用于执行一些资源的清理工作。

4. `configure(Map<String, ?> configs)`

   用来初始化此类的方法，这个是 ProducerInterceptor 接口的父接口 Configurable 中的方法。

一般情况下只需要关注并实现 onSend 或者 onAcknowledgement 方法即可。下面我们来举个案例，通过 onSend 方法来过滤消息体为空的消息以及通过 onAcknowledgement 方法来计算发送消息的成功率。

```java
public class ProducerInterceptorDemo implements ProducerInterceptor<String,String> {
  private volatile long sendSuccess = 0;
  private volatile long sendFailure = 0;

  @Override
  public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
    if(record.value().length()<=0)
      return null;
    return record;
  }

  @Override
  public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
    if (exception == null) {
      sendSuccess++;
    } else {
      sendFailure ++;
    }
  }

  @Override
  public void close() {
    double successRatio = (double)sendSuccess / (sendFailure + sendSuccess);
    System.out.println("[INFO] 发送成功率="+String.format("%f", successRatio * 100)+"%");
  }

  @Override
  public void configure(Map<String, ?> configs) {}
}
```

自定义的 ProducerInterceptorDemo 类实现之后就可以在 Kafka Producer 的主程序中指定，示例代码如下：

```java
public class ProducerMain {
    public static final String brokerList = "localhost:9092";
    public static final String topic = "hidden-topic";

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Properties properties = new Properties();
        properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        properties.put("bootstrap.servers", brokerList);
        properties.put("interceptor.classes", "com.hidden.producer.ProducerInterceptorDemo");

        Producer<String, String> producer = new KafkaProducer<String, String>(properties);

        for(int i=0;i<100;i++) {
            ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>(topic, "msg-" + i);
            producer.send(producerRecord).get();
        }
        producer.close();
    }
}
```

Kafka Producer 不仅可以指定一个拦截器，还可以指定多个拦截器以形成拦截链，这个拦截链会按照其中的拦截器的加入顺序一一执行。比如上面的程序多添加一个拦截器，示例如下：

```java
properties.put("interceptor.classes", "com.hidden.producer.ProducerInterceptorDemo,com.hidden.producer.ProducerInterceptorDemoPlus");
```

这样 Kafka Producer 会先执行拦截器 ProducerInterceptorDemo，之后再执行ProducerInterceptorDemoPlus。

有关 interceptor.classes 参数，在kafka 1.0.0 版本中的定义如下：

| NAME                 | DESCRIPTION                                                  | TYPE | DEFAULT | VALID VALUES | IMPORTANCE |
| :------------------- | :----------------------------------------------------------- | :--- | :------ | :----------- | :--------- |
| interceptor.calssses | A list of classes to use as interceptors. Implementing the org.apache.kafka.clients.producer.ProducerInterceptor interface allows you to intercept (and possibly mutate) the records received by the producer before they are published to the Kafka cluster. By default, there no interceptors. |      |         |              |            |