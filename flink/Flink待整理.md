# Flink架构

## 一、Flink的架构简介

### 1.1 Flink分布式运行环境

![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/processes.svg)

关于架构，先上一个官方的图，尽管对于Flink架构，上图不是很准确（比如client与JobManager的通讯已经改为REST方式， 而非AKKA的actor system），我们还是可以知道一些要点:

- FlinkCluster

  Flink的分布式运行环境是由一个(Active) JobManager 和多个 TaskManager 组成。每一个 JobManager(JM) 或 TaskManager(TM) 都运行在一个独立的 JVM 里，他们之间通过 AKKA 的 Actor system (建立的RPC service)通讯。

  所有的 TaskManager 都由 JobManager 管理，Flink distributed runtime 实际上是一个没有硬件资源管理的软件集群 (FlinkCluster)，JM 是这个 FlinkCluster 的 master， TM 是 worker。

  所以将 Flink 运行在真正的 cluster 环境里(能够动态分配硬件资源的 cluster，比如 Yarn, Mesos, kubernetes)，只需要将 JM 和 TM 运行在这些集群资源管理器分配的容器里，配置网络环境和集群服务使 AKKA 能工作起来，Flink cluster 看起来就可以工作了。

  具体的关于怎么将 Flink 部署到不同的环境，之后有介绍，虽然没有上面说的这么简单，还有一些额外的工作，不过大概就是这样: 因为 Flink runtime 自身已经通过 AKKA 的 sharding cluster 建立了 FlinkCluster, 部署到外围的集群管理只是为了获取硬件资源服务。

  Flink 不是搭建在零基础上的框架，任何功能都要自己重新孵化，实际上它使用了大量的优秀的开源框架，比如用 AKKA 实现软件集群及远程方法调用服务(RPC)，用 ZooKeeper 提高 JM 的高可用性，用 HDFS，S3, RocksDB 永久存储数据， 用 Yarn/Mesos/Kubernetes 做容器资管理，用 Netty 做高速数据流传输等等。

- JobManager

  JobManager 作为 FlinkCluster 的 manager，它是由一些 Services 组成的，有的service 接受从 flink client 端提交的 Dataflow Graph(JobGraph)，并将 JobGraph schedule 到 TaskManager 里运行，有的 Service 协调做每个 operator 的 checkpoint以备 job graph 运行失败后及时恢复到失败前的现场从而继续运行，有的 Service 负责资源管理，有的 service 负责高可用性，后面在详细介绍。值得一提的是，集群里有且只有一个工作的JM, 它会对每一个 job 实例化一个 Job

- TaskManager

  TaskManager 是 slot 的提供者和 sub task 的执行者。通常 Flink Cluster 里会有多个TM, 每个 TM 都拥有能够同时运行多个 SubTask 的限额, Flink 称之为 TaskSlot。

  当 TM 启动后, TM 将 slot 注册到 Cluster 中的 JM 的 ResourceManager(RM), RM 从而知道 Cluster 中的 slot 总量，并要求 TM 将一定数量的 slot 提供给 JM, 从而 JM 可以将 Dataflow Graph 的 task（sub task）分配给 TM 去执行。TM 是运行并行子任务(sub Task) 的载体(一个 Job workflow 需要分解成很多 task, 每一个 task 分解成一个过多个并行子任务 sub Task), TM 需要把这些 sub task 在自己的进程空间里运行起来, 而且负责传递他们之间的输入输出数据, 这些数据包括是本地Task 的和运行在另一个 TM 里的远程 Task 。关于如何具体 excute Tasks 和交换数据后面介绍。

- Client

  Client端(Flink Program)通过 invoke 用户jar文件(flink run 中提供的 jar file)的里 main 函数(注册 data source, apply operators, 注册 data sink, apply data sink), 从而在 ExecutionEnvironment 或 StreamingExecutionEnvironment 里建立sink operator 为根的一个或多个 FlinkPlan(以 sink 为根, source 为叶子, 其他operator 为中间节点的树状结构), 之后 client 用 Flink-Optimizer 将 Plan 优化成OptimimizedPlan(根据 Cost estimator 计算出来的 cost 优化 operator 在树中的原始顺序, 同时加入了Operator与Operator连接的边, 并根据规则设置每个边的 shipingStrategy, 实际上OptimizedPlan已经从一个树结构转换成一个图结构), 之后使用GraphGenerator (或 StreamingGraphGenerator)将 OptimizedPlan 转化成 JobGraph提交给 JobManager, 这个提交是通过 JM 的 DispatcherRestEndPoint 提交的。

- Communication

  JobManager 与 Taskanager 都是 AKKA cluster 里的注册的 actor, 他们之间很容易通过AKKA(实现的 RPCService)通讯。client 与 JobManager 在以前(Version 1.4及以前)也是通过 AKKA(实现的 RPCService)通讯的, 但 Version1.5及以后版本的 JobManager 里引入DispatcherRestEndPoint(目的是使 Client 请求可以在穿过 Firewall ?), 从此 client端与 JobManager 提供的 REST EndPoint 通讯。Task 与 Task 之间的数据(data stream records)(比如一个reduce task 的 input 来自与 graph 上前一个 map, output 给graph上的另一个 map), 如果这两个 Task 运行在不同的 TM 上, 数据是通过由 TM 上的channel manager 管理的 tcp channels 传递的。

### 1.2 JobManager

![](https://img2018.cnblogs.com/blog/1492000/201906/1492000-20190614151606518-961814249.png)

如上一章所述, JobManager 是一个单独的进程（JVM), 它是一个 Flink Cluster 的 master(中心和大脑)。

它由一堆 services 组成（主要是 Dispather, JobMaster 和 ResourceManager), 连接cluster里其他分布式组件(TaskManager, client 及其他外部组件), 指挥、获得协助、或提供服务。

ClusterEntrypoint 是 JobManager 的入口, 它有一个 main 方法, 用来启动 HearBeatService, HA Service, BlobServer, DispatherRESTEndPoint, Dispather, ResourceManager。不同的 FlinkCluster 有不同的 ClusterEntryPoint 的子类, 用于启动这些 Service 在不同 Cluster 里的不同实现类或子类。Flink目前(version1.11)实现的FlinkCluster 包括 3 大类:

- JobClusterEntrypoint
  - YarnJobClusterEntrypoint
  - MesosJobClusterEntrypoint
- SessionClusterEntrypoint
  - MesosSessionClusterEntrypoint
  - StandaloneSessionClusterEntrypoint
  - YarnSessionClusterEntrypoint
  - KubernetesSessionClusterEntrypoint
- ApplicationClusterEntryPoint
  - StandaloneApplicationClusterEntryPoint
  - YarnApplicationClusterEntryPoint
  - KubernetesApplicationClusterEntrypoint
- MiniCluster



- MiniCluster : JM和TM都运行在同一个JVM里，主要用于在 IDE (IntelliJ或Eclipse)调试 Flink Program (也叫做 application )。
- Standalone cluster : 不连接External Service (上图中灰色组件，如HA，Distributed storage, hardware Resoruce manager), JM和TM运行在不同的JVM里。 Flink release 中start-cluster.sh启动的就是StandaloneCluster.
- YarnCluster : Yarn管理的FlinkCluster, JM的ResourceManager连接Yarn的ResourceManager创建容器运行TaskManager。BlobServer, HAService 连接外部服务，使JM更可靠。
- MesosCluster : Mesos管理的FlinkCluster, JM的ResourceManager连接Mesos的ResourceManager创建容器运行TaskManager。BlobServer, HAService 连接外部服务，使JM更可靠。