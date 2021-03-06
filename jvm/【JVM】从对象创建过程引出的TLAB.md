## 【JVM】从对象创建过程引出的TLAB

### 对象的创建

例如: `User u = new User()`

1. 查看指令参数***(User)***是否能定位到常量池中的某个符号引用
2. 检查符号引用所代表的类***(com.github.java.User)***是否已经被加载, 解析和初始化过
3. 为对象分配内存空间
   - 如果内存规整(将已经分配内存放在一起,后面都是未分配的内存, 中间用一个指针来区分), 分配对象内存仅仅就是将指针向后移动的距离等于对象的大小, 这种分配方式称为***"指针碰撞"***
   - 如果内存不规整(已分配内存和未分配的内存完全杂乱无章), 这个时候JVM需要维护一张表来记录哪些内存使用哪些未使用, 分配的时候从表中找到一块足够大且未使用的内存分配给对象, 并更新表上记录, 这种分配方式称为***"空闲列表"***
   - 但是具体使用哪种分配方式由内存是否规整决定, 而内存是否规整又由JVM所使用的GC来决定的, 比如ParNew, Serial都是带Compact过程的垃圾收集器, 系统分配时采用指针碰撞, 而CMS这种垃圾收集器基于Mark-Sweep算法的收集器, 分配时通常采用空闲列表

> Tips:类完成加载后, 它所需内存大小就是可以确定的
>

### TLAB

在阅读<<深入理解JVM虚拟机>>过程中看到一句话;

> 除了如何划分内存的可用空间外, 还有一个需要考虑的问题是对象创建在虚拟机中是非常频繁的行为, 仅仅是修改一个指针所指向的位置, 在并发的情况下也并不是线程安全的, 可能出现正在给对象A分配内存, 指针还没来得及修改, ***<u>对象B又同时使用了原来的指针所分配的内存的情况.</u>***

开始看书并未理解这句话***"对象B又同时使用了原来的指针所分配的内存的情况"***

思考很久后, 认为这句话的意思是这样:

> **通常来说，当我们调用 new 指令时，它会在 Eden 区中划出一块作为存储对象的内存。由于堆空间 是线程共享的，因此直接在这里边划空间是需要进行同步的。**
>
> 否则，将有可能出现两个对象共用一段内存的事故。

举个简单的例子:

> 如果将一块块内存空间看做一个个"停车位", 这里就相当于两个司机（线程）同时将车停入同一个停车位，因而发生剐蹭事故。

Java 虚拟机的解决方法是为每个***司机(线程)***预先申请***多个停车位(一块内存缓冲区)***，并且只允许该司机停在自己的停车位 上。那么当司机的停车位用完了该怎么办呢（假设这个司机代客泊车）？

> **答案是**：再申请多个停车位便可以了。这项技术被称之为 **TLAB**（Thread Local Allocation Buffer， 对应虚拟机参数 `-XX:+UseTLAB`，默认开启）本地线程分配缓冲。

具体来说，每个线程可以向 Java 虚拟机申请一段连续的内存，比如 2048 字节，作为线程私有的 **TLAB**。

这个操作需要加锁，线程需要维护两个指针（实际上可能更多，但重要也就两个），一个指向 **TLAB** 中空余内存的起始位置，一个则指向 **TLAB** 末尾。

接下来的 **new** 指令，便可以直接通过**指针加法**（bump the pointer）来实现，即把指向空余内存位 置的指针加上所请求的字节数。

如果加法后空余内存指针的值仍小于或等于指向末尾的指针，则代表分配成功。否则，**TLAB** 已经没有足够的空间来满足本次新建操作。这个时候，便需要当前线程重新申请新的 **TLAB**。

