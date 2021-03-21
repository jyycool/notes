## 【FLINK】如何确定TM的启动数量

主要梳理一下-p, -yn 和实际Flink作业提交成功之后生成TaskManager的个数之间的关系。

yn参数官网解释： -yn,--yarncontainer <arg>   Number of YARN container to allocate (=Number of Task Managers)。

按照这个参数解释，那就是我们设置了job中有多少个TaskManagers，但是事实是不是这样的呢？？？？那看一下下面提交的job:

```bash
flink run \
-m yarn-cluster \
-ynm AliyunNginxStudy20190328 \  
-yn 3 \
-ys 3 \
-p 3 \
-yjm 2048m \
-ytm 8192m \
-c --classpath  /opt/jars/online_aliyun_ls_parse_nginx_test.jar \
--output ${elasticsearch} \
--ipDbPath /opt/lib/ \
--windowSize 10
```

下图是具体提交到yarn上，Flink最终TaskManagers和Task Slots的数量情况。

![flink_ui](/Users/sherlock/Desktop/notes/allPics/Flink/flink_ui.png)

通过上面图的反应的情况，证明-yn并不能决定TaskManager的数量。其实在flink-1.7版本提交任务的时候就可以通过日志信息发现这个参数是弃用的。flink-1.6日志虽然没有提醒，但该参数也是处于废弃状态。

![yn](/Users/sherlock/Desktop/notes/allPics/Flink/yn.png)

说到底还是确定不了TaskManager最终的数量谁来决定的，通过亲自测试得到图9的结果，测试中flink配置文件中的默认并行度是1(parallelism.default: 1)，代码中没有单独设置operators的并行度。
 yn(实际) = Math.ceil(p/ys)
 ys(总共) = yn(实际)  * ys(指定)
 ys(使用) = p(指定)

![p:ys](/Users/sherlock/Desktop/notes/allPics/Flink/p:ys.png)

为了验证上面的说法，咱们继续提交一个job来去测试。如果网友有时间可以实际操作一下。

```bash
flink run \
-m yarn-cluster \
-ynm AliyunNginxStudy20190328 \  
-yn 3 \
-ys 2 \
-p 3 \
-yjm 2048m \
-ytm 8192m \
-c --classpath  /opt/jars/online_aliyun_ls_parse_nginx_test.jar \
--output ${elasticsearch} \
--ipDbPath /opt/lib/ \
--windowSize 10
```

下图可以清楚的看到，实际的TaskManagers数和Task Slots数验证了上面所说的。

![flink_ui2](/Users/sherlock/Desktop/notes/allPics/Flink/flink_ui2.png)

