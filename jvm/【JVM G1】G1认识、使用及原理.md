# 【JVM G1】G1认识、使用及原理

G1全称是Garbage First Garbage Collector，使用G1的目的是简化性能优化的复杂性。例如，G1的主要输入参数是初始化和最大Java堆大小、最大GC中断时间。

G1 GC由Young Generation和Old Generation组成。G1将Java堆空间分割成了若干个Region，即年轻代/老年代是一系列Region的集合，这就意味着在分配空间时不需要一个连续的内存区间，即不需要在JVM启动时决定哪些Region属于老年代，哪些属于年轻代。因为随着时间推移，年轻代Region被回收后，又会变为可用状态（后面会说到的Unused Region或Available Region）了。