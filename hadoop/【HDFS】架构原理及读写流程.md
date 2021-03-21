# 【HDFS】架构原理及读写流程

## 一、HDFS 架构体系

HDFS 采用的是 master/slaves 这种主从的结构模型管理数据，这种结构模型主要由四个部分组成，分别是

- Client(客户端)
- Namenode(名称节点)
- Datanode(数据节点)
- SecondaryNameNode

### 1.1 组成HDFS的各模块作用

#### 1.2.1 Client

HDFS 客户端是在 DFSClient 类的基础上实现的，提供了命令行接口、API接口、浏览器接口等面向用户的接口，使用户可以不考虑 HDFS 的实现细节，简化操作。

客户端在整个 HDFS 的作用可以进行如下总结：

- 上传文件时按照Block块大小进行文件的切分；
- 和NameNode交互，获取文件位置信息；
- 和DataNode交互，读取和写入数据；
- 管理和访问整个HDFS。

#### 1.2.2 NameNode

NameNode 在 HDFS 结构模型里充当 Master 的就角色，因此一个 HDFS 集群里只会有一个 active 的 NameNode 节点。在集群里主要用来处理客户端的读写请求，它主要负责管理命名空间(NameSpace) 和文件 Block 映射信息。

##### NameSpace

NameSpace 维护着文件系统树(FileSystem Tree)和文件树上的所有文件及文件夹的元数据(metadata)，并使用 fsimage 和 editlog 这两个文件来管理这些信息。

fsimage(空间镜像文件)，它是文件系统元数据的一个完整的永久检查点，内部维护的是最近一次检查点的文件系统树和整棵树内部的所有文件和目录的元数据，如修改时间，访问时间，访问权限，副本数据，块大小，文件的块列表信息等等。

editlog(编辑日志文件)，当 HDFS 系统发生打开、关闭、创建、删除、重命名等操作产生的信息除了会保存在内存中外，还会持久化到编辑日志文件。比如上传一个文件后，日志文件里记录的有这次事务的txid, 文件的 inode id, 数据块的副本数, 数据块的id, 数据块大小, 访问时间, 修改时间等。

##### Block 映射信息

作为一个 master，NameNode 需要记录文件的每个块所在 DataNode 的位置信息，也就是我们常说的元数据信息 metaData。但是由于 NameNode 并不进行持久化存储，因此 NameNode 需要管理 Block 到 DataNode 的映射信息。一般元数据主要是: [文件名 —> 数据块映射], [数据块 —> Datanode] 列表映射。

其中,`[文件名 —> 数据块映射]保存在磁盘上进行持久化存储，但是 NameNode 并不保存[数据块 —> DataNode]列表映射，这份列表是通过心跳机制(heartbeat)建立起来的`。NameNode 执行文件系统的namespace 操作(如打开、关闭、重命名文件和目录)的同时决定了文件数据块到具体 DataNode 节点的映射。

##### HDFS 心跳机制

由于 HDFS 是 master/slave 结构，其中 master 包括 namenode 和 resourcemanager，slave 包括 datanode 和 nodemanager。

在 master 启动时会开启一个 IPC 服务，然后等待slave 连接。当 slave 启动后，会主动以默认3秒一次的频率链接 IPC 服务。当然这个时间是可以调整的，这个每隔一段时间连接一次的机制，就是心跳机制(默认的 `heartbeat.recheck.interval` 大小为 5 分钟，`dfs.heartbeat.interval` 默认的大小为 3 秒)。slave 通过心跳给 master 汇报自己信息，master 通过心跳下达命令。

具体来说就是：Namenode 通过心跳得知 DataNode 状态，Resourcemanager 通过心跳得知nodemanager 状态, 如果 master 很长时间没有收到 slave 信息时，就认为 slave 挂掉了。

这个判断挂掉的时间计算公式：2recheck+10heartbeat（Recheck 的时间单位为毫秒，heartbeat的时间单位为秒 ），默认为10分钟30秒 。

举例：如果 heartbeat.recheck.interval 设置为 6000(ms), dfs.heartbeat.interval设置为 5(秒), 则总的超时时间为 62 秒。

