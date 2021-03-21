# 【Jvm】性能监控工具

java/bin 下自带的调优工具大致有 - `jps、jstack、jmap、jhat、jstat、hprof`

现实企业级Java开发中，有时候我们会碰到下面这些问题：

- 内存不足(out of memory)
- 内存泄露(memory leak)
- 线程死锁(death lock)
- 锁争用（Lock Contention）
- Java进程消耗CPU过高
- ......

这些问题在日常开发中可能被很多人忽视（比如有的人遇到上面的问题只是重启服务器或者调大内存，而不会深究问题根源），但能够理解并解决这些问题是Java程序员进阶的必备要求。本文将对一些常用的JVM性能调优监控工具进行介绍，希望能起抛砖引玉之用。本文参考了网上很多资料，难以一一列举，在此对这些资料的作者表示感谢！关于JVM性能调优相关的资料，请参考文末。

## 一、Overview

### 1.1 术语--堆外、堆和非堆内存

#### 1.1.1 堆外内存

堆外内存(DirectMemory, 又称为直接内存)是一块由程序本身管理的一块内存空间，它的效率要比标准内存池要高，主要用于存放网络通信时数据缓冲和磁盘数据交换时的数据缓冲。

在新的 Java8 JMM 中已将原来的方法区(永久代)移除, 改为使用元数据区, 这个元数据区本质就是一块堆外内存。

目前在 Java code 中接触到使用堆外内存的就是 ByteBuffer.allocateDirect(int) 可以直接分配一块堆外内存来使用, 并可以通过 `-XX:MaxDirectMemorySize ` 指定程序可使用的堆外内存的大小，此参数的含义是当 Direct ByteBuffer 分配的堆外内存到达指定大小后，即触发 Full GC。注意该值是有上限的，默认是64M，最大为sun.misc.VM.maxDirectMemory()，在程序中可以获得-XX:MaxDirectMemorySize的设置的值。 

#### 1.1.2 堆内存和非堆内存

按照官方的说法: Java 虚拟机具有一个堆，堆是运行时数据区域，所有类实例和数组的内存均从此处分配。堆是在 Java 虚拟机启动时创建的。JVM 中堆之外的内存称为非堆内存(Non-heap memory)。

可以看出JVM主要管理两种类型的内存：堆和非堆。

简单来说: 

1. 堆就是 Java代码可及的内存，是留给开发人员使用的

2. 非堆就是 JVM 留给自己用的，所以方法区、JVM 内部处理或优化所需的内存 (如JIT编译后的代码缓存)、每个类结构(如运行时常数池、字段和方法数据)以及方法和构造方法的代码都在非堆内存中。 

>JVM初始分配的堆外内存 64MB
>
>JVM最大允许分配的堆外内存 `-XX:MaxMemorySize`

#### 1.1.3 栈与堆

***栈解决程序的运行问题，即程序如何执行，或者说如何处理数据***；

***堆解决的是数据存储的问题，即数据怎么放、放在哪儿。***

在Java中一个线程就会相应有一个线程栈与之对应，这点很容易理解，因为不同的线程执行逻辑有所不同，因此需要一个独立的线程栈。而堆则是所有线程共享的。栈因为是运行单位，因此里面存储的信息都是跟当前线程（或程序）相关信息的。包括局部变量、程序运行状态、方法返回值等等；而堆只负责存储对象信息。

***Java的堆是一个运行时数据区, 类的对象从中分配空间***。这些对象通过`new`、`newArray`、`anewarray`和`multianewarray`等  指令建立，它们不需要程序代码来显式的释放。堆是由垃圾回收来负责的

***堆的优势是可以动态地分配内存大小***，生存期也不必事先告诉编译器，因为它是在运行时 动态分配内存的，Java的垃圾收集器会自动收走这些不再使用的数据。但缺点是，由于要在运行时动态分配内存，存取速度较慢。  栈的优势是，存取速度比堆要快，仅次于寄存器，栈数据可以共享。但缺点是，存在栈中的数据大小与生存期必须是确定的，缺乏灵活性。栈中主要存放一些基本类 型的变量（,int, short, long, byte, float, double, boolean, char）和对象句柄。

