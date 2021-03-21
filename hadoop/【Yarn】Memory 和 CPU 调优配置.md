# 【Yarn】Memory 和 CPU 调优配置

Yarn 作为一个资源调度器，应该考虑到集群里面每一台机子的计算资源，然后根据 application 申请的资源进行分配 Container。Container 是 YARN 里面资源分配的基本单位，具有一定的内存以及CPU资源。

在 YARN 集群中，平衡内存、CPU、磁盘的资源的很重要的，根据经验，每两个 Container 使用一块磁盘以及一个 CPU 核的时候可以使集群的资源得到一个比较好的利用。



## 一、内存配置

关于内存相关的配置可以参考hortonwork公司的文档 [Determine HDP Memory Configuration Settings](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdocs.hortonworks.com%2FHDPDocuments%2FHDP2%2FHDP-2.1.1%2Fbk_installing_manually_book%2Fcontent%2Frpm-chap1-11.html) 来配置你的集群。

> YARN 以及 MAPREDUCE 所有可用的内存资源应该要除去系统运行需要的以及其他的 hadoop 的一些程序，总共保留的内存=系统内存+HBASE内存。

可以参考下面的表格确定应该保留的内存：

| 每台机子内存 | 系统需要的内存 | HBase需要的内存 |
| ------------ | -------------- | --------------- |
| 4GB          | 1GB            | 1GB             |
| 8GB          | 2GB            | 1GB             |
| 16GB         | 2GB            | 2GB             |
| 24GB         | 4GB            | 4GB             |
| 48GB         | 6GB            | 8GB             |
| 64GB         | 8GB            | 8GB             |
| 72GB         | 8GB            | 8GB             |
| 96GB         | 12GB           | 16GB            |
| 128GB        | 24GB           | 24GB            |
| 255GB        | 32GB           | 32GB            |
| 512GB        | 64GB           | 64GB            |

计算每台机子最多可以拥有多少个 container，可以使用下面的公式:

containers = min (2*CORES, 1.8*DISKS, (Total available RAM) / MIN_CONTAINER_SIZE)

说明：

- CORES为机器CPU核数
- DISKS为机器上挂载的磁盘个数
- Total available RAM为机器总内存
- MIN_CONTAINER_SIZE是指container最小的容量大小，这需要根据具体情况去设置，可以参考下面的表格：

| 每台机子可用的RAM | container最小值 |
| ----------------- | --------------- |
| 小于4GB           | 256MB           |
| 4GB到8GB之间      | 512MB           |
| 8GB到24GB之间     | 1024MB          |
| 大于24GB          | 2048MB          |

每个container的平均使用内存大小计算方式为：

RAM-per-container = max(MIN_CONTAINER_SIZE, (Total Available RAM) / containers))

通过上面的计算，YARN以及MAPREDUCE可以这样配置：

| 配置文件              | 配置设置                             | 默认值    | 计算值                           |
| --------------------- | ------------------------------------ | --------- | -------------------------------- |
| yarn-site.xml         | yarn.nodemanager.resource.memory-mb  | 8192 MB   | = containers * RAM-per-container |
| yarn-site.xml         | yarn.scheduler.minimum-allocation-mb | 1024MB    | = RAM-per-container              |
| yarn-site.xml         | yarn.scheduler.maximum-allocation-mb | 8192 MB   | = containers * RAM-per-container |
| yarn-site.xml (check) | yarn.app.mapreduce.am.resource.mb    | 1536 MB   | = 2 * RAM-per-container          |
| yarn-site.xml (check) | yarn.app.mapreduce.am.command-opts   | -Xmx1024m | = 0.8 * 2 * RAM-per-container    |
| mapred-site.xml       | mapreduce.map.memory.mb              | 1024 MB   | = RAM-per-container              |
| mapred-site.xml       | mapreduce.reduce.memory.mb           | 1024 MB   | = 2 * RAM-per-container          |
| mapred-site.xml       | mapreduce.map.java.opts              |           | = 0.8 * RAM-per-container        |
| mapred-site.xml       | mapreduce.reduce.java.opts           |           | = 0.8 * 2 * RAM-per-container    |

举个例子：对于128G内存、32核CPU的机器，挂载了7个磁盘，根据上面的说明，系统保留内存为24G，不适应HBase情况下，系统剩余可用内存为104G，计算containers值如下：

