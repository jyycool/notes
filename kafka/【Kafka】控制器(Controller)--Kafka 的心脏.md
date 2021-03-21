# 【Kafka】控制器(Controller)--Kafka 的心脏

在 Kafka 集群中会有一个或者多个 broker，其中有一个 broker 会被选举为控制器（Kafka Controller），***它负责管理整个集群中所有分区及其副本的状态***。当某个分区的 leader 副本出现故障时，由控制器负责为该分区选举新的 leader 副本。当检测到某个分区的 ISR 集合发生变化时，由控制器负责通知所有 broker 更新其元数据信息。当使用 kafka-topics.sh 脚本为某个topic 增加分区数量时，同样还是由控制器负责分区的重新分配。

Kafka 中的控制器选举的工作依赖于 Zookeeper，成功竞选为控制器的 broker 会在 Zookeeper 中创建 `/controller` 这个临时（EPHEMERAL）节点，此临时节点的内容参考如下：

```
{"version":1,"brokerid":0,"timestamp":"1529210278988"}
```

其中 version 在目前版本中固定为1，brokerid 表示称为控制器的 broker 的 id 编号，timestamp 表示竞选称为控制器时的时间戳。

***在任意时刻，集群中有且仅有一个控制器***。每个 broker 启动的时候会去尝试去读取 `/controller` 节点的 brokerid 的值，如果读取到 brokerid 的值不为 -1，则表示已经有其它 broker 节点成功竞选为控制器，所以当前 broker 就会放弃竞选；如果 Zookeeper 中不存在 `/controller` 这个节点，或者这个节点中的数据异常，那么就会尝试去创建 /controller 这个节点，当前 broker 去创建节点的时候，也有可能其他 broker 同时去尝试创建这个节点，只有创建成功的那个 broker 才会成为控制器，而创建失败的 broker 则表示竞选失败。每个 broker都会在内存中保存当前控制器的 brokerid 值，这个值可以标识为 activeControllerId。

Zookeeper 中还有一个与控制器有关的 `/controller_epoch` 节点，这个节点是持久（PERSISTENT）节点，节点中存放的是一个整型的 controller_epoch 值。controller_epoch 用于记录控制器发生变更的次数，即记录当前的控制器是第几代控制器，我们也可以称之为“控制器的纪元”。controller_epoch的初始值为1，即集群中第一个控制器的纪元为1，当控制器发生变更时，没选出一个新的控制器就将该字段值加 1。每个和控制器交互的请求都会携带上 controller_epoch 这个字段，如果请求的controller_epoch值小于内存中的controller_epoch值，则认为这个请求是向已经过期的控制器所发送的请求，那么这个请求会被认定为无效的请求。如果请求的 controller_epoch 值大于内存中的 controller_epoch 值，那么则说明已经有新的控制器当选了。由此可见，Kafka 通过 controller_epoch 来保证控制器的唯一性，进而保证相关操作的一致性。

具备控制器身份的 broker 需要比其他普通的 broker 多一份职责，具体细节如下：

1. 监听 partition 相关的变化。为 Zookeeper 中的 /admin/reassign_partitions 节点注册 PartitionReassignmentListener，用来处理分区重分配的动作。为 Zookeeper 中的 /isr_change_notification 节点注册 IsrChangeNotificetionListener，用来处理 ISR 集合变更的动作。为 Zookeeper 中的 /admin/preferred-replica-election 节点添加 PreferredReplicaElectionListener，用来处理优先副本的选举动作。
2. 监听 topic 相关的变化。为 Zookeeper 中的 /brokers/topics 节点添加TopicChangeListener，用来处理 topic 增减的变化；为 Zookeeper 中的 /admin/delete_topics 节点添加 TopicDeletionListener，用来处理删除 topic 的动作。
3. 监听 broker 相关的变化。为 Zookeeper 中的 /brokers/ids/ 节点添加BrokerChangeListener，用来处理 broker 增减的变化。
4. 从 Zookeeper 中读取获取当前所有与 topic、partition 以及 broker 有关的信息并进行相应的管理。对于所有 topic 所对应的 Zookeeper 中的 /brokers/topics/[topic]节点添加 PartitionModificationsListener，用来监听 topic 中的分区分配变化。
5. 启动并管理分区状态机和副本状态机。
6. 更新集群的元数据信息。
7. 如果参数 auto.leader.rebalance.enable 设置为 true，则还会开启一个名为“auto-leader-rebalance-task”的定时任务来负责维护分区的优先副本的均衡。

> 这个列表可能会让读者感到困惑，甚至完全不知所云。不要方~ 笔者这里只是用来突出控制器的职能很多，而这些功能的具体细节会在后面的文章中做具体的介绍。

控制器在选举成功之后会读取 Zookeeper 中各个节点的数据来初始化上下文信息（ControllerContext），并且也需要管理这些上下文信息，比如为某个 topic 增加了若干个分区，控制器在负责创建这些分区的同时也要更新上下文信息，并且也需要将这些变更信息同步到其他普通的 broker 节点中。不管是监听器触发的事件，还是定时任务触发的事件，亦或者是其他事件（比如 ControlledShutdown）都会读取或者更新控制器中的上下文信息，那么这样就会涉及到多线程间的同步，如果单纯的使用锁机制来实现，那么整体的性能也会大打折扣。针对这一现象，Kafka的控制器使用单线程基于事件队列的模型，将每个事件都做一层封装，然后按照事件发生的先后顺序暂存到LinkedBlockingQueue中，然后使用一个专用的线程（ControllerEventThread）按照FIFO（First Input First Output, 先入先出）的原则顺序处理各个事件，这样可以不需要锁机制就可以在多线程间维护线程安全。

[![img](http://image.honeypps.com/images/papers/2018/137.png)](http://image.honeypps.com/images/papers/2018/137.png)

在 Kafka 的早期版本中，并没有采用 Kafka Controller 这样一个概念来对分区和副本的状态进行管理，而是依赖于 Zookeeper，每个 broker 都会在 Zookeeper 上为分区和副本注册大量的监听器（Watcher）。当分区或者副本状态变化时，会唤醒很多不必要的监听器，这种严重依赖于Zookeeper 的设计会有脑裂、羊群效应以及造成 Zookeeper 过载的隐患。在目前的新版本的设计中，只有 Kafka Controller 在 Zookeeper 上注册相应的监听器，其他的 broker 极少需要再监听 Zookeeper 中的数据变化，这样省去了很多不必要的麻烦。不过每个 broker 还是会对 /controller 节点添加监听器的，以此来监听此节点的数据变化（参考ZkClient中的IZkDataListener）。

当 /controller 节点的数据发生变化时，每个 broker 都会更新自身内存中保存的activeControllerId。如果 broker 在数据变更前是控制器，那么如果在数据变更后自身的brokerid 值与新的 activeControllerId 值不一致的话，那么就需要“退位”，关闭相应的资源，比如关闭状态机、注销相应的监听器等。有可能控制器由于异常而下线，造成 /controller 这个临时节点会被自动删除；也有可能是其他原因将此节点删除了。

当 /controller 节点被删除时，每个 broker 都会进行选举，如果 broker 在节点被删除前是控制器的话，在选举前还需要有一个“退位”的动作。

***如果有特殊需要，可以手动删除 /controller 节点来触发新一轮的选举。当然关闭控制器所对应的broker 以及手动向 /controller 节点写入新的 brokerid 的所对应的数据同样可以触发新一轮的选举。***