> ***JVM初始分配的堆内存 `-Xms512m`***
>
> ***JVM最大允许分配的堆内存 `-Xmx1024m`***
>
> ***JVM初始分配的非堆内存 `-XX:PermSize=64m`*** 
>
> ***JVM最大允许分配的非堆内存 `-XX:MaxPermSize=128m`***

### 1.2 什么是堆Dump

堆Dump是反应Java堆使用情况的内存镜像，其中主要包括系统信息、虚拟机属性、完整的线程Dump、所有类和对象的状态等。 一般，在内存不足、GC异常等情况下，我们就会怀疑有[内存泄露](http://zh.wikipedia.org/zh-cn/内存泄漏)。这个时候我们就可以制作堆Dump来查看具体情况。分析原因。

#### 基础知识

[Java虚拟机的内存组成以及堆内存介绍](http://www.hollischuang.com/archives/80) 

[Java GC工作原理](http://www.hollischuang.com/archives/76) 

常见内存错误：

> java.lang.OutOfMemoryError 老年代内存不足。
>
> java.lang.OutOfMemoryError: PermGen Space 永久代内存不足。
>
> java.lang.OutOfMemoryError: GC overhead limit exceed 垃圾回收时间占用系统运行时间的98%或以上。
>
> java.lang.OutOfMemoryError: java heap space JDK8 将字面量存入了 java heap, 所以可能会 heap 移除。



## 二、JDK的监控工具

### 2.1 jps    

jps 全称 Java Virtual Machine Process Status Tool 主要用来输出JVM中运行的进程状态信息。语法格式如下：

```sh
usage: jps [-help]
       jps [-q] [-mlvV] [<hostid>]

Definitions:
    <hostid>:      <hostname>[:<port>]

Options:：
		-q 不输出类名、Jar名和传入main方法的参数
		-m 输出传入main方法的参数
		-l 输出main类或Jar的全限名
		-v 输出传入JVM的参数
```

如果不指定hostid就默认为当前主机或服务器。

#### 2.1.1 jps 示例

```java
root@ubuntu:/# jps -m -l2458 org.artifactory.standalone.main.Main /usr/local/artifactory-2.2.5/etc/jetty.xml29920 com.sun.tools.hat.Main -port 9998 /tmp/dump.dat3149 org.apache.catalina.startup.Bootstrap start30972 sun.tools.jps.Jps -m -l8247 org.apache.catalina.startup.Bootstrap start25687 com.sun.tools.hat.Main -port 9999 dump.dat21711 mrf-center.jar
```

### 2.2 jstack

jstack 主要用来查看某个 Java 进程内的线程堆栈信息。语法格式如下：

```sh
Usage:
    jstack [-l] <pid>
        (to connect to running process)
    jstack -F [-m] [-l] <pid>
        (to connect to a hung process)
    jstack [-m] [-l] <executable> <core>
        (to connect to a core file)
    jstack [-m] [-l] [server_id@]<remote server IP or hostname>
        (to connect to a remote debug server)

Options:
    -F  强制线程转储。当进程挂起 jstack <pid> 没有响应时, 使用 -F
    -m  mixed mode，不仅会输出Java堆栈信息，还会输出C/C++堆栈信息（比如Native方法）
    -l  long listing. 打印出额外的锁信息，在发生死锁时可以用jstack -l pid来观察锁持有情况
    -h or -help to print this help message 
```

jstack可以定位到线程堆栈，根据堆栈信息我们可以定位到具体代码，所以它在JVM性能调优中使用得非常多。

#### 2.2.1 jstack 示例

下面我们来一个实例找出某个Java进程中最耗费CPU的Java线程并定位堆栈信息，用到的命令有 ps、top、printf、jstack、grep。

1. 先找出Java进程ID，我部署在服务器上的Java应用名称为mrf-center：

   ```sh
   root@ubuntu:/# ps -ef | grep mrf-center | grep -v greproot     
   21711     1  1 14:47 pts/3    00:02:10 java -jar mrf-center.jar
   ```

   得到 PID 为 21711

2. 找出该进程内最耗费CPU的线程，可以使用

   - `ps -Lfp <pid>`

   - `ps -mp <pid> -o THREAD, tid, time`

   - `top -Hp <pid>`

   ```sh
   ps -mp 19272 -o THREAD,tid,time
   USER     %CPU PRI SCNT WCHAN  USER SYSTEM   TID     TIME
   USER  171.8   -    - -         -      -     - 00:36:54
   USER    0.0  19    - futex_    -      - 21741 00:00:00
   USER  168.8  19    - futex_    -      - 21742 00:13:18
   USER    0.2  19    - -         -      - 21743 00:05:50
   USER    0.2  19    - -         -      - 21744 00:05:50
   USER    0.2  19    - -         -      - 21745 00:05:50
   USER    0.3  19    - -         -      - 21746 00:05:49
   USER    0.4  19    - futex_    -      - 21747 00:00:05
   USER    0.0  19    - futex_    -      - 21748 00:00:00
   USER    0.0  19    - futex_    -      - 21749 00:00:00
   USER    0.0  19    - futex_    -      - 21750 00:00:00
   USER    0.4  19    - futex_    -      - 21751 00:00:04
   USER    0.3  19    - futex_    -      - 21752 00:00:03
   ```

   TIME 列就是各个Java线程耗费的CPU时间，CPU时间最长的是线程ID为21742的线程

   将十进制的 TID 转十六进制, 得到 TID: 21742 的十六进制值为 54ee，下面会用到。

   ```sh
   printf "%x\n" 21742
   54ee
   ```

    

3. 使用 jstack 用它来输出 PID: 21711 的堆栈信息，然后根据 TID 的十六进制值 grep，如下：

   ```sh
   root@ubuntu:/# jstack 21711 | grep 54ee
   "PollIntervalRetrySchedulerThread" prio=10 tid=0x00007f950043e000 nid=0x54ee in Object.wait() [0x00007f94c6eda000]
   ```

   可以看到CPU消耗在 `PollIntervalRetrySchedulerThread` 这个类的 Object.wait()，我找了下我的代码，定位到下面的代码：

   ```java
   // Idle waitgetLog().info("Thread [" + getName() + "] is idle waiting...");
   schedulerThreadState = PollTaskSchedulerThreadState.IdleWaiting;long now = System.currentTimeMillis();long waitTime = now + getIdleWaitTime();long timeUntilContinue = waitTime - now;synchronized(sigLock) {	try {
       	if(!halted.get()) {
       		sigLock.wait(timeUntilContinue);
       	}
       } 	catch (InterruptedException ignore) {
       }
   }
   ```

   它是轮询任务的空闲等待代码，上面的 `sigLock.wait(timeUntilContinue)` 就对应了前面的`Object.wait()`。

### 2.3 jmap 和 jhat

主要用于打印指定Java进程(或核心文件、远程调试服务器)的共享对象内存映射或堆内存细节。[官方文档](https://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jmap.html), 一般结合jhat使用。

jmap语法格式如下：

```javascript
Usage:
    jmap [option] <pid>
        (to connect to running process)
    jmap [option] <executable <core>
        (to connect to a core file)
    jmap [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    <none>               打印与 Solaris pmap 相同的信息
    
    -heap                打印 java heap 摘要
    
    -histo[:live]        显示堆中历史对象的统计信息;如果进程还存活
                         如果指定了子选项live，只计算当前存活对象
                         
    -clstats             打印类加载器信息
    
    -finalizerinfo       打印等候GC回收的对象信息
    
    -dump:<dump-options> 生成 heap 存储快照dump文件
                         dump-options:
                           live         只 dump 活动对象;
                                        如果不指定,dump heap 中的所有对象。             
                           format=b     binary format
                           file=<file>  存储 dump 内容的文件                          
												 
                         示例: 
                         jmap -dump:live,format=b,file=heap.bin <pid>
                           
    -F                   force. 与 -dump:<dump-options> <pid>或-histo 
												 一起使用, 当<pid>进程未响应时，强制进行 heap dump 。
                         在这种模式下, 不支持“live”子选项。

    -h | -help           to print this help message
    -J<flag>             to pass <flag> directly to the runtime system
```

如果运行在 64位 JVM上，可能需要指定-J-d64命令选项参数。

```sh
jmap -J-d64 [option] <pid>
```

从安全点日志看，从 Heap Dump 开始，整个 JVM 都是停顿的，考虑到IO（虽是写到 Page Cache，但或许会遇到 background flush），几G 的 Heap 可能产生几秒的停顿，在生产环境上执行时谨慎再谨慎。

live 的选项，实际上是产生一次Full GC 来保证只看还存活的对象。有时候也会故意不加live选项，看历史对象。

Dump 出来的文件建议用JDK自带的 VisualVM 或 Eclipse 的 MAT 插件打开，对象的大小有两种统计方式：

- 本身大小(Shallow Size): 对象本来的大小。
- 保留大小(Retained Size): 当前对象大小 + 当前对象直接或间接引用到的对象的大小总和。

看本身大小时，占大头的都是char[] ,byte[]之类的，没什么意思（用jmap -histo:live pid 看的也是本身大小）。所以需要关心的是保留大小比较大的对象，看谁在引用这些char[], byte[]。

(MAT能看的信息更多，但 VisualVM 胜在JVM 自带，用法如下：命令行输入 jvisualvm，文件->装入->堆Dump－>检查 -> 查找20保留大小最大的对象，就会触发保留大小的计算，然后就可以类视图里浏览，按保留大小排序了)

#### 2.3.1 jmap 示例1

`jmap -heap <pid>`打印进程的类加载器和类加载器加载的持久代对象信息，输出：类加载器名称、对象是否存活（不可靠）、对象地址、父类加载器、已加载的类大小等信息，比如下面的例子：

```javascript
root@ubuntu:/# jmap -heap 21711
Attaching to process ID 21711, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 20.10-b01

using thread-local object allocation.
Parallel GC with 4 thread(s)
Heap Configuration:
   MinHeapFreeRatio = 40   
	 MaxHeapFreeRatio = 70  
   MaxHeapSize      = 2067791872 (1972.0MB)
   NewSize          = 1310720 (1.25MB)
   MaxNewSize       = 17592186044415 MB
   OldSize          = 5439488 (5.1875MB)
   NewRatio         = 2  
   SurvivorRatio    = 8   
   PermSize         = 21757952 (20.75MB)
   MaxPermSize      = 85983232 (82.0MB)
  Heap Usage:
PS Young Generation
Eden Space:
   capacity = 6422528 (6.125MB)
   used     = 5445552 (5.1932830810546875MB)
   free     = 976976 (0.9317169189453125MB)
   84.78829520089286% used
From Space:
   capacity = 131072 (0.125MB)
   used     = 98304 (0.09375MB)
   free     = 32768 (0.03125MB)
   75.0% used
To Space:
   capacity = 131072 (0.125MB)
   used     = 0 (0.0MB)
   free     = 131072 (0.125MB)
   0.0% used
PS Old Generation
   capacity = 35258368 (33.625MB)
   used     = 4119544 (3.9287033081054688MB)
   free     = 31138824 (29.69629669189453MB)
   11.683876009235595% used
PS Perm Generation
   capacity = 52428800 (50.0MB)
   used     = 26075168 (24.867218017578125MB)
   free     = 26353632 (25.132781982421875MB)
   49.73443603515625% used
   ....
```

解析:

```sh
// 新生代采用的是并行线程处理方式
using parallel threads in the new generation.  

// 使用线程 TLAB 局部缓存进行对象分配
using thread-local object allocation.   

// 使用并行度 4 的 Parallel GC 
Parallel GC with 4 thread(s) 

// 堆配置情况，也就是JVM参数配置的结果[Java程序配置JVM参数，就是在配置这些]
Heap Configuration:  
	 
	 // 最小堆使用比例
   MinHeapFreeRatio = 40 
   // 最大堆可用比例
   MaxHeapFreeRatio = 70 
   // 最大堆空间大小
   MaxHeapSize      = 2147483648 (2048.0MB) 
   // 新生代分配大小
   NewSize          = 268435456 (256.0MB) 
   // 最大可新生代分配大小
   MaxNewSize       = 268435456 (256.0MB) 
   // 老年代大小
   OldSize          = 5439488 (5.1875MB) 
   // 新生代比例
   NewRatio         = 2  
   // 新生代与suvivor的比例
   SurvivorRatio    = 8 
   // perm区 永久代大小
   PermSize         = 134217728 (128.0MB) 
   // 最大可分配perm区 也就是永久代大小
   MaxPermSize      = 134217728 (128.0MB) 

// 堆使用情况[堆内存实际的使用情况]
Heap Usage: 

// 新生代（Eden + survior(from , to) = (1+2)空间）
New Generation (Eden + 1 Survivor Space):  

   // 新生代总容量
   capacity = 241631232 (230.4375MB)  
   // 新生代已用容量
   used     = 77776272 (74.17323303222656MB) 
   // 新生代剩余容量
   free     = 163854960 (156.26426696777344MB) 
   // 新生代已用比例
   32.188004570534986% used 

// Eden 区
Eden Space:  
	 // Eden 区总容量
   capacity = 214827008 (204.875MB)
   // Eden 区已用容量
   used     = 74442288 (70.99369812011719MB)
   // Eden 区剩余容量
   free     = 140384720 (133.8813018798828MB)
   // Eden 区已用比例
   34.65220164496263% used 

// From 区(survior1 区)
From Space: 
	 // From 区总容量
   capacity = 26804224 (25.5625MB) 
   // From 区已用容量
   used     = 3333984 (3.179534912109375MB) 
   // From 区剩余容量
   free     = 23470240 (22.382965087890625MB) 
   // From 区已用比例
   12.43827838477995% used 

// To 区(survior2 区)
To Space: 
   // To 区总容量
   capacity = 26804224 (25.5625MB) 
   // To 区已用容量
   used     = 0 (0.0MB) 
   // To 区剩余容量
   free     = 26804224 (25.5625MB)
   // To 区已用比例
   0.0% used 

// 老年代
PS Old  Generation: 
	 // 老年代容量
   capacity = 1879048192 (1792.0MB) 
   // 老年代已用容量
   used     = 30847928 (29.41887664794922MB) 
   // 老年代剩余容量
   free     = 1848200264 (1762.5811233520508MB) 
   // 老年代已用比例
   1.6416783843721663% used 

// 永久代
Perm Generation: 
   // perm 区容量
   capacity = 134217728 (128.0MB) 
   // perm 区已用容量
   used     = 47303016 (45.111671447753906MB) 
   // perm 区剩余容量
   free     = 86914712 (82.8883285522461MB) 
   // perm 区已用比例
   35.24349331855774% used 
```

新生代的内存回收就是采用空间换时间的方式；

如果from区使用率一直是 100% 说明程序创建大量的短生命周期的实例，使用jstat统计一下jvm在内存回收中发生的频率耗时以及是否有full gc，使用这个数据来评估一内存配置参数、gc参数是否合理；

#### 2.3.2 jmap 示例2

使用 `jmap -histo[:live] <pid>` 查看堆内存中的对象数目、大小统计直方图，如果带上live则只统计活对象，如下：

```javascript
root@ubuntu:/# jmap -histo:live 21711 | more 
   num        #instances   #bytes  class name
---------------------------------------
   1:         38445        5597736  <constMethodKlass>
   2:         38445        5237288  <methodKlass>
   3:          3500        3749504  <constantPoolKlass>
   4:         60858        3242600  <symbolKlass>
   5:          3500        2715264  <instanceKlassKlass>
   6:          2796        2131424  <constantPoolCacheKlass>
   7:          5543        1317400  [I
   8:         13714        1010768  [C
   9:          4752        1003344  [B
  10:          1225         639656  <methodDataKlass>
  11:         14194         454208  java.lang.String
  12:          3809         396136  java.lang.Class
  13:          4979         311952  [S
  14:          5598         287064  [[I
  15:          3028         266464  java.lang.reflect.Method
  16:           280         163520  <objArrayKlassKlass>
  17:          4355         139360  java.util.HashMap$Entry
  18:          1869         138568  [Ljava.util.HashMap$Entry;
  19:          2443          97720  java.util.LinkedHashMap$Entry
  20:          2072          82880  java.lang.ref.SoftReference
  21:          1807          71528  [Ljava.lang.Object;
  22:          2206          70592  java.lang.ref.WeakReference
  23:           934          52304  java.util.LinkedHashMap
  24:           871          48776  java.beans.MethodDescriptor
  25:          1442          46144  java.util.concurrent.ConcurrentHashMap$HashEntry
  26:           804          38592  java.util.HashMap
  27:           948          37920  java.util.concurrent.ConcurrentHashMap$Segment
  28:          1621          35696  [Ljava.lang.Class;
  29:          1313          34880  [Ljava.lang.String;
  30:          1396          33504  java.util.LinkedList$Entry
  31:           462          33264  java.lang.reflect.Field
  32:          1024          32768  java.util.Hashtable$Entry
  33:           948          31440  [Ljava.util.concurrent.ConcurrentHashMap$HashEntry;
```

class name 详细可见 JNI 描述符

#### 2.3.3 jmap 示例3

还有一个很常用的情况是：用jmap把进程内存使用情况dump到文件中，再用jhat分析查看。

jmap进行dump命令格式如下：`jmap -dump:format=b,file=path/to/file <pid>`

示例:

```sh
root@ubuntu:/# jmap -dump:format=b,file=/tmp/dump.dat 21711    
Dumping heap to /tmp/dump.dat ...
Heap dump file created
```

dump出来的文件可以用MAT、VisualVM等工具查看，这里用jhat查看：

```sh
root@ubuntu:/# jhat -port 9998 /tmp/dump.dat
Reading from /tmp/dump.dat...
Dump file created Tue Jan 28 17:46:14 CST 2014Snapshot read, resolving...
Resolving 132207 objects...
Chasing references, expect 26 dots..........................
Eliminating duplicate references..........................
Snapshot resolved.
Started HTTP server on port 9998
Server is ready.
```

注意如果Dump文件太大，可能需要加上-J-Xmx512m这种参数指定最大堆内存，即`jhat -J-Xmx512m -port 9998 /tmp/dump.dat`。

然后就可以在浏览器中输入主机地址:9998查看了.

其中最后的 Other Queries 一项支持OQL (对象查询语言), 可以自己去摸索下。



### 2.4 jstat(JVM统计监测工具)

语法格式如下：

```javascript
Usage: jstat --help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]

Definitions:
  <option>      An option reported by the -options option
  
  <vmid>        VM标识符. vmid形式如下: <lvmid>[@<hostname>[:<port>]]
                <lvmid>是目标Java虚拟机的本地虚拟机标识符，通常是一个进程id。
                <hostname>是运行目标Java虚拟机的主机名。
                <port>是目标主机上rmiregistry的端口号。
                有关虚拟机标识符的更完整描述，请参阅jvm stat文档。
                
  <lines>       标题行之间的样本数。

  <interval>    采样间隔。允许以下形式:<n>["ms"|"s"]，
                其中<n>是一个整数，后缀指定单位为毫秒("ms")或秒("s")。
                默认单位为“ms”。
                
  <count>       终止前取样数量。

  -J<flag>      Pass <flag> directly to the runtime system.
  -? -h --help  Prints this help message.
  -help         Prints this help message.
```

#### 2.4.1 jstat 示例:

vmid是Java虚拟机ID，在Linux/Unix系统上一般就是进程ID。interval是采样时间间隔。count是采样数目。比如下面输出的是GC信息，采样时间间隔为250ms，采样数为4：

```sh
root@ubuntu:/# jstat -gc 21711 250 4 
S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT  
192.0  192.0   64.0   0.0    6144.0   1854.9   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649192.0  
192.0   64.0   0.0    6144.0   1972.2   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649192.0  
192.0   64.0   0.0    6144.0   1972.2   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649192.0  
192.0   64.0   0.0    6144.0   2109.7   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649
```

要明白上面各列的意义，先看JVM堆内存布局：

可以看出：

```javascript
堆内存 = 年轻代 + 年老代 + 永久代
年轻代 = Eden区 + 两个Survivor区（From和To）
```

其对应的指标含义如下：

| 参数 | 描述                                                  |
| ---- | ----------------------------------------------------- |
| S0C  | 年轻代中第一个survivor（幸存区）的容量 (字节)         |
| S1C  | 年轻代中第二个survivor（幸存区）的容量 (字节)         |
| S0U  | 年轻代中第一个survivor（幸存区）目前已使用空间 (字节) |
| S1U  | 年轻代中第二个survivor（幸存区）目前已使用空间 (字节) |
| EC   | 年轻代中Eden（伊甸园）的容量 (字节)                   |
| EU   | 年轻代中Eden（伊甸园）目前已使用空间 (字节)           |
| OC   | Old(老年代)的容量 (字节)                              |
| OU   | Old(老年代)目前已使用空间 (字节)                      |
| PC   | Perm(持久代)的容量 (字节)                             |
| PU   | Perm(持久代)目前已使用空间 (字节)                     |
| YGC  | 从应用程序启动到采样时年轻代中gc次数                  |
| YGCT | 从应用程序启动到采样时年轻代中gc所用时间(s)           |
| FGC  | 从应用程序启动到采样时old代(全gc)gc次数               |
| FGCT | 从应用程序启动到采样时old代(全gc)gc所用时间(s)        |
| GCT  | 从应用程序启动到采样时gc用的总时间(s)                 |

### 2.5 hprof(Heap/CPU分析工具)

hprof能够展现CPU使用率，统计堆内存使用情况。

语法格式如下：

```javascript
java -agentlib:hprof[=options] ToBeProfiledClassjava -Xrunprof[:options] ToBeProfiledClassjavac -J-agentlib:hprof[=options] ToBeProfiledClass
```

完整的命令选项如下：

```javascript
Option Name and Value  Description                    Default
---------------------  -----------                    -------
heap=dump|sites|all    heap profiling                 all
cpu=samples|times|old  CPU usage                      off
monitor=y|n            monitor contention             n
format=a|b             text(txt) or binary output     a
file=<file>            write data to file             java.hprof[.txt]
net=<host>:<port>      send data over a socket        off
depth=<size>           stack trace depth              4
interval=<ms>          sample interval in ms          10
cutoff=<value>         output cutoff point            0.0001
lineno=y|n             line number in traces?         y
thread=y|n             thread in traces?              n
doe=y|n                dump on exit?                  y
msa=y|n                Solaris micro state accounting n
force=y|n              force output to <file>         y
verbose=y|n            print messages about dumps     y
```

来几个官方指南上的实例。

CPU Usage Sampling Profiling(cpu=samples)的例子：

```javascript
java -agentlib:hprof=cpu=samples,interval=20,depth=3 Hello
```

上面每隔20毫秒采样CPU消耗信息，堆栈深度为3，生成的profile文件名称是java.hprof.txt，在当前目录。 

CPU Usage Times Profiling(cpu=times)的例子，它相对于CPU Usage Sampling Profile能够获得更加细粒度的CPU消耗信息，能够细到每个方法调用的开始和结束，它的实现使用了字节码注入技术（BCI）：

```javascript
javac -J-agentlib:hprof=cpu=times Hello.java
```

Heap Allocation Profiling(heap=sites)的例子：

```javascript
javac -J-agentlib:hprof=heap=sites Hello.java
```

Heap Dump(heap=dump)的例子，它比上面的Heap Allocation Profiling能生成更详细的Heap Dump信息：

```javascript
javac -J-agentlib:hprof=heap=dump Hello.java
```

虽然在JVM启动参数中加入-Xrunprof:heap=sites参数可以生成CPU/Heap Profile文件，但对JVM性能影响非常大，不建议在线上服务器环境使用。

