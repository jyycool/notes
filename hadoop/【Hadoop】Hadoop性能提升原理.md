# 【Hadoop】Hadoop性能提升原理

最近在生产中碰到 Hadoop 的几个问题:

1. Hadoop 在超高的并发下会不工作的 bug 修复
2. NameNode 因为 fullGC 直接退出
3. 锁的性能不高
4. NameNode 写数据的优化