NameNode 作用小结：

- 管理命名空间 NameSpace;
- 管理 Block 映射信息；
- 配置副本策略；
- 处理客户端的读写请求。

#### 1.2.3 DataNode

DataNode 在 HDFS 结构中是 Slave 的角色，在整个 HDFS 集群里 DataNode 是有很多的。DataNode 负责数据块的存储和读取，数据块存储在 DataNode 节点所在的本地文件系统中 (包括block 和 block meta), 并且它会利用心跳机制定期向 NameNode 发送自己所存储的 block 块映射列表。

##### Block 数据块

数据块是指存储在 HDFS 中的最小单元，Hadoop 默认会的每个 Block 在多个 DataNode 节点存储3 份副本，每个数据块默认 128MB。

> 这里为什么是 128 MB 呢?
>
> 默认为 128M 是基于最佳传输损耗理论。不论对磁盘的文件进行读还是写，都需要先进行寻址。最佳传输损耗理论：在一次传输中，寻址时间占用总传输时间的 1% 时，本次传输的损耗最小，为最佳性价比传输。
>
> ```
> 目前硬件的发展条件，普通磁盘写的速率大概为 100 M/S, 寻址时间一般为 10 ms。
> 寻址时间: 10ms => 理论最佳传输时间: 10/0.01=1000ms=1s 
> 最佳传输大小: 100m/s x 1s = 100m
> 理论最佳传输文件大小为: 100mb
> Hadoop 数据块在传输时，每 64kb 还需要校验一次，因此 block_size 必须是 2 的 n 次方, 2 的 n次方整数中最接近 100 的就是 128mb
> ```
>
> 如果公司使用的是固态硬盘，写的速度是300M/S，将块大小调整到 256M
>
> 如果公司使用的是固态硬盘，写的速度是500M/S，将块大小调整到 512M

DataNode 作用小结：

- 存储实际的 Block 数据块
- 执行数据块的读/写操作

#### 1.2.4 SecondaryNameNode

SecondaryNameNode 不是 NameNode 的热备份，因为当 NameNode 停止服务时，它不能很快的替换 NameNode。它更像是咱们现实生活中的老板 NameNode 的秘书，平时整理整理文档，帮老板分担工作。

`它主要是用来辅助 NameNode 进行 fsimage 和 editlog 的合并工作，可以减少 editlog 的文件大小。这样就可以节省 NameNode 的重启时间，可以尽快的退出安全模式。`

##### checkpoint 检查点机制：

editlog 和 fsimage 两个文件的合并周期，被称为检查点机制(checkpoint)。

fsimage 文件是文件系统元数据的持久化检查点，不会在写操作后马上更新，因为 fsimage 写非常慢。由于 editlog 不断增长，在 NameNode 重启时，会造成长时间 NameNode 处于安全模式，不可用状态，是非常不符合 Hadoop 的设计初衷。所以要周期性合并 editlog，但是这个工作由 Namenode 来完成，会占用大量资源，这样就出现了 SecondaryNamenode，它可以进行 image 检查点的处理工作。

checkpoint 具体步骤如下：

- SNN(SecondaryNamenode) 请求 NN(Namenode) 进行 editlog 的滚动(创建一个新的 editlog)，然后将新的编辑操作记录到新生成的 editlog 文件。
- 通过 http get 方式，读取 NN 上的 fsimage 和 edits 文件，到 SNN 上。
- 读取 fsimage 到内存中，即加载 fsimage 到内存，然后执行 edits 中所有操作，并生成一个新的 fsimage 文件(fsimage.ckpt临时文件)，也就是这个检查点被创建了。
- 通过 http post 方式，将 fsimage.ckpt 临时文件传送到 Namenode。
- Namenode 使用新的 fsimage 替换原来的 fsimage 文件(fsimage.ckpt重命名为fsimage)，并且用第一步创建的 edits(editlog) 替代原来的 edits 文件, 然后更新fsimage 文件的检查点时间。

SecondaryNameNode 作用小结：

- 辅助 NameNode，分担其部分工作

- 定期合并 fsimage 和 fsedits，并推送给 NameNode

