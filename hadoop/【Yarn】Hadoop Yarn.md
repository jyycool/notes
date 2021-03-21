# 【Yarn】Hadoop Yarn

Apache Hadoop YARN 的基本思想是将 JobTracker 的两道主要职能: ***资源管理、作业调度/监控***, 拆分成两个独立的进程: ***一个全局的 ResourceManager (以下简称RM)*** 和 与***每个应用对应的 ApplicationMaster (以下简称AM)*** . RM 和每个节点上的 ***NodeManager (以下简称 NM)*** 组成了全新的通用操作系统以分布式的方式管理应用.

***下面是 Yarn 的架构图***

![Yarn](/Users/sherlock/Desktop/notes/allPics/Hadoop/Yarrn.png)

## 一、Yarn组件

通过添加新的功能, YARN为 Apache Hadoop 的工作流程带来了新的组件. 这些组件为用户提供了更加细粒度的控制, 并且同时为 Hadoop 生态系统提供了更加高级的功能.

### 1.1 ResourceManager

YRAN ResourceManager 是***一个纯粹的资源调度器***, 它根据应用程序的资源请求严格限制系统的可用资源. 在保证容量、公平性及服务等级(SAL)的前提下, 优化集群利用率,  即让所有资源都被充分利用. 为了适应不同的调度策略, ***RM有一个可以插拔的调度器(Sheduler)***, 来应用不同的调度算法, 例如, ***FairScheduler(公平调度算法), FIFO(先进先出)等***

### 1.2 ApplicationMaster

***AM 实际上是一个特定框架库的实例, 负责与 RM 协商申请资源, 并和 RM 协同工作来执行和监控 Container 以及它们的资源消耗***. 它有责任与 RM 协商获取资源合适的 Container, 跟踪他们的状态, 以及监控其进展.

AM 的设计使 YARN 可以提供以下新的重要特性:

- ***可扩展性*** : RM 作为一个单纯的资源调度器, 不必再提供跨集群的资源容错的功能. ***通过将资源容错功能转移到 AM 中, 控制就变得局部化***, 而不是全局的. 此外, 由于 A***M 是与应用程序一一对应的, 因此它不会成为集群的瓶颈***.
- ***开放性*** : 将所有与应用程序相关的代码都转移到 AM, 使得系统变的通用, 这样就可以支持如 MR, MPI 和图计算等多种框架
- 将所有复杂性尽可能的交给 AM, 同时提供足够多的功能给应用框架的开发者, 是只有足够的灵活性和能力
- 因为 AM 本质上还是客户端代码, 因此不能信任. 换言之, AM 不是一个特权服务
- YARN系统(RM 和 NM)必须保护自己免受错误的或者恶意的 AM 的影响, 并拥有所有资源的授权.

真实环境中, ***每一个应用都有自己的 AM 实例. 然而, 为一组应用提供一个 AM 实例是完全可行的***, 比如 Pig 或 Hive 的 AM. 另外, 这个概念已经延生到管理长时间运行的服务, 它们可以管理自己的应用, 例如, 通过一个特殊的 HBaseAppMaster 在 YRAN 中启动 HBase.

### 1.3 资源模型

一个应用(通过 AM) 可以请求非常具体的资源, 如下所示:

- 资源名称 (包括主机名称, 机架名称, 以及可能的复杂网络拓扑)
- 内存量
- CPU(核数/类型)
- 其他资源, 如 disk/network IO、GPU 等资源

### 1.4 RM 和 Container

YARN会感知集群的网络拓扑, 以便可以有效的调度及优化数据访问 (即尽可能的为应用减少数据移动)

为了达成这些目的, 位于 RM 内的中心调度器保存了应用程序的资源需求的信息, 以帮助它为集群中的所有应用做出更优的调度策略. 由此引出了 ResourceRequest 以及由此产生的 Container 概念.

本质上, 一个应用可以通过 AM 请求特定的资源需求来满足它的资源需要. Scheduler 会分配一个 Container 来响应资源需求, 用于满足由 AM 在 ResourceRequest 中提出的需求:

ResourceRequest具有以下形式:

***<资源名称, 优先级, 资源需求, Container 数>***

这些组成描述如下:

- 资源名称 : 资源期望所在的主机, 机架名称, 用 * 就是表示没有特殊需求. 未来可能支持更加复杂的拓扑, 比如一个主机上多个虚拟机, 更复杂的网络拓扑等.
- 优先级 : 应用程序内部请求的优先级 (不是多个应用程序之间的优先级). 优先级会调整应用程序内部各个 ResourceRequest 的次序
- 资源需求 : 需要的资源量, 如内存量, CPU 时间 (目前 YARN 仅支持内存和 CPU 两种资源维度)
- Container 数 :  表示需要这样的 Container 的数量, 它限制了用该 ResourceRequest 指定的 Container 总数

本质上, Container 是一种资源分配形式, 是 RM 为 ResourceRequest 成功分配的结果. Container 为应用程序授予在特定主机上使用资源 (如内存, CPU) 的权利.

AM 必须取走 Container, 并且交付给 NM, NM 会利用相应的资源来启动 Container 任务进程. 出于安全考虑, Container 的分配要以一种安全的方式进行验证, 来保证 AM 不能伪造集群中的应用.

### 1.5 Container 规范

Container 只是使用服务器(NM)上制定资源的权利, AM 必须向 NM 提供更多的信息来启动 Container. 与现有的 Hadoop MapReduce 不同, YARN 允许应用程序启动任何程序, 而不仅限于 Java 程序.

YRAN Container 的启动的 API 是与平台无关的, 包括以下元素:

- 启动 Container 内进程的命令行
- 环境变量
- 启动 Container 之前所需的本地资源, 如 JAR, 共享对象, 以及辅助的数据文件
- 安全相关的令牌

这种设计允许 AM 与 NM 协同工作来启动 Container 应用程序, 范围从简单的 Shell 脚本, 到 C/Java/Python 程序, 可能运行在 UNIX/Windows 上, 也可能运行在虚拟机上.



## 二、Yarn组件的功能概述

yarn 依赖于它的三个主要组件来实现所有功能.

1. ***ResourceManager(RM)*** : 集群资源的仲裁者, 它包含有两个主要部分: 一个***可插拔的调度器 (Scheduler)*** 和 一个 ***ApplicationManager*** 由于管理集群中的用户作业
2. ***NodeManager (NM)*** : 管理各个节点上的用户作业和工作流, 中央的 RM 和所有 NM 创建了集群统一的计算基础设施.
3. ***ApplicationMaster (AM)*** : 管理用户作业的生命周期, 是用户应用程序驻留的地方.

