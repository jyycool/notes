# 【Spark】Spark shuffle vs MR shuffle

## 一、Shuffle 简介

Shuffle 的本意是洗牌、混洗的意思，把一组有规则的数据尽量打乱成无规则的数据。而在 MR(MapReduce) 中，Shuffle 更像是洗牌的逆过程，指的是将 map 端的无规则输出按指定的规则“打乱”成具有一定规则的数据，以便 reduce 端接收处理。其在 MR 中所处的工作阶段是 map 输出后到 reduce 接收前，具体可以分为 map 端和 reduce 端前后两个部分。

在 shuffle 之前，也就是在 map 阶段，MR 会对要处理的数据进行分片（split）操作，为每一个分片分配一个 MapTask 任务。接下来 map 会对每一个分片中的每一行数据进行处理得到键值对（key,value）此时得到的键值对又叫做“中间结果”。此后便进入 reduce 阶段，由此可以看出Shuffle 阶段的作用是处理“中间结果”。

由于 Shuffle 涉及到了磁盘的读写和网络的传输，因此 Shuffle 性能的高低直接影响到了整个程序的运行效率。

## 二、MR Shuffle

对于 MapReduce 作业，完整的作业运行流程，这里借用刘军老师的[Hadoop大数据处理](http://item.jd.com/11315351.html)中的一张图：

[![hadoop](https://matt33.com/images/hadoop/hadoop.png)](https://matt33.com/images/hadoop/hadoop.png)

完整过程应该是分为7部分，分别是：

1. 作业启动：开发者通过控制台启动作业；
2. 作业初始化：这里主要是切分数据、创建作业和提交作业，与第三步紧密相联；
3. 作业/任务调度：对于 1.0 版的 Hadoop 来说就是 JobTracker 来负责任务调度，对于 2.0 版的 Hadoop 来说就是 Yarn 中的 Resource Manager 负责整个系统的资源管理与分配，Yarn 可以参考IBM的一篇博客 [Hadoop新MapReduce框架Yarn详解](https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-yarn/)；
4. Map任务；
5. Shuffle；
6. Reduce任务；
7. 作业完成：通知开发者任务完成。

而这其中最主要的 MapReduce 过程，主要是第4、5、6步三部分，详细作用如下：

- **Map**:数据输入,做初步的处理,输出形式的中间结果；
- **Shuffle**:按照 partition、key 对中间结果进行排序合并,输出给 reduce 线程；
- **Reduce**:对相同 key 的输入进行最终的处理,并将结果写入到文件中。

![](https://matt33.com/images/hadoop/mapreduce.png)

上图是把 MR 过程分为两个部分，而实际上从两边的 Map 和 Reduce 到中间的那一大块都属于Shuffle 过程，也就是说，Shuffle 过程有一部分是在 Map 端，有一部分是在 Reduce 端，下文也将会分两部分来介绍 Shuffle 过程。

对于 Hadoop 集群，当我们在运行作业时，大部分的情况下，map task 与 reduce task 的执行是分布在不同的节点上的，因此，很多情况下，reduce 执行时需要跨节点去拉取其他节点上的 map task结果，这样造成了集群内部的网络资源消耗很严重，而且在节点的内部，相比于内存，磁盘 IO 对性能的影响是非常严重的。如果集群中运行的作业有很多，那么 task 的执行对于集群内部网络的资源消费非常大。因此，我们对于 MapRedue 作业 Shuffle 过程的期望是：

- 完整地从 map task 端拉取数据到 reduce 端；
- 在跨节点拉取数据时，尽可能地减少对带宽的不必要消耗；
- 减少磁盘 IO 对 task 执行的影响。

### 2.1 Map shuffle

在进行海量数据处理时，外存文件数据**I/O访问**会成为一个制约系统性能的瓶颈，因此，Hadoop的Map过程实现的一个重要原则就是：**计算靠近数据**，这里主要指两个方面：

1. 代码靠近数据：
   - 原则：本地化数据处理（locality），即一个计算节点尽可能处理本地磁盘上所存储的数据；
   - 尽量选择数据所在 DataNode 启动 Map 任务；
   - 这样可以减少数据通信，提高计算效率；
2. 数据靠近代码：
   - 当本地没有数据处理时，尽可能从同一机架或最近其他节点传输数据进行处理（host选择算法）。

下面，我们分块去介绍 Hadoop 的 Map 过程，map 的经典流程图如下：

![](https://matt33.com/images/hadoop/map-shuffle.png)

#### 2.1.1 输入

1. map task 只读取 split 分片，split 与 block（hdfs的最小存储单位，默认为64MB）可能是一对一也能是一对多，但是对于一个 split 只会对应一个文件的一个 block 或多个 block，不允许一个 split 对应多个文件的多个 block(即 split 不能跨文件)；
2. 这里切分和输入数据的时会涉及到 InputFormat 的文件切分算法和 host 选择算法。

文件切分算法，主要用于确定InputSplit的个数以及每个InputSplit对应的数据段。FileInputFormat以文件为单位切分生成InputSplit，对于每个文件，由以下三个属性值决定其对应的InputSplit的个数：

- goalSize： 它是根据用户期望的InputSplit数目计算出来的，即totalSize/numSplits。其中，totalSize为文件的总大小；numSplits为用户设定的Map Task个数，默认情况下是1；
- minSize：InputSplit的最小值，由配置参数`mapred.min.split.size`确定，默认是1；
- blockSize：文件在hdfs中存储的block大小，不同文件可能不同，默认是64MB。

这三个参数共同决定InputSplit的最终大小，计算方法如下：

```
splitSize = max{minSize, min{gogalSize,blockSize}}
```

FileInputFormat 的 host选择算法参考《Hadoop技术内幕-深入解析MapReduce架构设计与实现原理》的p50.

#### 2.1.2 Partition

- 作用：将map的结果发送到相应的reduce端，总的partition的数目等于reducer的数量。
- 实现功能：
  1. map输出的是key/value对，决定于当前的mapper的part交给哪个reduce的方法是：mapreduce提供的Partitioner接口，对key进行hash后，再以reducetask数量取模，然后到指定的job上（**HashPartitioner**，可以通过`job.setPartitionerClass(MyPartition.class)`自定义）。
  2. 然后将数据写入到内存缓冲区，缓冲区的作用是批量收集map结果，减少磁盘IO的影响。key/value对以及Partition的结果都会被写入缓冲区。在写入之前，key与value值都会被序列化成字节数组。
- 要求：负载均衡，效率；

#### 2.1.3 spill（溢写）：sort & combiner

- 作用：把内存缓冲区中的数据写入到本地磁盘，在写入本地磁盘时先按照partition、再按照key进行排序（`quick sort`）；
- 注意：
  1. 这个spill是由**另外单独的线程**来完成，不影响往缓冲区写map结果的线程；
  2. 内存缓冲区默认大小限制为100MB，它有个溢写比例（`spill.percent`），默认为0.8，当缓冲区的数据达到阈值时，溢写线程就会启动，先锁定这80MB的内存，执行溢写过程，maptask的输出结果还可以往剩下的20MB内存中写，互不影响。然后再重新利用这块缓冲区，因此Map的内存缓冲区又叫做**环形缓冲区**（两个指针的方向不会变，下面会详述）；
  3. 在将数据写入磁盘之前，先要对要写入磁盘的数据进行一次**排序**操作，先按`<key,value,partition>`中的partition分区号排序，然后再按key排序，这个就是**sort操作**，最后溢出的小文件是分区的，且同一个分区内是保证key有序的；

**combine**：执行combine操作要求开发者必须在程序中设置了combine（程序中通过`job.setCombinerClass(myCombine.class)`自定义combine操作）。

- 程序中有两个阶段可能会执行combine操作：
  1. map输出数据根据分区排序完成后，在写入文件之前会执行一次combine操作（前提是作业中设置了这个操作）；
  2. 如果map输出比较大，溢出文件个数大于3（此值可以通过属性`min.num.spills.for.combine`配置）时，在merge的过程（多个spill文件合并为一个大文件）中还会执行combine操作；
- combine主要是把形如`<aa,1>,<aa,2>`这样的key值相同的数据进行计算，计算规则与reduce一致，比如：当前计算是求key对应的值求和，则combine操作后得到`<aa,3>`这样的结果。
- 注意事项：不是每种作业都可以做combine操作的，只有满足以下条件才可以：
  1. reduce的输入输出类型都一样，因为combine本质上就是reduce操作；
  2. 计算逻辑上，combine操作后不会影响计算结果，像求和就不会影响；

#### 2.1.4 merge

- merge过程：当map很大时，每次溢写会产生一个spill_file，这样会有多个spill_file，而最终的一个map task输出只有一个文件，因此，最终的结果输出之前会对多个中间过程进行多次溢写文件（spill_file）的合并，此过程就是merge过程。也即是，待Map Task任务的所有数据都处理完后，会对任务产生的所有中间数据文件做一次合并操作，以确保一个Map Task最终只生成一个中间数据文件。
- 注意：
  1. 如果生成的文件太多，可能会执行多次合并，每次最多能合并的文件数默认为10，可以通过属性`min.num.spills.for.combine`配置；
  2. 多个溢出文件合并时，会进行一次排序，排序算法是**多路归并排序**；
  3. 是否还需要做combine操作，一是看是否设置了combine，二是看溢出的文件数是否大于等于3；
  4. 最终生成的文件格式与单个溢出文件一致，也是按分区顺序存储，并且输出文件会有一个对应的索引文件，记录每个分区数据的起始位置，长度以及压缩长度，这个索引文件名叫做`file.out.index`。

#### 2.1.5 内存缓冲区

1. 在 Map Task 任务的业务处理方法 map()中，最后一步通过`OutputCollector.collect(key,value)`或`context.write(key,value)`输出Map Task的中间处理结果，在相关的`collect(key,value)`方法中，会调用`Partitioner.getPartition(K2 key, V2 value, int numPartitions)`方法获得输出的key/value对应的分区号(分区号可以认为对应着一个要执行Reduce Task的节点)，然后将`<key,value,partition>`暂时保存在内存中的MapOutputBuffe内部的环形数据缓冲区，该缓冲区的默认大小是100MB，可以通过参数`io.sort.mb`来调整其大小。
2. 当缓冲区中的数据使用率达到一定阀值后，触发一次Spill操作，将环形缓冲区中的部分数据写到磁盘上，生成一个临时的Linux本地数据的spill文件；然后在缓冲区的使用率再次达到阀值后，再次生成一个spill文件。直到数据处理完毕，在磁盘上会生成很多的临时文件。
3. 缓存有一个阀值比例配置，当达到整个缓存的这个比例时，会触发spill操作；触发时，map输出还会接着往剩下的空间写入，但是写满的空间会被锁定，数据溢出写入磁盘。当这部分溢出的数据写完后，空出的内存空间可以接着被使用，形成像环一样的被循环使用的效果，所以又叫做**环形内存缓冲区**；
4. MapOutputBuffe内部存数的数据采用了两个索引结构，涉及三个环形内存缓冲区。下来看一下两级索引结构：

[![buffer](https://matt33.com/images/hadoop/buffer.jpg)](https://matt33.com/images/hadoop/buffer.jpg)buffer

**写入到缓冲区的数据采取了压缩算法 http://www.cnblogs.com/edisonchou/p/4298423.html**
这三个环形缓冲区的含义分别如下：

1. **kvoffsets**缓冲区：也叫偏移量索引数组，用于保存`key/value`信息在位置索引 `kvindices` 中的偏移量。当 `kvoffsets` 的使用率超过 `io.sort.spill.percent` (默认为80%)后，便会触发一次 SpillThread 线程的“溢写”操作，也就是开始一次 Spill 阶段的操作。
2. **kvindices**缓冲区：也叫位置索引数组，用于保存 `key/value` 在数据缓冲区 `kvbuffer` 中的起始位置。
3. **kvbuffer**即数据缓冲区：用于保存实际的 `key/value` 的值。默认情况下该缓冲区最多可以使用 `io.sort.mb` 的95%，当 `kvbuffer` 使用率超过 `io.sort.spill.percent` (默认为80%)后，便会出发一次 SpillThread 线程的“溢写”操作，也就是开始一次 Spill 阶段的操作。

写入到本地磁盘时，对数据进行排序，实际上是对**kvoffsets**这个偏移量索引数组进行排序。

### 2.2 Reduce

Reduce过程的经典流程图如下：

[![reduce-shuffle](https://matt33.com/images/hadoop/reduce-shuffle.png)](https://matt33.com/images/hadoop/reduce-shuffle.png)reduce-shuffle

#### 2.2.1 copy过程

- 作用：拉取数据；
- 过程：Reduce进程启动一些数据copy线程(`Fetcher`)，通过HTTP方式请求map task所在的TaskTracker获取map task的输出文件。因为这时map task早已结束，这些文件就归TaskTracker管理在本地磁盘中。
- 默认情况下，当整个MapReduce作业的所有已执行完成的Map Task任务数超过Map Task总数的5%后，JobTracker便会开始调度执行Reduce Task任务。然后Reduce Task任务默认启动`mapred.reduce.parallel.copies`(默认为5）个MapOutputCopier线程到已完成的Map Task任务节点上分别copy一份属于自己的数据。 这些copy的数据会首先保存的内存缓冲区中，当内冲缓冲区的使用率达到一定阀值后，则写到磁盘上。

**内存缓冲区**

- 这个内存缓冲区大小的控制就不像map那样可以通过`io.sort.mb`来设定了，而是通过另外一个参数来设置：`mapred.job.shuffle.input.buffer.percent（default 0.7）`， 这个参数其实是一个百分比，意思是说，shuffile在reduce内存中的数据最多使用内存量为：0.7 × `maxHeap of reduce task`。
- 如果该reduce task的最大heap使用量（通常通过`mapred.child.java.opts`来设置，比如设置为-Xmx1024m）的一定比例用来缓存数据。默认情况下，reduce会使用其heapsize的70%来在内存中缓存数据。如果reduce的heap由于业务原因调整的比较大，相应的缓存大小也会变大，这也是为什么reduce用来做缓存的参数是一个百分比，而不是一个固定的值了。

#### 2.2.2 merge过程

- Copy过来的数据会先放入内存缓冲区中，这里的缓冲区大小要比 map 端的更为灵活，它基于 JVM 的`heap size`设置，因为 Shuffle 阶段 Reducer 不运行，所以应该把绝大部分的内存都给 Shuffle 用。
- 这里需要强调的是，merge 有三种形式：1)内存到内存 2)内存到磁盘 3)磁盘到磁盘。默认情况下第一种形式是不启用的。当内存中的数据量到达一定阈值，就启动内存到磁盘的 merge（图中的第一个merge，之所以进行merge是因为reduce端在从多个map端copy数据的时候，并没有进行sort，只是把它们加载到内存，当达到阈值写入磁盘时，需要进行merge） 。这和map端的很类似，这实际上就是溢写的过程，在这个过程中如果你设置有Combiner，它也是会启用的，然后在磁盘中生成了众多的溢写文件，这种merge方式一直在运行，直到没有 map 端的数据时才结束，然后才会启动第三种磁盘到磁盘的 merge （图中的第二个merge）方式生成最终的那个文件。
- 在远程copy数据的同时，Reduce Task在后台启动了两个后台线程对内存和磁盘上的数据文件做合并操作，以防止内存使用过多或磁盘生的文件过多。

#### 2.2.3 reducer的输入文件

- merge的最后会生成一个文件，大多数情况下存在于磁盘中，但是需要将其放入内存中。当reducer 输入文件已定，整个 Shuffle 阶段才算结束。然后就是 Reducer 执行，把结果放到 HDFS 上。



## 三、Spark Shuffle

Spark 的 Shuffle 是在 MR Shuffle 基础上进行的调优。其实就是对排序、合并逻辑做了一些优化。在 Spark 中 Shuffle write 相当于 MR 的map，Shuffle read 相当于 MR 的reduce。

Spark 丰富了任务类型，有些任务之间数据流转不需要通过 Shuffle，但是有些任务之间还是需要通过 Shuffle 来传递数据，比如宽依赖的 group by key 以及各种 by key 算子。宽依赖之间会划分 stage，而 Stage 之间就是 Shuffle，如下图中的 stage1 和 stage3 之间就会产生 Shuffle。

![](https://mmbiz.qpic.cn/mmbiz_jpg/MtezESMLd6GrBFdYOGYuI5sw0CoRXjvWu1icoNNtj0CQ3Ria97EYBib9uic6NCAJJ10FXUwyCCv0RRO6RJPLNHLpdg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在Spark的中，负责shuffle过程的执行、计算和处理的组件主要就是ShuffleManager，也即shuffle管理器。ShuffleManager随着Spark的发展有两种实现的方式，分别为HashShuffleManager和SortShuffleManager，因此spark的Shuffle有Hash Shuffle和Sort Shuffle两种。

### 3.1 Spark Shuffle 发展史

- Spark 0.8及以前 Hash Based Shuffle
- Spark 0.8.1 为 Hash Based Shuffle 引入 File Consolidation 机制
- Spark 0.9 引入 ExternalAppendOnlyMap
- Spark 1.1 引入 Sort Based Shuffle，但默认仍为 Hash Based Shuffle
- Spark 1.2 默认的 Shuffle 方式改为 Sort Based Shuffle
- Spark 1.4 引入 Tungsten-Sort Based Shuffle
- Spark 1.6 Tungsten-sort 并入 Sort Based Shuffle
- Spark 2.0 Hash Based Shuffle 退出历史舞台

在 Spark 的版本的发展，ShuffleManager 在不断迭代，变得越来越先进。

在 Spark 1.2 以前，默认的 shuffle 计算引擎是 HashShuffleManager。该ShuffleManager 而 HashShuffleManager 有着一个非常严重的弊端，就是会产生大量的中间磁盘文件，进而由大量的磁盘 IO 操作影响了性能。因此在 Spark 1.2 以后的版本中，默认的ShuffleManager 改成了 SortShuffleManager。

SortShuffleManager 相较于 HashShuffleManager 来说，有了一定的改进。主要就在于，每个 Task 在进行 shuffle 操作时，虽然也会产生较多的临时磁盘文件，但是最后会将所有的临时文件合并(merge)成一个磁盘文件，因此每个 Task 就只有一个磁盘文件。在下一个 stage 的shuffle read task 拉取自己的数据时，只要根据索引读取每个磁盘文件中的部分数据即可。

### 3.2 Hash Shuffle

HashShuffleManager 的运行机制主要分成两种，一种是普通运行机制，另一种是合并的运行机制。合并机制主要是通过复用 buffer 来优化 Shuffle 过程中产生的小文件的数量。Hash shuffle是不具有排序的 Shuffle。

#### 3.2.1 普通机制的 Hash Shuffle

最开始使用的 Hash Based Shuffle，每个 Mapper 会根据 Reducer 的数量创建对应的bucket，bucket 的数量是 M * R，M 是 map 的数量，R 是 Reduce的数量。如下图所示：2个core, 4 个 map task, 3 个 reduce task，会产生 4*3=12 个小文件。
![图片](https://mmbiz.qpic.cn/mmbiz_jpg/MtezESMLd6GrBFdYOGYuI5sw0CoRXjvW647604PeelVFic1ONCNrBzTYaNbXVzCefYfiaPCRCdUkicHlMr6DJVKww/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 3.2.2 优化后的 Hash Shuffle

普通机制 Hash Shuffle 会产生大量的小文件(M * R），对文件系统的压力也很大，也不利于IO的吞吐量，后来做了优化（设置 spark.shuffle.consolidateFiles=true 开启，默认false），把在同一个 core 上的多个 Mapper 输出到同一个文件，这样文件数就变成 core * R 个了。如下图所示：2个core 4个map task 3 个reduce task，会产生2*3=6个小文件。
![图片](https://mmbiz.qpic.cn/mmbiz_jpg/MtezESMLd6GrBFdYOGYuI5sw0CoRXjvWXLicEqqdOml6ibTgQQg0mIrk6EwKCFfueJbhVgBoLKyAXvYKEic9AtVUw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Hash shuffle 合并机制的问题：如果 Reducer 端的并行任务或者是数据分片过多的话则 Core * Reducer Task 依旧过大，也会产生很多小文件。进而引出了更优化的 sort shuffle。在Spark 1.2 以后的版本中，默认的 ShuffleManager 改成了 SortShuffleManager。

### 3.3 Sort Shuffle

SortShuffleManager 的运行机制主要分成两种，一种是普通运行机制，另一种是 bypass 运行机制。当 shuffle read task 的数量小于等于 spark.shuffle.sort.bypassMergeThreshold 参数的值时(默认为200)，就会启用 bypass 机制。

#### 3.3.1 普通机制的 Sort Shuffle

这种机制和 mapreduce 差不多，在该模式下，数据会先写入一个内存数据结构中，此时根据不同的shuffle 算子，可能选用不同的数据结构。如果是 reduceByKey 这种聚合类的 shuffle 算子，那么会选用 Map 数据结构，一边通过 Map 进行聚合，一边写入内存；如果是 join 这种普通的shuffle 算子，那么会选用 Array 数据结构，直接写入内存。接着，每写一条数据进入内存数据结构之后，就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘，然后清空内存数据结构。
![图片](https://mmbiz.qpic.cn/mmbiz_jpg/MtezESMLd6GrBFdYOGYuI5sw0CoRXjvWTAbIf3a085k7xflENqGqW4gH7amBpys3E3kQGproibrBIyfBX2YGQLw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在溢写到磁盘文件之前，会先根据key对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。默认的batch数量是10000条，也就是说，排序好的数据，会以每批1万条数据的形式分批写入磁盘文件。

一个task将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也会产生多个临时文件。最后会将之前所有的临时磁盘文件都进行合并，由于一个task就只对应一个磁盘文件因此还会单独写一份索引文件，其中标识了下游各个task的数据在文件中的start offset与end offset。

SortShuffleManager由于有一个磁盘文件merge的过程，因此大大减少了文件数量，由于每个task最终只有一个磁盘文件所以文件个数等于上游shuffle write个数。

#### 3.3.2 bypass 机制的 Sort Shuffle

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/MtezESMLd6GrBFdYOGYuI5sw0CoRXjvWBDvAq7TibwcGuvwUGYKTeibYT1VnVKlrvvlX9s4ZK6Viav0wV8nPlkO6A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

bypass运行机制的触发条件如下：

- shuffle map task数量小于spark.shuffle.sort.bypassMergeThreshold参数的值，默认值200。
- 不是聚合类的shuffle算子(比如reduceByKey)。

此时task会为每个reduce端的task都创建一个临时磁盘文件，并将数据按key进行hash然后根据key的hash值，将key写入对应的磁盘文件之中。当然，写入磁盘文件时也是先写入内存缓冲，缓冲写满之后再溢写到磁盘文件的。最后，同样会将所有临时磁盘文件都合并成一个磁盘文件，并创建一个单独的索引文件。

该过程的磁盘写机制其实跟未经优化的HashShuffleManager是一模一样的，因为都要创建数量惊人的磁盘文件，只是在最后会做一个磁盘文件的合并而已。因此少量的最终磁盘文件，也让该机制相对未经优化的HashShuffleManager来说，shuffle read的性能会更好。

而该机制与普通SortShuffleManager运行机制的不同在于：

第一，磁盘写机制不同;

第二，不会进行排序。也就是说，启用该机制的最大好处在于，shuffle write过程中，不需要进行数据的排序操作，也就节省掉了这部分的性能开销。

### 3.4 Spark Shuffle 总结

Shuffle 过程本质上都是将 Map 端获得的数据使用分区器进行划分，并将数据发送给对应的 Reducer 的过程。

Shuffle作为处理连接map端和reduce端的枢纽，其shuffle的性能高低直接影响了整个程序的性能和吞吐量。map端的shuffle一般为shuffle的Write阶段，reduce端的shuffle一般为shuffle的read阶段。Hadoop和spark的shuffle在实现上面存在很大的不同，spark的shuffle分为两种实现，分别为HashShuffle和SortShuffle。

HashShuffle又分为普通机制和合并机制，普通机制因为其会产生MR个数的巨量磁盘小文件而产生大量性能低下的Io操作，从而性能较低，因为其巨量的磁盘小文件还可能导致OOM，HashShuffle的合并机制通过重复利用buffer从而将磁盘小文件的数量降低到CoreR个，但是当Reducer 端的并行任务或者是数据分片过多的时候，依然会产生大量的磁盘小文件。

SortShuffle也分为普通机制和bypass机制，普通机制在内存数据结构(默认为5M)完成排序，会产生2M个磁盘小文件。而当shuffle map task数量小于spark.shuffle.sort.bypassMergeThreshold参数的值。或者算子不是聚合类的shuffle算子(比如reduceByKey)的时候会触发SortShuffle的bypass机制，SortShuffle的bypass机制不会进行排序，极大的提高了其性能。

在Spark 1.2以前，默认的shuffle计算引擎是HashShuffleManager，因为HashShuffleManager会产生大量的磁盘小文件而性能低下，在Spark 1.2以后的版本中，默认的ShuffleManager改成了SortShuffleManager。

SortShuffleManager相较于HashShuffleManager来说，有了一定的改进。主要就在于，每个Task在进行shuffle操作时，虽然也会产生较多的临时磁盘文件，但是最后会将所有的临时文件合并(merge)成一个磁盘文件，因此每个Task就只有一个磁盘文件。在下一个stage的shuffle read task拉取自己的数据时，只要根据索引读取每个磁盘文件中的部分数据即可。

## 四、Spark & MR Shuffle 的异同

- 从整体功能上看，两者并没有大的差别。都是将 mapper（Spark 里是 ShuffleMapTask）的输出进行 partition，不同的 partition 送到不同的 reducer（Spark 里 reducer 可能是下一个 stage 里的 ShuffleMapTask，也可能是 ResultTask）。Reducer 以内存作缓冲区，边 shuffle 边 aggregate 数据，等到数据 aggregate 好以后进行 reduce（Spark 里可能是后续的一系列操作）。
- 从流程的上看，两者差别不小。Hadoop MapReduce 是 sort-based，进入 combine和 reduce的 records 必须先 sort。这样的好处在于 combine/reduce可以处理大规模的数据，因为其输入数据可以通过外排得到（mapper 对每段数据先做排序，reducer 的 shuffle 对排好序的每段数据做归并）。以前 Spark 默认选择的是 hash-based，通常使用 HashMap 来对 shuffle 来的数据进行合并，不会对数据进行提前排序。如果用户需要经过排序的数据，那么需要自己调用类似 sortByKey的操作。在Spark 1.2之后，sort-based变为默认的Shuffle实现。
- 从流程实现角度来看，两者也有不少差别。Hadoop MapReduce 将处理流程划分出明显的几个阶段：map, spill, merge, shuffle, sort, reduce等。每个阶段各司其职，可以按照过程式的编程思想来逐一实现每个阶段的功能。在 Spark 中，没有这样功能明确的阶段，只有不同的 stage 和一系列的 transformation，所以 spill, merge, aggregate 等操作需要蕴含在 transformation中。