containers = min (2*32, 1.8* 7 , (128-24)/2) = min (64, 12.6 , 51) = 13

计算RAM-per-container值如下：

RAM-per-container = max (2, (124-24)/13) = max (2, 8) = 8

你也可以使用脚本[yarn-utils.py](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdocs.hortonworks.com%2FHDPDocuments%2FHDP2%2FHDP-2.1.1%2Fbk_installing_manually_book%2Fcontent%2Frpm-chap1-9.html)来计算上面的值：

点击(此处)折叠或打开

1. \#!/usr/bin/env python
2. import optparse
3. from pprint import pprint
4. import logging
5. import sys
6. import math
7. import ast
8.  
9. ''' Reserved for OS + DN + NM, Map: Memory => Reservation '''
10. reservedStack = { 4:1, 8:2, 16:2, 24:4, 48:6, 64:8, 72:8, 96:12, 
11. ​          128:24, 256:32, 512:64}
12. ''' Reserved for HBase. Map: Memory => Reservation '''
13.  
14. reservedHBase = {4:1, 8:1, 16:2, 24:4, 48:8, 64:8, 72:8, 96:16, 
15. ​          128:24, 256:32, 512:64}
16. GB = 1024
17.  
18. def getMinContainerSize(memory):
19.  if (memory <= 4):
20.   return 256
21.  elif (memory <= 8):
22.   return 512
23.  elif (memory <= 24):
24.   return 1024
25.  else:
26.   return 2048
27.  pass
28.  
29. def getReservedStackMemory(memory):
30.  if (reservedStack.has_key(memory)):
31.   return reservedStack[memory]
32.  if (memory <= 4):
33.   ret = 1
34.  elif (memory >= 512):
35.   ret = 64
36.  else:
37.   ret = 1
38.  return ret
39.  
40. def getReservedHBaseMem(memory):
41.  if (reservedHBase.has_key(memory)):
42.   return reservedHBase[memory]
43.  if (memory <= 4):
44.   ret = 1
45.  elif (memory >= 512):
46.   ret = 64
47.  else:
48.   ret = 2
49.  return ret
50. ​          
51. def main():
52.  log = logging.getLogger(__name__)
53.  out_hdlr = logging.StreamHandler(sys.stdout)
54.  out_hdlr.setFormatter(logging.Formatter(' %(message)s'))
55.  out_hdlr.setLevel(logging.INFO)
56.  log.addHandler(out_hdlr)
57.  log.setLevel(logging.INFO)
58.  parser = optparse.OptionParser()
59.  memory = 0
60.  cores = 0
61.  disks = 0
62.  hbaseEnabled = True
63.  parser.add_option('-c', '--cores', default = 16,
64. ​           help = 'Number of cores on each host')
65.  parser.add_option('-m', '--memory', default = 64, 
66. ​          help = 'Amount of Memory on each host in GB')
67.  parser.add_option('-d', '--disks', default = 4, 
68. ​          help = 'Number of disks on each host')
69.  parser.add_option('-k', '--hbase', default = "True",
70. ​          help = 'True if HBase is installed, False is not')
71.  (options, args) = parser.parse_args()
72.  
73.  cores = int (options.cores)
74.  memory = int (options.memory)
75.  disks = int (options.disks)
76.  hbaseEnabled = ast.literal_eval(options.hbase)
77.  
78.  log.info("Using cores=" + str(cores) + " memory=" + str(memory) + "GB" +
79. ​      " disks=" + str(disks) + " hbase=" + str(hbaseEnabled))
80.  minContainerSize = getMinContainerSize(memory)
81.  reservedStackMemory = getReservedStackMemory(memory)
82.  reservedHBaseMemory = 0
83.  if (hbaseEnabled):
84.   reservedHBaseMemory = getReservedHBaseMem(memory)
85.  reservedMem = reservedStackMemory + reservedHBaseMemory
86.  usableMem = memory - reservedMem
87.  memory -= (reservedMem)
88.  if (memory < 2):
89.   memory = 2
90.   reservedMem = max(0, memory - reservedMem)
91.   
92.  memory *= GB
93.  
94.  containers = int (min(2 * cores,
95. ​             min(math.ceil(1.8 * float(disks)),
96. ​               memory/minContainerSize)))
97.  if (containers <= 2):
98.   containers = 3
99.  
100.  log.info("Profile: cores=" + str(cores) + " memory=" + str(memory) + "MB"
101. ​      \+ " reserved=" + str(reservedMem) + "GB" + " usableMem="
102. ​      \+ str(usableMem) + "GB" + " disks=" + str(disks))
103.   
104.  container_ram = abs(memory/containers)
105.  if (container_ram > GB):
106.   container_ram = int(math.floor(container_ram / 512)) * 512
107.  log.info("Num Container=" + str(containers))
108.  log.info("Container Ram=" + str(container_ram) + "MB")
109.  log.info("Used Ram=" + str(int (containers*container_ram/float(GB))) + "GB")
110.  log.info("Unused Ram=" + str(reservedMem) + "GB")
111.  log.info("yarn.scheduler.minimum-allocation-mb=" + str(container_ram))
112.  log.info("yarn.scheduler.maximum-allocation-mb=" + str(containers*container_ram))
113.  log.info("yarn.nodemanager.resource.memory-mb=" + str(containers*container_ram))
114.  map_memory = container_ram
115.  reduce_memory = 2*container_ram if (container_ram <= 2048) else container_ram
116.  am_memory = max(map_memory, reduce_memory)
117.  log.info("mapreduce.map.memory.mb=" + str(map_memory))
118.  log.info("mapreduce.map.java.opts=-Xmx" + str(int(0.8 * map_memory)) +"m")
119.  log.info("mapreduce.reduce.memory.mb=" + str(reduce_memory))
120.  log.info("mapreduce.reduce.java.opts=-Xmx" + str(int(0.8 * reduce_memory)) + "m")
121.  log.info("yarn.app.mapreduce.am.resource.mb=" + str(am_memory))
122.  log.info("yarn.app.mapreduce.am.command-opts=-Xmx" + str(int(0.8*am_memory)) + "m")
123.  log.info("mapreduce.task.io.sort.mb=" + str(int(0.4 * map_memory)))
124.  pass
125.  
126. if __name__ == '__main__':
127.  try:
128.   main()
129.  except(KeyboardInterrupt, EOFError):
130.   print("\nAborting ... Keyboard Interrupt.")
131.   sys.exit(1)

 

