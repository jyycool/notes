# 为什么CPU缓存对数组友好而对链表不友好?

去遍历相同的链表和数组 通过时间复杂度分析的话都是 O(n)。所以按道理是差不多的 但是在实践中， 这2者却有极大的差异。通过下面的分析你会发现，其实数组比链表要快很多。

我们首先要明白计算机中的存储层次结构.

## 存储层次结构

- CPU 寄存器 – immediate access (0-1个CPU时钟周期)
- CPU L1 缓存 – fast access (3个CPU时钟周期)
- CPU L2 缓存 – slightly slower access (10个CPU时钟周期)
- 内存 (RAM) – slow access (100个CPU时钟周期)
- 硬盘 (file system) – very slow (10,000,000个CPU时钟周期)

各级别的存储器速度差异非常大，CPU寄存器速度是内存速度的100倍. 这就是为什么CPU产商发明了CPU缓存。 而这个CPU缓存，就是数组和链表的区别的关键所在。

CPU 缓存行通常大小为 64 byte, 而缓存行在加载数据时会把一片连续的内存空间读入. 因为数组结构是连续的内存地址，所以数组全部或者部分元素被连续存在CPU缓存里面, 平均读取每个元素的时间只要3个CPU时钟周期。  

而链表的节点是分散在堆空间里面的，这时候CPU缓存帮不上忙，只能是去读取内存，平均读取时间需要100个CPU时钟周期。这样算下来，数组访问的速度比链表快33倍！(这里只是介绍概念，具体的数字因CPU而异)

但是这里也有个问题, 因为缓存行大小只有 64byte,  所以如果数组比较大, 只有小部分会被读取到缓存内, 如果在缓存中无法命中的话, 剩余部分还是要到内存中加载, 这样的话, 缓存对数据友好的优势便不存在了. 

