# 【Kafka】常用指令大全

> TIPS:
>
> 本地安装了两个版本的 kafka, 分别是 kafka-0.11 和 kafka-2.4, 它们在 zookeeper 上存储元数据的目录不是默认的目录, 分别是 /kafka-0.11 和 /kafka-2.4, 所以下面所有命令参数涉及到 zookeeper 的其值一定要指定对应元数据目录。

## 一、topics

1. create topic

   ```sh
   kafka-topics.sh --create --zookeeper localhost:2181/kafka-2.4 --partitions 2 --replication-factor 2 --topic first
   ```

2. list topics

   ```sh
   # 旧版本(0.9 版本以下)命令
   kafka-topics.sh --list --zookeeper localhost:2181/kafka-2.4
   
   # 支持 0.9 及更新的版本
   kafka-topics.sh --list --bootstrap-server localhost:9092
   ```

3. delete topic

   ```sh
   kafka-topics.sh --delete --zookeeper localhost:2181/kafka-2.4 --topic first
   ```

4. desc topic

   ```sh
   kafka-topics.sh --describe --zookeeper localhost:2181/kafka-2.4
   ```

## 二、producer & consumer

4. 新消费者列表查询

   ```sh
   # 支持 0.9 版本+
   kafka-consumer-groups.sh --new-consumer --bootstrap-server localhost:9092 --list
   
   # 支持 0.10 版本+
   kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
   ```

5. 显示指定消费组的消费详情

   ```sh
   # 仅支持 offset 存储在 zookeeper 上的
   kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper localhost:2181/kafka-2.4 --group test
   
   # 仅支持版本 0.9 ~ 0.10.1.0
   kafka-consumer-groups.sh --new-consumer --bootstrap-server localhost:9092 --describe --group test-consumer-group
   
   # 支持版本 0.10.1.0+
   kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group
   ```

6. console consumer

   ```sh
   # 默认从最新 offset 处开始消费
   kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic first
   
   # 从 offset=0 处开始重新消费
   kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic first
   
   # 仅支持版本 0.9+
   # 且需要将 config/producer.properties 和 config/consumer.properties 中的 group.id 设置成一样的。
   kafka-console-consumer.sh --bootstrap-server localhost:9092 --new-consumer --from-beginning --consumer.config config/consumer.properties --topic test 
   
   # 高级点的用法
   kafka-simple-consumer-shell.sh --brist localhost:9092 --topic test --partition 0 --offset 1234  --max-messages 10
   ```


7. console producer

   ```sh
   kafka-console-producer.sh --broker-list localhost:9092 --topic first
   ```

8. rebalance leader

   ```sh
   kafka-preferred-replica-election.sh --zookeeper zk_host:port/chroot
   ```

   

