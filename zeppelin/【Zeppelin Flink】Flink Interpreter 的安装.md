# 【Zeppelin Flink】Flink Interpreter 的安装

坑爹的来了, 在安装完 Zeppelin 后, 按照页面提示, 只需要下载 scala-2.11版本的 Flink-1.10, 然后配置 interpreter 就可以. 我确实也是按照这么来的, 但是查看日志一直报错, interpreter process is not running.

最后才知道需要引入依赖包到 $ZEPPELIN_HOME/lib 目录下.

> Tips: 目前 zeppelin-0.9 还不支持 scala-2.12 版本的 Flink

## 1. 前提准备

首先确保 Zeppelin 已经安装完成.

下面很重要!很重要!很重要!

### 1.1 下载两个插件包:

1. `flink-hadoop-compatibility_2.11-1.10.2.jar` 

   我本地有两个版本的flink(1.10和1.11), zeppelin 使用的是 1.10, 所以上面包的版本一定要正确. [下载地址](https://repo1.maven.org/maven2/org/apache/flink/flink-hadoop-compatibility_2.11/1.10.2/), 选择对应本地 flink 的版本下载

2. `flink-shaded-hadoop-2-uber-2.7.5-7.0.jar`

   [下载地址](https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.7.5-7.0/), 暂时没有搞懂后面 2.7.5-7.0 的意思......

下载完成之后, 将这两个 jar 包 copy 到 $zeppelin_home/lib 目录下.

### 1.2 本地安装 Flink

这个不多说了, 下载对应上面的 flink, 比如上面使用的是 flink-1.10.2-bin-scala_2.11.tgz, 下载解压就 OK!



## 2. Flink on Zeppelin Local 模式配置

上面步骤都 OK 后, 登录 zeppelin(http:localhost:18080).

Flink的Local模式会在本地创建一个MiniCluster，适合做POC或者小数据量的试验。必须配置**FLINK_HOME** 和 **flink.execution.mode**。

当我们只用 zeppelin 连接到 Flink 的 local时，只需要配置 Flink Interpreter的以下两个参数即可

必须配置**FLINK_HOME** 和 **flink.execution.mode**

如下图所示：


![img](https://img-blog.csdnimg.cn/20201219175802411.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpdGxpdDAyMw==,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20201220195605627.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpdGxpdDAyMw==,size_16,color_FFFFFF,t_70)

修改完成, 保存. 

Batch 测试, 运行 Flink Tutorial/Flink Basics/Batch WordCount 示例.

Stream 测试, 运行 Flink Tutorial/Flink Basics/Streaming WordCount 示例

![](https://img-blog.csdnimg.cn/20201220195843598.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpdGxpdDAyMw==,size_16,color_FFFFFF,t_70)

观察Zeppelin的运行日志，运行过上述两个demo后，zeppelin会在log目录下生成一个新的日志文件，如下图所示，里面便是运行上述demo的详细日志

![img](https://img-blog.csdnimg.cn/20201220200008822.png)

 

至此，Flink on Zeppelin的Local模式已经搭建成功，这里建议按照文中给出的软件版本操作，即可成功（当时由于软件之间版本兼容问题，走了一些弯路。。。）



## 3. Flink on Zeppelin Remote 模式配置

Flink on Zeppelin环境的Remote模式，与Local模式基本是一样的，只是Remote模式需要额外启动一个Flink集群，并设置以下4个参数即可：

- FLINK_HOME   

  这还是配置你本地 Flink path, 它是作为客户端

- flink.execution.mode    

  必须填 remote       

- flink.execution.remote.host

  这个需要你提前启动一个远端上的 Flink 集群, 这个参数就是填写远端 Flink集群的 JobManager 的主机地址

- flink.execution.remote.port

  这个参数就是填写远端 Flink集群的 JobManager 的主机通信端口



## 4. Flink on Zeppelin Yarn 模式 peizhi

Flink的Yarn模式会在Yarn集群中动态创建一个Flink Cluster，然后你就可以往这个Flink Session Cluster提交Flink Job了。

> 这里要注意, zeppelin 创建的是 session mode 的 flink集群. 如果你提交一个 Job 后打开 yarn 的 webUI(http://localhost:8088), 可以在 Applications 中看到一个 Running 状态的yarn job(Zeppelin Flink Session)

- 除了配置 FLINK_HOME 和 flink.execution.mode 外还需要配置 HADOOP_CONF_DIR，并且要确保 Zeppelin 这台机器可以访问你的 hadoop 集群, 你可以运行hadoop fs 命令来验证是否能够连接hadoop集群。

  ![image.png](https://cdn.nlark.com/yuque/0/2020/png/2211963/1596425799170-04dab326-8f38-4f97-9db4-5c175f26e5d7.png?x-oss-process=image%2Fresize%2Cw_1500)



- 确保Zeppelin这台机器上安装了 hadoop 客户端（就是hadoop的安装包），并且hadoop命令在PATH上。因为Flink本身不带hadoop相关依赖（你可以把flink-shaded-hadoop 拷贝到flink lib下，但这种方式不推荐），所以flink的推荐方式是在提交Flink作业的时候运行hadoop classpath命令，然后把hadoop相关的jar放到CLASSPATH上。

  > 如果正确安装了hadoop, 可以使用命令 `hadoop classpath` 查看hadoop的clasppath
  >
  > ```sh
  > ➜ datas hadoop classpath
  > /usr/local/hadoop/etc/hadoop:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/common/lib/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/common/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/hdfs:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/hdfs/lib/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/hdfs/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/yarn/lib/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/yarn/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/mapreduce/lib/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/mapreduce/*:/usr/local/hadoop/etc/hadoop:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/common/lib/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/common/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/hdfs:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/hdfs/lib/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/hdfs/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/yarn/lib/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/yarn/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/mapreduce/lib/*:/Users/sherlock/devTools/opt/hadoop-2.7.7/share/hadoop/mapreduce/*:/usr/local/hadoop/contrib/capacity-scheduler/*.jar:/usr/local/hadoop/contrib/capacity-scheduler/*.jar
  > ```
  >
  > 之本地环境变量中配置 `export HADOOP_CLASSPATH=$(hadoop classpath)`

flink bacth 读取 hadoop 文件测试:

![flinkOnZepYarn](../allPics/Flink/flinkOnZepYarn.png)