执行下面命令：

```
python yarn-utils.py -c 32 -m 128 -d 7 -k False
```

 

返回结果如下：

点击(此处)折叠或打开

1. Using cores=32 memory=128GB disks=7 hbase=False
2.  Profile: cores=32 memory=106496MB reserved=24GB usableMem=104GB disks=7
3.  Num Container=13
4.  Container Ram=8192MB
5.  Used Ram=104GB
6.  Unused Ram=24GB
7.  yarn.scheduler.minimum-allocation-mb=8192
8.  yarn.scheduler.maximum-allocation-mb=106496
9.  yarn.nodemanager.resource.memory-mb=106496
10.  mapreduce.map.memory.mb=8192
11.  mapreduce.map.java.opts=-Xmx6553m
12.  mapreduce.reduce.memory.mb=8192
13.  mapreduce.reduce.java.opts=-Xmx6553m
14.  yarn.app.mapreduce.am.resource.mb=8192
15.  yarn.app.mapreduce.am.command-opts=-Xmx6553m
16.  mapreduce.task.io.sort.mb=3276

 

 

这样的话，每个container内存为8G，似乎有点多，我更愿意根据集群使用情况任务将其调整为2G内存，则集群中下面的参数配置值如下：

| 配置文件              | 配置设置                             | 计算值             |
| --------------------- | ------------------------------------ | ------------------ |
| yarn-site.xml         | yarn.nodemanager.resource.memory-mb  | = 52 * 2 =104 G    |
| yarn-site.xml         | yarn.scheduler.minimum-allocation-mb | = 2G               |
| yarn-site.xml         | yarn.scheduler.maximum-allocation-mb | = 52 * 2 = 104G    |
| yarn-site.xml (check) | yarn.app.mapreduce.am.resource.mb    | = 2 * 2=4G         |
| yarn-site.xml (check) | yarn.app.mapreduce.am.command-opts   | = 0.8 * 2 * 2=3.2G |
| mapred-site.xml       | mapreduce.map.memory.mb              | = 2G               |
| mapred-site.xml       | mapreduce.reduce.memory.mb           | = 2 * 2=4G         |
| mapred-site.xml       | mapreduce.map.java.opts              | = 0.8 * 2=1.6G     |
| mapred-site.xml       | mapreduce.reduce.java.opts           | = 0.8 * 2 * 2=3.2G |

