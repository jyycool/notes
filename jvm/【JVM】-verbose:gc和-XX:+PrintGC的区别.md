## 【JVM】-verbose:gc和-XX:+PrintGC的区别

jvm调优，参数 ***-verbose:gc*** 和 ***-XX:+PrintGC*** 有什么具体的区别？还是说效果一样的，打印下了没发现什么差别。

1. 参数1：***-XX:+PrintGC***

   ```sh
   -XX:+PrintGC
   -XX:+PrintGCDetails
   -XX:+PrintGCDateStamps
   -Xloggc:C:\Users\ligj\Downloads\gc.log
   Java HotSpot(TM) 64-Bit Server VM (24.80-b11) for windows-amd64 JRE (1.7.0_80-b15), built on Apr 10 2015 11:26:34 by "java_re" with unknown MS VC++:1600
   Memory: 4k page, physical 4184440k(823480k free), swap 8367040k(3693052k free)
   CommandLine flags: -XX:InitialHeapSize=805306368 -XX:MaxHeapSize=805306368 -XX:MaxNewSize=536870912 -XX:NewSize=536870912 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC 
   2016-01-22T01:42:26.325+0800: 8.799: [GC [PSYoungGen: 393216K->60289K(458752K)] 393216K->60361K(720896K), 0.1290115 secs] [Times: user=0.17 sys=0.00, real=0.13 secs] 
   2016-01-22T01:42:35.849+0800: 18.322: [GC [PSYoungGen: 453505K->45971K(458752K)] 453577K->46043K(720896K), 0.1065373 secs] [Times: user=0.17 sys=0.00, real=0.11 secs] 
   2016-01-22T01:42:48.524+0800: 30.999: [GC [PSYoungGen: 439187K->52498K(458752K)] 439259K->52578K(720896K), 0.1795032 secs] [Times: user=0.20 sys=0.00, real=0.18 secs] 
   ```

   

2. 参数2：***-verbose:gc***

   ```sh
   -verbose:gc
   -XX:+PrintGCDetails
   -XX:+PrintGCDateStamps
   -Xloggc:C:\Users\ligj\Downloads\gc.log
   Java HotSpot(TM) 64-Bit Server VM (24.80-b11) for windows-amd64 JRE (1.7.0_80-b15), built on Apr 10 2015 11:26:34 by "java_re" with unknown MS VC++:1600
   Memory: 4k page, physical 4184440k(958020k free), swap 8367040k(3884064k free)
   CommandLine flags: -XX:InitialHeapSize=805306368 -XX:MaxHeapSize=805306368 -XX:MaxNewSize=536870912 -XX:NewSize=536870912 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC 
   2016-01-22T01:49:18.778+0800: 7.315: [GC [PSYoungGen: 393216K->60678K(458752K)] 393216K->60686K(720896K), 0.1241514 secs] [Times: user=0.13 sys=0.05, real=0.12 secs] 
   2016-01-22T01:49:27.794+0800: 16.332: [GC [PSYoungGen: 453894K->45567K(458752K)] 453902K->45583K(720896K), 0.1833980 secs] [Times: user=0.22 sys=0.00, real=0.18 secs] 
   2016-01-22T01:49:39.641+0800: 28.179: [GC [PSYoungGen: 438783K->53967K(458752K)] 438799K->53983K(720896K), 0.1683523 secs] [Times: user=0.31 sys=0.00, real=0.17 secs] 
   ```

   