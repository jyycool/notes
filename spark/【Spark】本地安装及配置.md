# 【Spark】本地安装及配置

spark-shell 也炸了, 好像是因为spark绑定了 hadoop-2.6.0, 有 bug, 算了反正因为 hadoop 已经升级到 2.7.7 了所以这次 spark 也就重新安装啦...

本次安装的是 spark-2.3.4-bin-hadoop2.7.tgz

### 1. 解压 spark-2.4.7.tgz

```sh
➜  opt tar -zxvf ~/Downloads/spark-2.3.4-bin-hadoop2.7.tgz -C ~/devTools/opt/

# 修改文件夹名字为 spark-2.4.7
➜  opt mv spark-2.3.4-bin-hadoop2.7 spark-2.3.4
```



### 2. 修改环境变量

***SPARK_HOME=/Users/sherlock/devTools/opt/spark-2.4.7***

```sh
➜  spark-2.4.7 vi ~/.bash_profile
```

添加如下内容

```sh
export SPARK_HOME=/Users/sherlock/devTools/opt/spark-2.4.7
export PATH=$PATH:$SPARK_HOME/bin
```

在`sbin/spark-config.sh`文件中添加:

```sh
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
```



### 3. 本地 Local 测试

```sh
➜  spark-2.3.4 spark-submit \
--class org.apache.spark.examples.SparkPi \
--master local[/yarn/spark://localhost:7077] \
--executor-memory 1G \
--total-executor-cores 2 \
./examples/jars/spark-examples_2.11-2.3.4.jar \
50
```

执行结果:

```
2020-10-15 12:08:51 INFO  DAGScheduler:54 - Job 0 finished: reduce at SparkPi.scala:38, took 3.129657 s
Pi is roughly 3.1415499141549916
```



### 4. 各模式配置

目前 Apache Spark 支持三种分布式部署方式，分别是 standalone、spark on mesos和 spark on  YARN，其中，第一种类似于MapReduce  1.0所采用的模式，内部实现了容错性和资源管理，后两种则是未来发展的趋势，部分容错性和资源管理交由统一的资源管理系统完成：让Spark运行在一个通用的资源管理系统之上，这样可以与其他计算框架，比如MapReduce，公用一个集群资源，最大的好处是降低运维成本和提高资源利用率（资源按需分配）。本文将介绍这三种部署方式，并比较其优缺点。 

#### 4.1 standalone 模式

 即独立模式，自带完整的服务，可单独部署到一个集群中，无需依赖任何其他资源管理系统。从一定程度上说，该模式是其他两种的基础。借鉴Spark开发模式，我们可以得到一种开发新型计算框架的一般思路：先设计出它的standalone模式，为了快速开发，起初不需要考虑服务（比如master/slave）的容错性，之后再开发相应的wrapper，将stanlone模式下的服务原封不动的部署到资源管理系统yarn或者mesos上，由资源管理系统负责服务本身的容错。目前Spark在standalone模式下是没有任何单点故障问题的，这是借助zookeeper实现的，思想类似于Hbase master单点故障解决方案。将Spark standalone与MapReduce比较，会发现它们两个在架构上是完全一致的： 

1. 都是由master/slaves服务组成的，且起初 maste r均存在单点故障，后来均通过zookeeper 解决（Apache MRv1的 JobTracker仍存在单点问题，但CDH版本得到了解决）； 
2. 各个节点上的资源被抽象成粗粒度的 slot，有多少 slot 就能同时运行多少 task。不同的是，MapReduce 将 slot 分为 map slot 和 reduce slot，它们分别只能供 Map Task 和Reduce  Task 使用，而不能共享，这是 MapReduce 资源利率低效的原因之一，而Spark则更优化一些，它不区分 slot 类型，只有一种 slot，可以供各种类型的 Task 使用，这种方式可以提高资源利用率，但是不够灵活，不能为不同类型的 Task 定制 slot 资源。总之，这两种方式各有优缺点。 

#### 4.1.1 SA 之本地 Local 模式

1. 修改 slavers 文件, 添加如下内容:

   ```
   localhost
   ```

2. 修改 spark-env.sh, 添加如下内容:

   ```sh
   #=========================
   # standalone Mode (Local) 
   #=========================
   SPARK_MASTER_HOST=localhost
   SPARK_MASTER_PORT=7077
   ```



#### 4.1.2 SA 之 HA 模式

 1. 修改 spark-env.sh, 添加如下内容:

    ```sh
    # 注释掉下面的 Local 配置后, 添加 zk 配置
    #=========================
    # standalone Mode (Local) 
    #=========================
    #SPARK_MASTER_HOST=localhost
    #SPARK_MASTER_PORT=7077
    
    #==============================
    # standalone HA Mode (Local)
    #==============================
    export SPARK_DAEMON_JAVA_OPTS="
    -Dspark.deploy.recoveryMode=ZOOKEEPER
    -Dspark.deploy.zookeeper.url=localhost
    -Dspark.deploy.zookeeper.dir=/spark
    "
    ```



#### 4.1.3 开启历史任务日志

1. 修改 spark-env.sh, 添加如下内容:

   ```sh
   #========================================================
   # 配置 spark History Server
   #
   # spark.history.ui.port                 服务的 web ui 端口号
   # spark.history.retainedApplications    指定保存 App 历史记录的个数
   # spark.history.fs.logDirectory         任务历史保存位置, 配置后在 start-history-server.sh 中无需显式的指定路径.
   #========================================================
   export SPARK_HISTORY_OPTS="
   -Dspark.history.ui.port=18080
   -Dspark.history.retainedApplications=30
   -Dspark.history.fs.logDirectory=hdfs://localhost:9000/logs/spark
   "
   ```

2. 修改 spark-default.conf,  添加如下内容

   ```sh
   #========================================= 
   # standalone Mode
   #            启用任务历史日志, 及日志存储位置
   #=========================================
   spark.eventLog.enabled	true
   spark.eventLog.dir		  hdfs://localhost:9000/logs/spark
   ```

   

#### 4.1.4 yarn 模式

1. 修改 spark-env.sh, 添加如下内容:

   ```sh
   #注释掉下面的 Local 和 HA 配置后, 添加 yarn 配置
   #=========================
   # standalone Mode (Local) 
   #=========================
   #SPARK_MASTER_HOST=localhost
   #SPARK_MASTER_PORT=7077
   
   #==============================
   # standalone HA Mode (Local)
   #==============================
   #export SPARK_DAEMON_JAVA_OPTS="
   #-Dspark.deploy.recoveryMode=ZOOKEEPER
   #-Dspark.deploy.zookeeper.url=localhost
   #-Dspark.deploy.zookeeper.dir=/spark
   #"
   
   #=============================
   # yarn Mode
   #=============================
   YARN_CONF_DIR=/Users/sherlock/devTools/opt/hadoop-2.7.7/etc/hadoop
   ```

2. 开启 yarn 追踪 spark 历史日志 (将 spark historyserver 转发到 yarn 的 historyserver 端口中)

   修改 spark-env.sh, 添加如下内容:

   ```conf
   #=========================================
   # yarn Mode
   #      开启任务历史日志     
   #=========================================
   spark.yarn.historyServer.address    localhost:18080
   spark.history.ui.propt              18080
   ```

   