对应的xml配置为：

点击(此处)折叠或打开

1. <property>
2.    <name>yarn.nodemanager.resource.memory-mb</name>
3.    <value>106496</value>
4.  </property>
5.  <property>
6.    <name>yarn.scheduler.minimum-allocation-mb</name>
7.    <value>2048</value>
8.  </property>
9.  <property>
10.    <name>yarn.scheduler.maximum-allocation-mb</name>
11.    <value>106496</value>
12.  </property>
13.  <property>
14.    <name>yarn.app.mapreduce.am.resource.mb</name>
15.    <value>4096</value>
16.  </property>
17.  <property>
18.    <name>yarn.app.mapreduce.am.command-opts</name>
19.    <value>-Xmx3276m</value>
20.  </property>

另外，还有一下几个参数：

- yarn.nodemanager.vmem-pmem-ratio：任务每使用1MB物理内存，最多可使用虚拟内存量，默认是2.1。
- yarn.nodemanager.pmem-check-enabled：是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是true。
- yarn.nodemanager.vmem-pmem-ratio：是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是true。

第一个参数的意思是当一个map任务总共分配的物理内存为2G的时候，该任务的container最多内分配的堆内存为1.6G，可以分配的虚拟内存上限为2*2.1=4.2G。另外，照这样算下去，每个节点上YARN可以启动的Map数为104/2=52个。



# CPU配置

YARN中目前的CPU被划分成虚拟CPU（CPU virtual Core），这里的虚拟CPU是YARN自己引入的概念，初衷是，考虑到不同节点的CPU性能可能不同，每个CPU具有的计算能力也是不一样的，比如某个物理CPU的计算能力可能是另外一个物理CPU的2倍，这时候，你可以通过为第一个物理CPU多配置几个虚拟CPU弥补这种差异。用户提交作业时，可以指定每个任务需要的虚拟CPU个数。

在YARN中，CPU相关配置参数如下：

- yarn.nodemanager.resource.cpu-vcores：表示该节点上YARN可使用的虚拟CPU个数，默认是8，注意，目前推荐将该值设值为与物理CPU核数数目相同。如果你的节点CPU核数不够8个，则需要调减小这个值，而YARN不会智能的探测节点的物理CPU总数。
- yarn.scheduler.minimum-allocation-vcores：单个任务可申请的最小虚拟CPU个数，默认是1，如果一个任务申请的CPU个数少于该数，则该对应的值改为这个数。
- yarn.scheduler.maximum-allocation-vcores：单个任务可申请的最多虚拟CPU个数，默认是32。

对于一个CPU核数较多的集群来说，上面的默认配置显然是不合适的，在我的测试集群中，4个节点每个机器CPU核数为31，留一个给操作系统，可以配置为：

点击(此处)折叠或打开

1. <property>
2.    <name>yarn.nodemanager.resource.cpu-vcores</name>
3.    <value>31</value>
4.  </property>
5.  <property>
6.    <name>yarn.scheduler.maximum-allocation-vcores</name>
7.    <value>124</value>
8.  </property>

把CDH搭建起来了，跑其中的例子程序word-count。在控制台界面一直显示map 0%  reduce 0% ， 通过web页面查看job的状态一直是run,但是map没有执行。感觉是是资源的分配有问题。接着查看了任务的日志。

```
2014-07-04 17:30:37,492 INFO [RMCommunicator Allocator] org.apache.hadoop.mapreduce.v2.app.rm.RMContainerAllocator: Recalculating schedule, headroom=0
2014-07-04 17:30:37,492 INFO [RMCommunicator Allocator] org.apache.hadoop.mapreduce.v2.app.rm.RMContainerAllocator: Reduce slow start threshold not met. completedMapsForReduceSlowstart 2
2014-07-04 17:30:38,496 INFO [RMCommunicator Allocator] org.apache.hadoop.mapreduce.v2.app.rm.RMContainerAllocator: Ramping down all scheduled reduces:0
2014-07-04 17:30:38,496 INFO [RMCommunicator Allocator] org.apache.hadoop.mapreduce.v2.app.rm.RMContainerAllocator: Going to preempt 0
```

 