- 在紧急情况下，可辅助恢复NameNode

## 二、HDFS 读写流程

HDFS 的数据读写都分为很多步骤，想要轻松的掌握它，有个简单的小技巧--从粗略到细致。也就是我们先知道大致的流程，然后再把每一步进行细化，最终全部掌握。

开始掌握读写流程前，需要先了解3个基本概念：

- block

  前面也提到过 block 块, 它是 Hadoop 中数据存储的最小单位, 在进行文件上传前, client 会对文件进行分块, 分得的块就是 block，默认128M, 这是在综合考虑寻址时间和传输效率的的情况下得出的最佳大小。

- packet

  packet 是 client 向 DataNode 传输数据时候的基本单位，默认 64KB。

- chunk

  chunk 是进行数据校验的基本单位, 默认 512Byte，加上 4Byte 的校验位，实际上 chunk 写入 packet 的大小为 516Byte，常见于 client 向 DataNode 进行的数据校验。

### 2.1 读流程

首先读操作的命令/API：

```sh
# shell
$ hdfs dfs -get /file02 ./file02    
$ hdfs dfs -copyToLocal  /file02 ./file02 

# API
FSDataInputStream fsis = fs.open(path);    
fsis.read(byte[] a);
fs.copyToLocal(path1,path2); 
```

具体流程详解, 如下图所示是整个读流程及原理（上传）

