# 【Kafka】本地安装及配置



## 准备

1. 解压 tar 包, 创建软连接及配置环境变量

   ```sh
   # 1. 解压 tar包,创建软连接
   ➜  ~ tar -zxvf /Users/sherlock/Downloads/softpack/apache soft/kafka/kafka_2.12-0.11.0.3.tgz -C /Users/sherlock/devTools/opt
   
   # 2. 创建软连接
   ➜  ~ sudo ln -s /Users/sherlock/devTools/opt/kafka-0.11.0.3 /usr/local/kafka
   
   # 3. 修改环境变量(修改 ~/.bash_profile,添加如下内容)
   export KAFKA_HOME=/usr/local/kafka
   export PATH=$KAFKA_HOME/bin:${path}
   ```

   

## 配置文件

因为本地准备模拟三台 kafka 服务器, 所以, 我们需要在本地启动三个 kafka broker, 需要准备 3 个 server.properties, 保证端口不冲突

1. ***producer.properties***

   ```
   bootstrap.servers=localhost:9092,localhost:9093,localhost:9094
   ```

2. ***consumer.properties***

   这个不用改, 如果 zookeeper 不是本地的话, 需要改 `zookeeper.connect` 这个参数

3. ***server101.properties***

   ```properties
   broker.id=101
   delete.topic.enable=true
   listeners=PLAINTEXT://localhost:9092
   # 这个配置很有迷惑性,它配置的是kafka topic中的数据的存储路径 
   log.dirs=/Users/sherlock/devTools/opt/kafka-0.11.0.3/data/broker101
   # 修改 kafka 在 zookeeper上元数据的存储目录
   zookeeper.connect=localhost:2181/kafka-0.11
   ```

4. ***server102.properties***

   ```properties
   broker.id=102
   delete.topic.enable=true
   listeners=PLAINTEXT://localhost:9093
   # 这个配置很有迷惑性,它配置的是kafka topic中的数据的存储路径 
   log.dirs=/Users/sherlock/devTools/opt/kafka-0.11.0.3/data/broker102
   # 修改 kafka 在 zookeeper上元数据的存储目录
   zookeeper.connect=localhost:2181/kafka-0.11
   ```

5. ***server103.properties***

   ```properties
   broker.id=103
   delete.topic.enable=true
   listeners=PLAINTEXT://localhost:9094
   # 这个配置很有迷惑性,它配置的是kafka topic中的数据的存储路径 
   log.dirs=/Users/sherlock/devTools/opt/kafka-0.11.0.3/data/broker103
   # 修改 kafka 在 zookeeper上元数据的存储目录
   zookeeper.connect=localhost:2181/kafka-0.11
   ```



## 服务测试

1. 前提条件，先启动zookeeper
   
   ```sh
${ZOOKEEPER_HOME}/bin/zkServer.sh start
   ```
   
   ​			
   
2. 启动broker

   ```sh
   bin/kafka-server-start.sh -daemon config/server101.properties
   bin/kafka-server-start.sh -daemon config/server102.properties
   bin/kafka-server-start.sh -daemon config/server103.properties
   ```

   