日志中没有任何的错误，但是一直打印该信息，应该是RM资源分配不够。

YARN中，资源包括内存和CPU，资源的管理是由ResourceManager和NodeManager共同完成，ResourceManager负责所有节点资源的管理和调度。NodeManager负责进程所在结点资源的分配和隔离。ResourceManager将某个NodeManager上资源分配给任务。下面详细介绍其中的一些重要参数。



#### yarn.nodemanager.resource.memory-mb

每个节点可用的内存， 它的单 位是mb，默认是8G，用于供NodeManager分配的。我出现的问题是资源分配太小，只有1G。

yarn.scheduler.minimum-allocation-mb

单个任务可申请的最小内存，默认是1024mb,稍微大一点，避免小的资源浪费情况，我本机资源少，所以给他分配了512mb， 失败的原因也就是这个分配过大。

yarn.scheduler.maximum-allocation-mb

单个任务可申请的最大内存，默认是8192mb. 如果是spark任务的话，这里调大吧

mapreduce.map.memory.mb

每个map任务的物理内存限制,应该大于或等于yarn.scheduler.minimum-allocation-mb

mapreduce.reduce.memory.mb

每个reduce任务的物理内存限制

mapreduce.map.java.opts

每个map进程的jvm堆的大小

mapreduce.reduce.java.opts

每个reduce进程的jvm堆的大小

 

每个节点可以运行map数和redue输，由yarn.nodemanager.resource.memory-mb除于mapreduce.map.memory.mb和mapreduce.reduce.memory.mb得到



# [yarn资源memory与core计算配置](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fwww.cnblogs.com%2Fxjh713%2Fp%2F9855238.html)

 

yarn调度分配主要是针对Memory与CPU进行管理分配,并将其组合抽象成container来管理计算使用

 

 

**memory配置**

　　计算每台机子最多可以拥有多少个container:

　　　　 containers = min (2*CORES, 1.8*DISKS, (Total available RAM) / MIN_CONTAINER_SIZE) 

 

　　　说明：

　　　　　　CORES为机器CPU核数

　　　　　　*DISKS为机器上挂载的磁盘个数*

　　　　　　**Total available RAM为机器总内存
　　　　　　MIN_CONTAINER_SIZE是指container最小的容量大小，这需要根据具体情况去设置，可以参考下面的表格：**

　　　　        ![img](https://static.oschina.net/uploads/img/201811/28124240_hVuD.jpg)

　　每个container的平均使用内存大小计算方式为：

　　　　　　 RAM-per-container = max(MIN_CONTAINER_SIZE, (Total Available RAM) / containers)) 

 

　相关配置调整说明:

