# 【Spark】Spark Job调优

当你开始编写 Apache Spark 代码时, 并在页面查询一些 API 时，你通常会遇到诸如*transform*，*action* 和 *RDD* 之类的词。在此级别上了解 Spark 对于编写 Spark 程序至关重要。同样，当程序运行出现问题时，或者当你尝试进入 Spark Web UI 尝试了解为什么应用程序需要这么长时间时，你将面对新的词汇，例如 *job*，*stage* 和 *task*。在这个级别上了解 Spark 对于编写*优秀的*Spark程序至关重要，当然，*好的，*我的意思是*很快*。要编写将高效执行的Spark程序，了解 Spark 的基础执行模型非常非常有帮助。

在本文中，你将学习如何在集群上实际执行 Spark 程序的基础知识。然后，您将获得一些有关Spark 的执行模型对编写高效程序的含义的实用建议。

## 一、代码调优

### 1.1 Spark如何执行程序

Spark 应用程序由单个 driver 进程和一组 executor 进程组成，这些进程分散在群集中的各个节点上。

driver 是负责完成用户使用高级 API 编写的一系列处理程序的过程。executor 进程负责以*任务*的形式执行此工作，并负责存储用户选择缓存的任何数据。驱动程序和执行程序通常都在应用程序运行的整个过程中始终存在，尽管[动态资源分配](http://spark.apache.org/docs/1.2.0/job-scheduling.html#dynamic-resource-allocation)改变了后者。单个 executor 具有多个 slot 用于运行任务，并且将在 executor 的整个生命周期中可以同时运行多个 slot。在群集上部署这些进程(driver & executors)取决于所使用的群集管理器（YARN，Mesos或Spark Standalone），但是 driver 程序和 executor 执行程序本身存在于每个Spark应用程序中。

![](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2019/08/spark-tuning-f1.png)

在执行层次结构的顶部是 *job*。在 Spark 应用程序内部调用 action 会触发 Spark job 的启动以完成该作业。为了确定 job 如何执行，Spark 会检查 RDD Graph 上的 transformation，并制定 execution plan。该 execution plan 从源端的 RDD（即不依赖其他 RDD 或引用已缓存数据的 RDD）开始，最终达到产生操作结果所需的最终 RDD。

执行计划包括将 job 的 transformations 组装到各个 *stage*。一个 *stage* 对应于一组*task*，这些 *task* 全部执行相同的代码，每个*task*都在数据的不同子集上。每个 *stage* 都包含一系列 transformations，这些 transformations 可以在不需要对数据进行 *shuffle*的情况下完成。

是什么决定是否需要 *shuffle* 数据？回想一下，RDD 包含固定数量的分区，每个分区都包含多个记录。对于由诸如 map 和 filter 之类的 *narrow* transformations 返回的RDD，在单个分区中计算记录所需的记录位于父 RDD 的单个分区中。子 RDD 的每个分区仅依赖于父 RDD 中的单个分区。诸如 coalesce 之类的操作可能导致任务处理多个输入分区，但是由于用于计算任何单个输出记录的输入记录仍只能驻留在分区的有限子集中，因此 coalesce 仍被认为是 *narrow* transformations的。

但是，Spark 还支持具有宽依赖的 transformation，例如 groupByKey 和 reduceByKey。在这些依赖性中，在单个分区中计算记录所需的数据可能驻留在父 RDD 的许多分区中。具有相同键的所有元组必须最终位于同一分区中，并由同一任务处理。为了满足这些操作，Spark 必须执行 *shuffle*，*shuffle* 将在集群内传输数据，并进入具有新分区集的新阶段。

> shuffle 一定发生在宽依赖中, 并且发生 shuffle 会触发 stage 的划分

例如，考虑以下代码：

```scala
sc.textFile("someFile.txt").
  map(mapFunc).
  flatMap(flatMapFunc).
  filter(filterFunc).
  count()
```

它执行一个 action，该 action 取决于从 textFile 派生的 RDD 上的一系列transformation。该代码将在单个阶段中执行，因为这三个操作的输出都不依赖于与其输入不同的分区的数据。

相反，下面代码查找每个字符在文本文件中出现超过1000次的所有单词中出现的次数。

```scala
val tokenized = sc.textFile(args(0)).flatMap(_.split(' '))
val wordCounts = tokenized.map((_, 1)).reduceByKey(_ + _)
val filtered = wordCounts.filter(_._2 >= 1000)
val charCounts = filtered.flatMap(_._1.toCharArray).map((_, 1)).
  reduceByKey(_ + _)
charCounts.collect()
```

这个过程将分为三个阶段。`reduceByKey` 操作会导致 stage 的划分，因为计算其输出需要通过键对数据进行重新分区。

这是一个更复杂的转换图，其中包括具有多个依赖性的 join 转换。

![](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2019/08/spark-tuning-f2.png)

粉色框显示用于执行该结果的阶段图。

![](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2019/08/spark-tuning-f3.png)

在每个 *stage* 中，父 stage 的 task(MapTask)将数据写入磁盘，然后在 子 stage 的 task 通过网络获取数据。由于它们会占用大量磁盘和网络 I/O，因此当产生 *stage* 时可能很消耗资源，应尽可能避免。父 stage 中数据分区的数量可能不同于子 stage 中数据分区的数量。会触发 *stage* 的 transformation 通常可以接受一个 `numPartitions` 参数，该参数确定在子*stage* 将数据划分成多少个分区。

正如 reducers 的数量是调整 MapReduce 作业的重要参数一样，调整 stage 上的分区数量通常会影响或破坏应用程序的性能。在后面的部分中，我们将更深入地研究如何调整 stage 上的分区数。

### 1.2 选择合适的 operators

尝试使用 Spark 完成某些任务时，开发人员通常可以从许多 action 和 transformation 中进行选择，以产生相同的结果。但是，并非所有这些可用的 operators 都会导致相同的性能：避免常见的陷阱并选择正确的 operators 可以使应用程序的性能与众不同。当有很多选择出现时，一些常见的优化规则和优化建议将帮助你选择合适的 operators 。

[SPARK-5097](https://issues.apache.org/jira/browse/SPARK-5097) 的最新工作是使 SchemaRDD 成为正式版中一个稳定组件，这将使得使用 Spark 核心 API 的程序员可以使用 Spark 的 Catalyst 优化器，从而使 Spark 可以对使用哪种 operators 做出更高层次的选择。当 SchemaRDD 成为稳定的组件时，Spark 将会替用户选择最合适的 operators。

在一系列 operators 中选择最合适的首要原则就是减少 shuffle 的次数和减少 shuffle 的数据量。这是因为 shuffle 是相当昂贵的操作；必须将所有随机播放数据写入磁盘，然后通过网络传输。 `repartition`，`join`，`cogroup` 和任何的 `*By, *ByKey` 操作都会导致 shuffle。但是，并非所有这些操作都是相等的，对于 Spark 新手开发人员来说，以下的错误操作会导致一些最常见的性能陷阱:

- **执行关联约简操作时，避免使用 groupByKey。**

  例如，`rdd.groupByKey().mapValues(_.sum)` 和 `rdd.reduceByKey(_ + _)` 将产生与相同的结果。但是，前者将在整个网络上传输整个数据集，而后者将为每个分区中的每个键计算局部和，并在改组后将这些局部和组合为较大的和。

- **当输入和输出值类型不同时，避免使用 reduceByKey。**

  例如，考虑编写一个转换，以查找与每个键对应的所有唯一字符串。一种方法是使用 map 将每个元素转换为 `Set`，然后将 `Set`与`reduceByKey` 组合：

  ```scala
  rdd.map(kv => (kv._1, new Set[String]() + kv._2))
      .reduceByKey(_ ++ _)
  ```

  此代码导致大量不必要的对象创建，因为必须为每个记录分配一个新的集合。最好使用 `aggregateByKey`，它可以更有效地执行 map-side 聚合：

  ```scala
  val zero = new collection.mutable.Set[String]()
  rdd.aggregateByKey(zero)(
      (set, v) => set += v,
      (set1, set2) => set1 ++= set2)
  ```

- **避免 flatMap-join-groupBy 模式。**

  如果两个数据集已经按键分组，并且你想将它们合并并保持分组，则可以使用 cogroup。这避免了与解压缩和重新打包组相关的所有开销。

### 1.3 当未发生 shuffle

注意上面示例中的 transformations *不会*导致 shuffle 的情况也很有用。当先前的 transformation 已根据同一 partitioner 对数据进行分区时，Spark知道可以避免 shuffle。考虑以下流程：

```scala
rdd1 = someRdd.reduceByKey(...)
rdd2 = someOtherRdd.reduceByKey(...)
rdd3 = rdd1.join(rdd2)
```

因为没有 partitioner 传递给 `reduceByKey`，所以将使用默认 partitioner，导致 rdd1 和 rdd2 都进行了 hash-partitioned。这两个 `reduceByKey` 将导致两次 shuffle。如果RDD 具有相同数量的分区，则 join 将不需要任何额外的 shuffle。由于 RDD 的分区相同，因此rdd1 的任何单个分区中的密钥集只能显示在 rdd2 的单个分区中。因此，rdd3 的任何单个输出分区的内容将仅取决于 rdd1 中的单个分区和 rdd2 中的单个分区的内容，并且不需要第三次 shuffle。

> 因为 rdd1, rdd2 都是用了 hash-partitioner, 所以 shuffle 后, reduce 端相同分区内的 key 一定是 相同的, 简单点说, rdd1, rdd2, 使用 hash-partitioner 后, 它们之间一定是窄依赖的关系, 所以它们 join 的时候不会发生 shuffle。

例如，如果 `someRdd` 有四个分区，`someOtherRdd` 有两个分区，并且两个都 `reduceByKey` 使用三个分区，则执行的任务集如下所示：

![](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2019/08/spark-tuning-f4.png)

如果 rdd1 和 rdd2 使用不同的 partitioner 或使用具有不同编号分区的 default (hash) partitioner 怎么办？在这种情况下，分区数量较少的 rdds 将需要重新组合以进行 join。

![](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2019/08/spark-tuning-f5.png)

合并两个数据集时避免 shuffle 的一种方法是利用[广播变量](http://spark.apache.org/docs/latest/programming-guide.html#broadcast-variables)。当其中一个数据集足够小以适合单个executor 的内存时，可以将其加载到驱动程序上的哈希表中，然后广播给每个 executor。然后，map transformation 可以引用哈希表进行查找。

### 1.4 什么时候 shuffle 更好

对于尽量减少 shuffle 的规则也是有偶尔例外的情况。额外的 shuffle 在提高并行度时可能对性能有利。例如，如果你的数据到达了几个无法拆分的大文件中， `InputFormat` 分区器会在每个分区中放置大量记录，从而无法会生成足够多的分区数来充分的利用所有可用的 CPU core。在这种情况下，在加载数据后调用 `repartition`（这将触发 shuffle）将允许其后的操作利用更多的 CPU。

使用 reduce 或 aggregate 将数据聚合到 driver 中时，可能会出现此异常的另一个实例。在大量分区上进行聚合时，计算可能很快成为单个线程上的 driver 中的瓶颈，将所有结果合并在一起。要减轻 driver 程序的负担，可以首先使用 `reduceByKey` 或 `aggregateByKey` 进行一轮分布式聚合，以将数据集划分为较少数量的分区。每个分区中的值会相互并行合并，然后再将其结果发送给 driver 以进行最后一轮聚合。看一看，[`treeReduce`](https://github.com/apache/spark/blob/f90ad5d426cb726079c490a9bb4b1100e2b4e602/mllib/src/main/scala/org/apache/spark/mllib/rdd/RDDFunctions.scala#L58) 和 [`treeAggregate`](https://github.com/apache/spark/blob/f90ad5d426cb726079c490a9bb4b1100e2b4e602/mllib/src/main/scala/org/apache/spark/mllib/rdd/RDDFunctions.scala#L90) 获取有关如何执行此操作的示例。（请注意，在撰写本文时，1.2是最新版本，它们被标记为开发人员API，但是[SPARK-5430](https://issues.apache.org/jira/browse/SPARK-5430) 试图在内核中添加它们的稳定版本。）

当 aggregation 已经按关键字分组时，此技巧特别有用。例如，考虑一个应用，该应用想要计算语料库中每个单词的出现次数，然后将结果作为 map 提取到 driver 中。可以通过 aggregation 操作完成的一种方法是在每个分区上计算本地 map，然后在驱动程序处合并这些 map。可以用另一种方法实现的是 `aggregateByKey`，以完全分布式的方式执行计数，然后 `collectAsMap` 将结果简单地提供给 driver。



## 二、执行参数调优

在本文中，我将尽力介绍几乎所有你想知道的有关使 Spark 程序快速运行的信息。特别是，你将学习资源调优，或配置 Spark 以利用群集必须提供的一切。然后，我们将转向调整并行度，这是工作绩效中最困难也是最重要的参数。最后，您将了解如何将数据本身表示为磁盘形式(Spark将读取磁盘形式(剧透警告:使用Apache Avro或Apache Parquet)，以及数据在缓存或在系统中移动时采用的内存格式。

### 2.1 调整资源分配

Spark用户列表有一连串的问题，类似于“我有一个500个节点的集群，但是当我运行我的应用程序时，我一次只看到两个任务在执行”。考虑到控制 Spark 资源利用率的参数数量，这些问题并非不公平，但在本节中，您将学习如何榨干你的集群的最后一点资源。这里的建议和配置在 Spark 的集群管理器(YARN、Mesos和Spark Standalone)之间有一点不同，但我们将只关注 Cloudera 向所有用户推荐的 YARN。

Spark（和YARN）考虑的两个主要资源是 *CPU* 和 *内存*。磁盘和网络 I/O 当然也影响 Spark 的性能，但是 Spark 和 YARN 目前都没有采取任何措施来主动管理它们。

应用程序中的每个 Spark executor 都有相同的固定核数和固定的堆大小。当从命令行调用spark-submit、spark-shell 和 pyspark 时，可以使用 `--executor-cores` 标志来指定内核数，也可以通过设置 `spark.executor.cores` 来指定。在 spark-defaults.conf 文件或SparkConf 对象上设置 core 属性。类似地，堆大小可以通过 `--executor-memory` 标志或`spark.executor.memory` 来控制。cores 属性可以控制执行程序运行的并发任务的数量。`--executor-cores 5` 表示每个 executor 最多可以同时运行 5 个任务。memory 属性会影响Spark 可以缓存的数据量，以及用于group、aggregate 和 join 的 shuffle 数据的最大大小。

`--num-executors` 命令行参数, `spark.executor.instances` 配置参数配置了当前 Spark 程序可以运行的最大 executor 数. 从 CDH 5.4/Spark 1.3 开始，你可以通过启用[动态分配](https://spark.apache.org/docs/latest/job-scheduling.html#dynamic-resource-allocation) spark.dynamicAllocation.enabled 来避免设置这个属性。动态分配允许 Spark 应用程序在有等待任务的积压时请求 executor，在 executor 空闲时释放 executor。

考虑一下 Spark 请求的资源将如何适应 YARN 的可用资源也很重要。YARN的相关属性是：

- `yarn.nodemanager.resource.memory-mb` 控制每个节点上的 container 使用的最大内存总和。
- `yarn.nodemanager.resource.cpu-vcores` 控制每个节点上的 container 使用的最大内核总数。

请求 5 个 executor cores 将导致向 YARN 请求 5 个虚拟核心。向 YARN 请求的内存有些复杂，原因如下: 

- `--executor-memory/spark.executor.memory` 控制 executor 堆的大小，但是 JVM 也可以在使用一些堆外内存，例如，interned String 和 direct byte buffers。该属性`spark.yarn.executor.memoryOverhead` 的值将添加到 executor 内存中，以确定每个执行程序对 YARN 的完整内存请求。默认为 max(384, 0.07 * spark.executor.memory)。
- YARN 可能会将请求的内存向上舍入一点。YARN `yarn.scheduler.minimum-allocation-mb` 和 `yarn.scheduler.increment-allocation-mb` 属性分别控制最小和增量请求值。

以下内容（不适用于默认值）显示了 Spark 和 YARN 中的内存属性层次结构：

![](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2019/08/spark-tuning2-f1.png)

如果这还不足以考虑，那么在确定 Spark executor 数的大小时需要考虑的最后几个问题：

- application master 是一个 non-executor container，具有从 YARN 请求containers 的特殊功能，它必须占用自己预算的资源。在 *yarn-client* 模式下，它默认为1024MB 和一个 vcore。在 *yarn-cluster* 模式下，application master 需要运行 driver 程序，因此使用 `--driver-memory` 和 `--driver-cores` 支持其资源通常很有用。
- 运行内存过多的 executor 通常会导致过多的垃圾回收延迟。对于单个 executor 来说，64GB是一个不错的上限。

- 我注意到 HDFS 客户端遇到大量并发线程的麻烦。粗略猜测是，每个执行者最多可以完成 5 个任务，以实现完整的写入吞吐量，因此最好将每个 executor 的内核数量保持在该数量以下。

- 运行 tiny executor(例如，只有一个 vcore 和 运行单个任务所需的内存)会丢掉在单个JVM 中运行多个任务带来的好处。例如，广播变量需要在每个执行器上复制一次，所以很多小的执行器会产生更多的数据拷贝。

为了使所有这些更加具体，下面是一个配置 Spark 应用程序以使用尽可能多的集群的工作示例：假想一个集群，其中有 6 个 NodeManager 节点，每个节点配备 16 core 和 64GB 内存。NodeManager 的容量, `yarn.nodemanager.resource.memory-mb` 和 `yarn.nodemanager.resource.cpu-vcores` 可能应分别设置为 63 * 1024 = 64512(MB) 和 15。我们应该避免将 100％ 的资源分配给 YARN container，因为该节点需要一些资源来运行OS 和 Hadoop 守护程序。在这种情况下，我们为这些系统进程保留了 1GB 和 1vcore。Cloudera Manager 通过考虑这些并自动配置这些 YARN 属性来提供帮助。

看到这里, 你可能一冲动然后使用如下配置:

```sh
 --num-executors 6 
 --executor-cores 15 
 --executor-memory 63G 
```

然而，这是错误的方法，因为:

- 63GB + executor.memoryOverhead 已经超过 NodeManager 的最大可以内存 63GB 了。
- application master 将占用一个节点上的一个 vcore，这意味着该节点上没有空间容纳一个15vcore 的 executor。
- 每个 executor 配置 15 个 vcore 会导致 HDFS I/O 吞吐量下降。

更好的选择是使用如下配置, 那么是如何得出这些配置参数的呢?

```sh
--num-executors 17 
--executor-cores 5 
--executor-memory 19G 
```

- 此配置将在所有节点(除了带有 AM 进程的那个 NodeManager 外, 因为它只有 2 个 executor)上产生 3 个 executor(3 * 5 + 2 * 1 = 17)

- `--executor-memory`的推导方法:

  ```sh
  每个 NodeManager 上的 container 的可用内存yarn.nodemanager.resource.memory-mb =  63GB
  
  每个 NodeManager 的 executor(container) 总数 = 3
  
  每个 NodeManager 的单个 container 可用内存 = 63 / 3 = 21GB
  
  container 内存 = executor.memoryOverhead + executor.memory
  executor.memoryOverhead = 0.07 * container = 21 * 0.07 = 1.47
  executor.memory = 21 - 1.47 ≈ 19GB
  ```

### 2.2 调整并行度

正如您可能已经指出的那样，Spark是一个并行处理引擎。可能不太明显的是，Spark不是“神奇的”并行处理引擎，并且其计算出最佳并行度的能力受到限制。Spark 每个 stage 都有许多任务，每个任务都按顺序处理数据。在调整 Spark 作业时，此数字可能是确定性能的唯一最重要的参数。

如何确定这个数字？在上一篇[文章中](https://clouderatemp.wpengine.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-1/)介绍了 Spark 将 RDD 分为多个阶段的方式。（作为快速提醒，像 repartition 和 reduceByKey 这样的 transformation 会产生 stage）stage 中的任务数与该 stage 中最后一个 RDD 中的分区数相同(shuffle, map 和 reduce task 的分区数相同)。RDD 中的分区数与它所依赖的 RDD 中的分区数相同，但有几个例外：`coalesce` 允许新创建的 RDD 的分区数少于其父 RDD 的分区数, `union` 创建的 RDD 的分区数为其父 RDD 分区的数量的总和, `cartesian` 则使用使用两个 join 的 RDD 中的一个的分区数来创建一个新的 RDD。

对于没有父 RDD 的 RDD 呢？由 textFile 或 hadoopFile 生成的 RDD 的分区由使用的底层MapReduce InputFormat 决定。通常，每个读取的 HDFS block 都有一个分区。RDD 的 分区数通常由用户给出的并行度参数来决定，如果没有给出，则使用 `spark.default.parallelism`。

要确定 RDD 中的分区数，您可以随时调用 `rdd.partitions().size()`

主要的问题是任务的数量太少。如果可用的任务比可运行它们的 slot 少，stage 阶段就不会充分利用所有可用的 CPU。

少量的任务还意味着对每个任务中发生的聚合操作会施加更多的内存压力。`任何join、cogroup 或 *ByKey 操作都涉及到将对象保存在 hashmap 或内存缓冲区中以进行分组或排序。`join、cogroup 和 groupByKey 在 shuffle 的 reduce 阶段(fetch 数据的阶段) 会从这些内存中拉取数据。reduceByKey 和 aggregateByKey 在 shuffle 的 map 阶段 和 reduce 阶段都会使用到这些内存中的数据。

当用于这些聚合操作的数据不容易装入内存时，可能会出现一些混乱。首先，在这些数据结构中保存许多记录会给垃圾收集带来压力，这可能会导致后续的暂停。第二，当记录不适合内存时，Spark 会将它们转移到磁盘，从而导致磁盘I/O和排序。这可能是我在Cloudera客户中看到的造成工作停顿的首要原因。

那么如何增加分区数呢？如果有问题的阶段正在从 Hadoop 读取，则您的选择是：

- 使用重新分区转换，这将触发 shuffle。
- 配置 InputFormat 来创建更多的 splits。
- 将输入数据以较小的 block size 写到 HDFS。

如果该 stage 的输入是从另一个 stage 中获取的，则触发 stage 的 transformation 将接受一个 `numPartitions` 参数，例如

```scala
val rdd2 = rdd1.reduceByKey(_ + _, numPartitions = X)
```

那么 “X”应该设置为多大呢？调整分区数量最直接的方法是实验：查看父 RDD 中的分区数量，然后将其乘以 1.5，直到性能不再提高。

还有一种更有原则的方法来计算 X，但它很难先验地应用，因为有些量很难计算。我把它包括在这里不是因为希望它被推荐为日常使用，而是因为它有助于理解发生了什么。主要目标是运行足够多的任务，以便分配给每个任务的数据能够装入该任务可用的内存中。

每个任务可用的内存为(spark.executor.memory * spark.shuffle.memoryFraction * spark.shuffle.safetyFraction) / spark.executor.cores。memoryFraction 和safetyFraction 的默认值分别为 0.2 和 0.8。

总 shuffle 数据的内存大小更难确定。最接近的启发式方法是查找已运行阶段的 Shuffle Spill (Memory) 和 Shuffle Spill (Disk) 的比率。然后将总的随机写入数乘以该数字。但是，如果该阶段正在执行减少操作，则可能会更加复杂：

![](https://ndu0e1pobsf1dobtvj5nls3q-wpengine.netdna-ssl.com/wp-content/uploads/2019/08/spark-tuning2-f2.png)

然后四舍五入，因为分区数多通常比分区数少好。

实际上，如果有疑问，最好总是在大量任务（以及分区）方面犯错。该建议与 MapReduce 的建议相反，后者要求您对任务数量更为保守。区别源于以下事实：MapReduce 具有较高的任务启动开销，而Spark 没有。

### 2.3 简化数据结构

数据以记录的形式流经 Spark。一条记录有两种表示形式：反序列化的Java对象表示形式和序列化的二进制表示形式。通常，Spark 将反序列化表示形式用于内存中的记录，将序列化表示形式用于存储在磁盘上或通过网络传输的记录。目前 Spark 已[计划](https://issues.apache.org/jira/browse/SPARK-2926)进行一些[工作](https://issues.apache.org/jira/browse/SPARK-4550) ，以序列化形式在内存中存储 shuffle 数据。

> Flink 就是使用 MemorySegment 在内存中保存数据的二进制形式, 这样不仅在反序列化的时候更有优势, 而且在排序, 合并的时候对内存的要求更精简

`spark.serializer` 属性控制用于在这两种表示形式之间进行转换的序列化器。首选序列化器就是 KryoSerializer: `org.apache.spark.serializer.KryoSerializer`。不幸的是，它不是默认值，因为在早期版本的 Spark 中 Kryo 中有些不稳定，并且希望不破坏兼容性，但是应*始终*使用 Kryo 序列化程序

记录在这两种表示中的占用空间对 Spark 性能有很大的影响。有必要回顾一下四处传播的数据类型，并寻找一些地方进行精简。

膨胀的反序列化对象会导致 Spark 更频繁地将数据溢出到磁盘，并减少 Spark 可以缓存的反序列化记录的数量(例如在内存存储级别)。火花调谐指南有一个伟大的部分瘦身这些。

膨胀的序列化对象将导致更大的磁盘和网络I/O，同时减少了 Spark 可以缓存的序列化记录的数量(例如在MEMORY_SER存储级别)。这里的主要操作项是确保注册您定义的任何自定义类，并使用SparkConf#registerkyoclasses API传递。

### 2.4 Data Formats

当您有权决定如何将数据存储在磁盘上时，请使用可扩展的二进制格式，如 Avro、Parquet、Thrift 或 Protobuf。选择其中一种格式并坚持使用。需要明确的是，当谈到在 Hadoop 上使用Avro、Thrift 或 Protobuf 时，它们意味着每个记录都是一个存储在序列文件中的Avro/Thrift/Protobuf结构体。JSON 是不值得的。

每次你考虑使用 JSON 存储大量的数据, 请联想到在即将在中东发生的冲突, 堵塞在加拿大的美丽的河流, 或来自核电站的放射性尘埃将建在美国中心地带权力所花费的CPU周期解析一遍又一遍你的文件。此外，试着学习人际交往技巧，这样你就能说服你的同事和上级也这么做。





本文翻译自: https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-2/

