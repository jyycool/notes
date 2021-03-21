# 【HDFS】FsImage、Editslog & Checkpoint



## 1. FsImage & Editslog

- `Editslog`

  保存了所有对 hdfs 中文件的操作信息

- `FsImage`

  是内存元数据在本地磁盘的映射，用于维护管理文件系统树，即元数据(metadata)

在 hdfs 中主要是通过两个数据结构 FsImage 和 EditsLog 来实现 metadata 的更新。在某次启动hdfs时，会从 FsImage 文件中读取当前 HDFS 文件的 metadata，之后对 HDFS 的操作步骤都会记录到 editslog 文件中。

metadata 信息就应该由 FSImage 文件和 editslog 文件组成。fsimage 中存储的信息就相当于整个 hdfs 在某一时刻的一个快照。

FsImage 文件和 EditsLog 文件可以通过 ID 来互相关联。如果是非HA集群的话，这两个数据文件保存在***dfs.namenode.name.dir***设置的路径下，会保存 FsImage 文件和 EditsLog 文件，如果是HA集群的话，EditsLog文件保存在参数***dfs.journalnode.edits.dir***设置的路径下，即edits文件由 journal 集群管理。

![hadoop_edits](/Users/sherlock/Desktop/notes/allPics/Hadoop/hadoop_edits.png)

在上图中 editslog 文件以 `edits_` 开头，后面跟一个 txid 范围段，并且多个 editslog 之间首尾相连，正在使用的 editslog 名字 `edits_inprogress_txid`。该路径下还会保存两个 fsimage 文件(`dfs.namenode.num.checkpoints.retained` 在namenode上保存的 fsimage 的数目，超出的会被删除, 默认保存2个), 文件格式为 fsimage_txid。上图中可以看出 fsimage 文件已经加载到了最新的一个 editslog 文件，仅仅只有 inprogress 状态的 edit log 未被加载。

在启动 HDFS 时，只需要读入 fsimage_0000000000000014590 以及edits_inprogress_0000000000000014591 就可以还原出当前 hdfs 的最新状况。FsImageid 总是比 editslogid小.

那么这两个文件是如何合并的呢？这就引入了checkpoint机制

## checkpoint

因为文件合并过程需要消耗 io 和 cpu 所以需要将这个过程独立出来，在 Hadoop1.x 中是由Secondnamenode 来完成，且 Secondnamenode 必须启动在单独的一个节点最好不要和 namenode 在同一个节点，这样会增加 namenode 节点的负担，而且维护时也比较方便。同样在HA集群中这个合并的过程是由 Standbynamenode 完成的。

### 1.1 合并的过程：

过程类似于TCP协议的关闭过程（四次挥手）

![hadoop_hebing](/Users/sherlock/Desktop/notes/allPics/Hadoop/hadoop_hebing.png)

1. 首先 Standbynamenode 进行判断是否达到 checkpoint 的条件（是否距离上次合并过了1小时或者事务条数是否达到100万条）
2. 当达到 checkpoint 条件后，Standbynamenode 会将 qjournal 集群中的 edits 和本地fsImage 文件合并生成一个文件 fsimage_ckpt_txid（此时的txid是与合并的editslog_txid的txid值相同），同时 Standbynamenode 还会生成一个 MD5 文件，并将fsimage_ckpt_txid 文件重命名为 fsimage_txid
3. 向 Activenamenode 发送 http 请求（请求中包含了 Standbynamenode 的域名，端口以及新fsimage_txid 的 txid），询问是否进行获取
4. Activenamenode 获取到请求后，会返回一个 http 请求来向 Standbynamenode 获取新的fsimage_txid，并保存为 fsimage.ckpt_txid，生成一个 MD5，最后再改名为fsimage_txid。合并成功

### 1.2 合并的时机

什么时候进行checkpoint呢？这由两个参数 ***dfs.namenode.checkpoint.preiod*** ***(默认值是3600s)*** 和 ***dfs.namenode.checkpoint.txns(默认值是1000000)***来决定

1. 距离上次checkpoint的时间间隔 ***( dfs.namenode.checkpoint.period )***
2. Edits中的事务条数达到 ***( dfs.namenode.checkpoint.txns )***

3. 事物条数又由 ***( dfs.namenode.checkpoint.check.period )(默认值是60）***决定，checkpoint节点隔 60 秒就会去统计一次 hdfs 的操作次数。
   

## 查看 edits 和 fsimage

将 fsimage 文件转换为 xml 文件查看

```shell
$ hdfs oiv -p XML -i fsimage_0000000000000014590 -o ~/mySpace/fsiamge_14590.xml
```

将 edits 文件转换为 xml 文件查看

```shell
$ hdfs oev -p XML -i edits_0000000000000014450-0000000000000014450 -o ~/mySpace/edits_14450.xml
```

