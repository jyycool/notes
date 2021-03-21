## 【FLINK】slot

### 什么是slot

- flink-conf.yaml中默认taskmanager.numberOfTaskSlots=1;
- 以flink架构模型为例进行分析：

![flink架构](/Users/sherlock/Desktop/notes/allPics/Flink/flink架构.jpg)

- 图中 Task Manager 是从 Job Manager 处接收需要部署的 Task，任务的并行性由每个 Task Manager 上可用的 slot 决定。每个任务代表分配给任务槽的一组资源，slot 在 Flink 里面可以认为是资源组，Flink 将每个任务分成子任务并且将这些子任务分配到 slot 来并行执行程序。

- 例如，如果 Task Manager 有四个 slot，那么它将为每个 slot 分配 25％ 的内存。 可以在一个 slot 中运行一个或多个线程。 同一 slot 中的线程共享相同的 JVM。 同一 JVM 中的任务共享 TCP 连接和心跳消息。Task Manager 的一个 Slot 代表一个可用线程，该线程具有固定的内存，***<u>注意 Slot 只对内存隔离，没有对 CPU 隔离</u>***。默认情况下，***<u>Flink 允许子任务共享 Slot，即使它们是不同 task 的 subtask，只要它们来自相同的 job。这种共享可以有更好的资源利用率。</u>***

- 以官网上的thread process为例说明一下

  ![taskmanager](/Users/sherlock/Desktop/notes/allPics/Flink/taskmanager.jpg)

- 上面图片中有两个 Task Manager，每个 Task Manager 有三个 slot，这样我们的算子最大并行度那么就可以达到 6 个，在同一个 slot 里面可以执行 1 至多个子任务。

- 那么再看上面的图片，source/map/keyby/window/apply 最大可以有 6 个并行度，sink 只用了 1 个并行。

- 每个 Flink TaskManager 在集群中提供 slot。 slot 的数量通常与每个 TaskManager 的可用 CPU 内核数成比例。一般情况下你的 slot 数是你每个 TaskManager 的 cpu 的核数。

### parallelism与slot的区别

1. slot 是指 taskmanager 的并发执行能力；

   ![slot-tm](/Users/sherlock/Desktop/notes/allPics/Flink/slot-tm.jpg)

   如上图所示：taskmanager.numberOfTaskSlots:3；即每一个 taskmanager 中的分配 3 个 TaskSlot, 3 个 taskmanager 一共有 9 个 TaskSlot。

   

2. parallelism 是指 taskmanager 实际使用的并发能力

   ![p-tm](/Users/sherlock/Desktop/notes/allPics/Flink/p-tm.jpg)

   如上图所示：parallelism.default:1；即运行程序默认的并行度为 1，9 个 TaskSlot 只用了 1 个，有 8 个空闲。设置合适的并行度才能提高效率。

   

3. parallelism 是可配置、可指定的；

   ![parallelism](/Users/sherlock/Desktop/notes/allPics/Flink/parallelism.jpg)

   上图中 example2 每个算子设置的并行度是 2， example3 每个算子设置的并行度是 9。		![example-4](/Users/sherlock/Desktop/notes/allPics/Flink/example-4.jpg)

   上图中example4 除了 sink 是设置的并行度为 1，其他算子设置的并行度都是 9。