![复制代码](https://static.oschina.net/uploads/img/201811/28124240_faoE.gif)

```
（1）yarn.nodemanager.resource.memory-mb
    表示该节点上YARN可使用的物理内存总量，默认是8192（MB），注意，如果你的节点内存资源不够8GB，则需要调减小这个值，而YARN不会智能的探测节点的物理内存总量。

（2）yarn.nodemanager.vmem-pmem-ratio
    任务每使用1MB物理内存，最多可使用虚拟内存量，默认是2.1。

（3） yarn.nodemanager.pmem-check-enabled
    是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是true。

（4） yarn.nodemanager.vmem-check-enabled
    是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是true。

（5）yarn.scheduler.minimum-allocation-mb
    单个container可申请的最少物理内存量，默认是1024（MB），如果一个任务申请的物理内存量少于该值，则该对应的值改为这个数。

（6）yarn.scheduler.maximum-allocation-mb
    单个container可申请的最多物理内存量，默认是8192（MB）。
```

![复制代码](https://static.oschina.net/uploads/img/201811/28124240_faoE.gif)

　　![img](https://static.oschina.net/uploads/img/201811/28124240_C6wY.jpg)

　　默认情况下，YARN采用了线程监控的方法判断任务是否超量使用内存，一旦发现超量，则直接将其杀死。由于Cgroups对内存的控制缺乏灵活性（即任务任何时刻不能超过内存上限，如果超过，则直接将其杀死或者报OOM），而Java进程在创建瞬间内存将翻倍，之后骤降到正常值，这种情况下，采用线程监控的方式更加灵活（当发现进程树内存瞬间翻倍超过设定值时，可认为是正常现象，不会将任务杀死），因此YARN未提供Cgroups内存隔离机制。

 

 

**CPU配置**

　　在yarn中使用的是虚拟CPU,这里的虚拟CPU是YARN自己引入的概念，初衷是，考虑到不同节点的CPU性能可能不同，每个CPU具有的计算能力也是不一样的，比如某个物理CPU的计算能力可能是另外一个物理CPU的2倍，这时候，你可以通过为第一个物理CPU多配置几个虚拟CPU弥补这种差异。用户提交作业时，可以指定每个任务需要的虚拟CPU个数。在YARN中，CPU相关配置参数如下：

![复制代码](https://static.oschina.net/uploads/img/201811/28124240_faoE.gif)

```
(1)yarn.nodemanager.resource.cpu-vcores
    表示该节点上YARN可使用的虚拟CPU个数，默认是8，注意，目前推荐将该值设值为与物理CPU核数数目相同。
    如果你的节点CPU核数不够8个，则需要调减小这个值，而YARN不会智能的探测节点的物理CPU总数。

(2)yarn.scheduler.minimum-allocation-vcores
    单个任务可申请的最小虚拟CPU个数，默认是1，如果一个任务申请的CPU个数少于该数，则该对应的值改为这个数。

(3)yarn.scheduler.maximum-allocation-vcores
    单个任务可申请的最多虚拟CPU个数，默认是32。
```

![复制代码](https://static.oschina.net/uploads/img/201811/28124240_faoE.gif)

　　默认情况下，YARN是不会对CPU资源进行调度的，你需要配置相应的资源调度器让你支持,具体参看以下链接:

　　（1）[Hadoop YARN配置参数剖析（4）—Fair Scheduler相关参数](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdongxicheng.org%2Fmapreduce-nextgen%2Fhadoop-yarn-configurations-fair-scheduler%2F)

　　（2）[Hadoop YARN配置参数剖析（5）—Capacity Scheduler相关参数](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdongxicheng.org%2Fmapreduce-nextgen%2Fhadoop-yarn-configurations-capacity-scheduler%2F)

 

　　默认情况下，NodeManager不会对CPU资源进行任何隔离，你可以通过启用Cgroups让你支持CPU隔离。

由于CPU资源的独特性，目前这种CPU分配方式仍然是粗粒度的。举个例子，很多任务可能是IO密集型的，消耗的CPU资源非常少，如果此时你为它分配一个CPU，则是一种严重浪费，你完全可以让他与其他几个任务公用一个CPU，也就是说，我们需要支持更粒度的CPU表达方式。

　　借鉴亚马逊EC2中CPU资源的划分方式，即提出了CPU最小单位为EC2 Compute Unit（ECU），一个ECU代表相当于1.0-1.2 GHz 2007 Opteron or 2007 Xeon处理器的处理能力。YARN提出了CPU最小单位YARN Compute Unit（YCU），目前这个数是一个整数，默认是720，由参数yarn.nodemanager.resource.cpu-ycus-per-core设置，表示一个CPU core具备的计算能力（该feature在2.2.0版本中并不存在，可能增加到2.3.0版本中），这样，用户提交作业时，直接指定需要的YCU即可，比如指定值为360，表示用1/2个CPU core，实际表现为，只使用一个CPU core的1/2计算时间。注意，在操作系统层，CPU资源是按照时间片分配的，你可以说，一个进程使用1/3的CPU时间片，或者1/5的时间片。对于CPU资源划分和调度的探讨，可参考以下几个链接：

[https://issues.apache.org/jira/browse/YARN-1089](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fissues.apache.org%2Fjira%2Fbrowse%2FYARN-1089)

[https://issues.apache.org/jira/browse/YARN-1024](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fissues.apache.org%2Fjira%2Fbrowse%2FYARN-1024)

[Hadoop 新特性、改进、优化和Bug分析系列5：YARN-3](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdongxicheng.org%2Fmapreduce-nextgen%2Fhadoop-jira-yarn-3%2F)

本文汇总了几个hadoop yarn中常见问题以及解决方案，注意，本文介绍解决方案适用于hadoop 2.2.0以及以上版本。

 

**（1） 默认情况下，各个节点的负载不均衡（任务数目不同），有的节点很多任务在跑，有的没有任务，怎样让各个节点任务数目尽可能均衡呢？**

答： 默认情况下，资源调度器处于批调度模式下，即一个心跳会尽可能多的分配任务，这样，优先发送心跳过来的节点将会把任务领光（前提：任务数目远小于集群可以同时运行的任务数量），为了避免该情况发生，可以按照以下说明配置参数：

如果采用的是fair scheduler，可在yarn-site.xml中，将参数yarn.scheduler.fair.max.assign设置为1（默认是-1,）

如果采用的是capacity scheduler（默认调度器），则不能配置，目前该调度器不带负载均衡之类的功能。

当然，从hadoop集群利用率角度看，该问题不算问题，因为一般情况下，用户任务数目要远远大于集群的并发处理能力的，也就是说，通常情况下，集群时刻处于忙碌状态，没有节点一直空闲着。

 

**（2）某个节点上任务数目太多，资源利用率太高，怎么控制一个节点上的任务数目?**

答：一个节点上运行的任务数目主要由两个因素决定，一个是NodeManager可使用的资源总量，一个是单个任务的资源需求量，比如一个NodeManager上可用资源为8 GB内存，8 cpu，单个任务资源需求量为1 GB内存，1cpu，则该节点最多运行8个任务。

NodeManager上可用资源是由管理员在配置文件yarn-site.xml中配置的，相关参数如下：

yarn.nodemanager.resource.memory-mb：总的可用物理内存量，默认是8096

yarn.nodemanager.resource.cpu-vcores：总的可用CPU数目，默认是8

对于MapReduce而言，每个作业的任务资源量可通过以下参数设置：

mapreduce.map.memory.mb：物理内存量，默认是1024

mapreduce.map.cpu.vcores：CPU数目，默认是1

注：以上这些配置属性的详细介绍可参考文章：[Hadoop YARN配置参数剖析(1)—RM与NM相关参数](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdongxicheng.org%2Fmapreduce-nextgen%2Fhadoop-yarn-configurations-resourcemanager-nodemanager%2F)。

默认情况，各个调度器只会对内存资源进行调度，不会考虑CPU资源，你需要在调度器配置文件中进行相关设置，具体可参考文章：[Hadoop YARN配置参数剖析(4)—Fair Scheduler相关参数](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdongxicheng.org%2Fmapreduce-nextgen%2Fhadoop-yarn-configurations-fair-scheduler%2F)和[Hadoop YARN配置参数剖析(5)—Capacity Scheduler相关参数](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdongxicheng.org%2Fmapreduce-nextgen%2Fhadoop-yarn-configurations-capacity-scheduler%2F)。

 

**（3）如何设置单个任务占用的内存量和CPU数目？**

答：对于MapReduce而言，每个作业的任务资源量可通过以下参数设置：

mapreduce.map.memory.mb：物理内存量，默认是1024

mapreduce.map.cpu.vcores：CPU数目，默认是1

需要注意的是，默认情况，各个调度器只会对内存资源进行调度，不会考虑CPU资源，你需要在调度器配置文件中进行相关设置。

 

**（4） 用户给任务设置的内存量为1000MB，为何最终分配的内存却是1024MB？**

答：为了易于管理资源和调度资源，Hadoop YARN内置了资源规整化算法，它规定了最小可申请资源量、最大可申请资源量和资源规整化因子，如果应用程序申请的资源量小于最小可申请资源量，则YARN会将其大小改为最小可申请量，也就是说，应用程序获得资源不会小于自己申请的资源，但也不一定相等；如果应用程序申请的资源量大于最大可申请资源量，则会抛出异常，无法申请成功；规整化因子是用来规整化应用程序资源的，应用程序申请的资源如果不是该因子的整数倍，则将被修改为最小的整数倍对应的值，公式为ceil(a/b)*b，其中a是应用程序申请的资源，b为规整化因子。

以上介绍的参数需在yarn-site.xml中设置，相关参数如下：

yarn.scheduler.minimum-allocation-mb：最小可申请内存量，默认是1024

yarn.scheduler.minimum-allocation-vcores：最小可申请CPU数，默认是1

yarn.scheduler.maximum-allocation-mb：最大可申请内存量，默认是8096

yarn.scheduler.maximum-allocation-vcores：最大可申请CPU数，默认是4

对于规整化因子，不同调度器不同，具体如下：

FIFO和Capacity Scheduler，规整化因子等于最小可申请资源量，不可单独配置。

Fair Scheduler：规整化因子通过参数yarn.scheduler.increment-allocation-mb和yarn.scheduler.increment-allocation-vcores设置，默认是1024和1。

通过以上介绍可知，应用程序申请到资源量可能大于资源申请的资源量，比如YARN的最小可申请资源内存量为1024，规整因子是1024，如果一个应用程序申请1500内存，则会得到2048内存，如果规整因子是512，则得到1536内存。

 

**（5）我们使用的是Fairscheduler，配置了多个队列，当用户提交一个作业，指定的队列不存在时，Fair Scheduler会自动创建一个新队列而不是报错（比如报错：队列XXX不存在），如何避免这种情况发生？**

答：在yarn-site.xml中设置yarn.scheduler.fair.allow-undeclared-pools，将它的值配置为false（默认是true）。

 

**（6）使用Hadoop 2.0过程中，遇到了错误，怎样排查错误？**

答：从hadoop 日志入手，Hadoop日志存放位置可参考我这篇文章：[Hadoop日志到底存在哪里？](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fdongxicheng.org%2Fmapreduce-nextgen%2Fhadoop-logs-placement%2F)

 前言：hadoop2.x版本和hadoop1.x版本的一个区别就是：hadoop1.x中负责资源和作业调度的是MapReduce，hadoop2.x版本后，MapReduce只专注于计算，资源和作业的调度由YARN来负责。Container是YARN里面资源分配的基本单位，具有一定的内存以及CPU资源。我们的应用在工作的时候，需要消耗内存和CPU，故当YARN收到application申请，则会根据application申请的资源，分配Container。

  在YARN的NodeManager节点上，会将机器的CPU和内存的一定值抽离出来，抽离成虚拟的值，然后这些虚拟的值在根据配置组成多个Container，当application提出申请时，就会分配相应的Container资源。关于默认值我们可以查看官网，如下表所示。

参数   默认值
yarn.nodemanager.resource.memory-mb
-1
yarn.nodemanager.resource.cpu-vcores
-1
yarn.scheduler.minimum-allocation-mb
1024MB
yarn.scheduler.maximum-allocation-mb
8192MB
yarn.scheduler.minimum-allocation-vcores
1
yarn.scheduler.maximum-allocation-vcores
4
  内存配置

  yarn.nodemanager.resource.memory-mb默认值为-1，代表着YARN的NodeManager占总内存的80%。也就是说加入我们的机器为64GB内存，出去非YARN进程需要的20%内存，我们大概需要64*0.8≈51GB，在分配的时候，单个任务可以申请的默认最小内存为1G，任务量大的话可最大提高到8GB。

  在生产场景中，简单的配置，一般情况下：yarn.nodemanager.resource.memory-mb直接设置成我们需要的值，且要是最大和最小内存需求的整数倍；（一般Container容器中最小内存为4G，最大内存为16G）

  假如：64GB的机器内存，我们有51GB的内存可用于NodeManager分配，根据上面的介绍，我们可以直接将yarn.nodemanager.resource.memory-mb值为48GB，然后容器最小内存为4GB，最大内存为16GB，也就是在当前的NodeManager节点下，我们最多可以有12个容器，最少可以有3个容器。

  CPU配置

  此处的CPU指的是虚拟的CPU（CPU virtual core），之所以产生虚拟CPU（CPU vCore）这一概念，是因为物理CPU的处理能力的差异，为平衡这种差异，就引入这一概念。

  yarn.nodemanager.resource.cpu-vcores表示能够分配给Container的CPU核数，默认配置为-1，代表值为8个虚拟CPU，推荐该值的设置和物理CPU的核数数量相同，若不够，则需要调小该值。

  yarn.scheduler.minimum-allocation-vcores的默认值为1，表示每个Container容器在处理任务的时候可申请的最少CPU个数为1个。

  yarn.scheduler.maximum-allocation-vcores的默认值为4，表示每个Container容器在处理任务的时候可申请的最大CPU个数为4个。