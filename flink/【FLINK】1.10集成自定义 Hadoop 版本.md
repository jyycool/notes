## 【FLINK】1.10集成自定义 Hadoop 版本

flink-1.10版本中默认情况下是没有集成hadoop的，官网给的原因希望用户自己去集成hadoop、hbase等等。这样flink本身就减少Jar包依赖产生的冲突。依赖冲突交给用户来解决。官网给出flink与hadoop集成的两种方式，

### 与hadoop依赖

#### Adding Hadoop Classpaths

在flink启动时，将机器上hadoop的jar包添加到flink classpath上。

***export HADOOP_CLASSPATH=hadoop classpath***

这种方式是flink推荐方式，但容易引发jar包冲突的问题。原因hadoop依赖中有太多太多的包了，很容易出现问题。

#### 将maven-shaded包放入 flink lib下。

此种方式能够最大限度避免jar包冲突带来的问题。目前flink-1.5.1(blink)之前好像都是采用这种方式。flink官网目前只支持hadoop 几个版本，所以如果不在flink官网发布的hadoop-shaded包的话，那你只能自己下载源码，maven编译打包了。本次安装采用此种方式。使用的hadoop版本2.6.5。直接下载https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.6.5-10.0/flink-shaded-hadoop-2-uber-2.6.5-10.0.jar

***但是官网上整合的 Hadoop 版本有限, 可以编译打包自己需要的 Hadoop 版本的 flink-shaded-hadoop.jar***

### 测试成功标志

未添加hadoop依赖

***bin/flink run -h 显示***

***"run" action options: ......***

***Options for executor mode: ......***

***Options for default mode: ......***



#### 添加hadoop依赖

***bin/flink run -h 显示***

***"run" action options: ......***

***Options for yarn-cluster mode: ......***

***Options for executor mode: ......***

***Options for default mode: ......***

可以看到多了一个yarn-cluster mode.

执行测试用例：./bin/flink run -e yarn-per-job ./examples/batch/WordCount.jar