![](https://img-blog.csdnimg.cn/20200312172516793.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70)



**粗略流程：**

1. client 向 namenode 请求 block 所在的 datanode 节点列表。
2. client 从最近位置逐个依次从 datanode 中读取 block 信息。
3. 整个通过 io 流读取的过程需要校验每个快信息。
4. 读取完成，关闭所有流。

**细致流程:**

1. 首先调用 FileSystem 的 open 方法获取一个 DistributedFileSystem 实例。
2. 然后 DistributedFileSystem 实例通过 RPC 在 NameNode 里获得文件的第一批 block 的locations(可能是需要读取文件的全部，也可能是一部分), 同一个 block 会按照在 DataNode的重复数返回多个 locations。
3. 返回的多个 locations 会按照 Hadoop 拓扑结构排序，按照就近原则来排序。
4. 前面三步结束后会返回一个 FSDataInputStream 对象，通过调用 read 方法时，该对象会找出离客户端最近的 DataNode 并与之建立连接。
5. 数据通过 IO 流从 DataNode 源源不断地流向客户端。
6. 如果第一个 block 块数据读取完成，就会关闭指向第一个 block 块的 DataNode 连接，接着读取下一个 block 块，直到把这一批的 block 块数据读取完成。
7. 每读取完一个 block 块都会进行 checksum 验证(校验每个块的信息通过偏移量和预写值对比，写的时候是校验 packet 的信息)，`如果读取 DataNode 时出现错误，客户端会通知 NameNode`，然后再从下一个拥有该 block 拷贝的 DataNode 继续读。
8. 如果第一批 blocks 读取完成，且文件读取还没有结束，也就是文件还没读完。FSDataInputStream 就会向 NameNode 获取下一批 blocks 的 locations，然后重复上面的步骤，直到所有 blocks 读取完成，这时就会关闭所有的流。
   

### 2.2 写流程

首先写操作的shell命令/API：

```sh
# shell 
$ hdfs dfs -put ./file02 /file02   
$ hdfs dfs -copyFromLocal  ./file02 /file02   

# API
FSDataOutputStream fsout = fs.create(path);
fsout.write(byte[]);
fs.copyFromLocal(path1,path2);
```

具体流程详解, 如下图所示是整个写流程及原理

![](https://img-blog.csdnimg.cn/20200312172424343.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2ODAzNzk1,size_16,color_FFFFFF,t_70)

**粗略流程:**

1. client 向 NameNode 发送写文件请求。
2. NameNode 检查文件，如果通过就返回输出流对象。
3. client 切分文件并且把数据和 NameNode 返回的 DataNode 列表一起发送给最近的一个DataNode 节点。
4. DataNode 写完之后返回确认信息。
5. 数据全部写完，关闭输入输出流，并发送完成信号给 NameNode。

**细致流程:**

1. 客户端使用 Configuration 类加载配置文件信息，然后调用 FileSystem#get()，获取一个分布式文件系统对象 DistributedFileSystem。然后通过调用DistributedFileSystem #create()，向 NameNode 发送写文件请求。
2. 客户端通过 RPC 与 NameNode 进行通信，NameNode 需要经过各种不同的检查，比如命名空间里该路径文件是否存在，客户端是否有相应权限。如果没有通过，返回 IOException，反之如果检查通过，NameNode 就会在命名空间下新建该文件(此时新文件大小为0字节)，并记录元数据，返回一个 FSDataOutputStream 输出流对象。
3. FSDataOutputStream 封装了一个 DFSDataOutputStream 对象，由该对象负责处理datanode 和 namenode 之间的通信。(DistributedFileSystem#create()会调用DFSClient#create() 方法创建 DFSOutputStream 输出流并构造一个HdfsDataOutputStream 来包装 DFSOutputStream)。
4. 客户端把数据按照 block 块进行切分。
5. 然后调用 DFSOutputStream#create()方法，开始执行写入操作（FSDataOutputStream封装了一个DFSDataOutputStream对象，由该对象负责处理 datanode 和 namenode 之间的通信），DFSOutputStream 会把数据切成一个个小 packet，然后排成队列 dataQueue
6. DataStreamer（DFSOutputStream的内部线程类）会去处理 dataQueue，它先问询 NameNode 这个新的 block 最适合存储的在哪几个 DataNode 里，比如副本数是3，那么就找到 3 个最适合的 DataNode，把它们排成一个 pipeline。DataStreamer 把 packet 按队列输出到管道的第一个 DataNode 的内存中，然后第一个 DataNode又把 packet 输出到第二个 DataNode 中，以此类推。
7. 在 DataStreamer 将 packet 写入 pipeline 时，同时也会将该 packet 存储到另外一个由 ResponseProcessor 线程管理的缓存队列 ackqueue 确认队列中。ResponseProcessor 线程会等待 DataNode 的确认响应。当收到所有的 DataNode 的确认信息后，该线程再将 ackqueue里的 packet 删除。
8. 如果写入期间发生故障，会首先关闭 pipeline，把 ackqueue 的所有 packet 都放回dataqueue 的最前端，以确保故障节点后的节点不会漏掉任一 packet。同时，会标识正常的DataNode，方便在故障节点恢复后，删除错误的部分数据块。然后从管线中删除故障节点，基于新的DataNode 构建一个新的管线。
9. 在一个 block 块大小的 n 个 packet 数据包写完后，客户端会调用 FSDataOutputStream 的 close 方法关闭写入流，当然在调用 close 之前，DataNode 会将内存中的数据写入本地磁盘。
10. 最后 DataStreamer 会继续向 NameNode 请求下一个块的 DataNode 列表，开始下一个块的写入。直到写完整个文件的最后一个块数据，然后客户端通知 NameNode 把文件标示为已完成，到这里整个写入过程就结束了。
    

> 在 HDFS 的写流程有几个核心问题:
>
> Q1: 传输 BLOCK1 的过程中, dn3 如果宕机了, 集群会怎么处理？
>
> A: 不做任何处理，错误会向 nn 报告
>
> Q2: 接Q1, 如果dn3又启动了，集群会如何处理？
>
> A: dn3 启动时，会向 nn 发送块报告，然后 nn 指示 dn3 删除 BLOCK1 (因为传输数据不完整)
>
> Q3: 客户端建立通道时，发现 dn3 连接不上，会怎么办？
>
> A: nn 会重新分配三个节点
>
> Q4: 传输过程中，packet 出错，会如何处理？
>
> A: 会重新上传，但是重传次数只有4次，超过限制则提示传输失败
>
> Q5: 如果 BLOCK1 上传成功, BLOCK2 坏了, 或者 BLOCK2 上传时，dn1 宕机, 如何处理？
>
> A: nn 会将整个文件标记为无效，下次 dn 向 nn 发送块报告时，nn 会通知这些块所在的节点删除

