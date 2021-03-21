# 【Spark】那些躺过的坑



### 1. 启动 spark-shell

启动 spark-shell 时报错:

```sh
➜  spark-2.3.4 spark-shell
20/10/15 18:14:57 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
20/10/15 18:15:09 ERROR SparkContext: Error initializing SparkContext.
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.hdfs.server.namenode.SafeModeException): Cannot create file/logs/spark/local-1602756907934.inprogress. Name node is in safe mode.
The reported blocks 3 has reached the threshold 0.9990 of total blocks 3. The number of live datanodes 1 has reached the minimum number 0. In safe mode extension. Safe mode will be turned off automatically in 20 seconds.
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.checkNameNodeSafeMode(FSNamesystem.java:1335)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFileInt(FSNamesystem.java:2474)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFile(FSNamesystem.java:2363)
	at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.create(NameNodeRpcServer.java:624)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.create(ClientNamenodeProtocolServerSideTranslatorPB.java:398)
	at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:616)
	at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:982)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2217)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2213)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1762)
	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2211)

	at org.apache.hadoop.ipc.Client.call(Client.java:1475)
	at org.apache.hadoop.ipc.Client.call(Client.java:1412)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:229)
	at com.sun.proxy.$Proxy15.create(Unknown Source)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.create(ClientNamenodeProtocolTranslatorPB.java:296)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:191)
	at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:102)
	at com.sun.proxy.$Proxy16.create(Unknown Source)
	at org.apache.hadoop.hdfs.DFSOutputStream.newStreamForCreate(DFSOutputStream.java:1648)
	at org.apache.hadoop.hdfs.DFSClient.create(DFSClient.java:1689)
	at org.apache.hadoop.hdfs.DFSClient.create(DFSClient.java:1624)
	at org.apache.hadoop.hdfs.DistributedFileSystem$7.doCall(DistributedFileSystem.java:448)
	at org.apache.hadoop.hdfs.DistributedFileSystem$7.doCall(DistributedFileSystem.java:444)
	at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
	at org.apache.hadoop.hdfs.DistributedFileSystem.create(DistributedFileSystem.java:459)
	at org.apache.hadoop.hdfs.DistributedFileSystem.create(DistributedFileSystem.java:387)
	at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:911)
	at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:892)
	at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:789)
	at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:778)
	at org.apache.spark.scheduler.EventLoggingListener.start(EventLoggingListener.scala:120)
	at org.apache.spark.SparkContext.<init>(SparkContext.scala:522)
	at org.apache.spark.SparkContext$.getOrCreate(SparkContext.scala:2493)
	at org.apache.spark.sql.SparkSession$Builder$$anonfun$7.apply(SparkSession.scala:934)
	at org.apache.spark.sql.SparkSession$Builder$$anonfun$7.apply(SparkSession.scala:925)
	at scala.Option.getOrElse(Option.scala:121)
	at org.apache.spark.sql.SparkSession$Builder.getOrCreate(SparkSession.scala:925)
	at org.apache.spark.repl.Main$.createSparkSession(Main.scala:103)
	at $line3.$read$$iw$$iw.<init>(<console>:15)
	at $line3.$read$$iw.<init>(<console>:43)
	at $line3.$read.<init>(<console>:45)
	at $line3.$read$.<init>(<console>:49)
	at $line3.$read$.<clinit>(<console>)
	at $line3.$eval$.$print$lzycompute(<console>:7)
	at $line3.$eval$.$print(<console>:6)
	at $line3.$eval.$print(<console>)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at scala.tools.nsc.interpreter.IMain$ReadEvalPrint.call(IMain.scala:786)
	at scala.tools.nsc.interpreter.IMain$Request.loadAndRun(IMain.scala:1047)
	at scala.tools.nsc.interpreter.IMain$WrappedRequest$$anonfun$loadAndRunReq$1.apply(IMain.scala:638)
	at scala.tools.nsc.interpreter.IMain$WrappedRequest$$anonfun$loadAndRunReq$1.apply(IMain.scala:637)
	at scala.reflect.internal.util.ScalaClassLoader$class.asContext(ScalaClassLoader.scala:31)
	at scala.reflect.internal.util.AbstractFileClassLoader.asContext(AbstractFileClassLoader.scala:19)
	at scala.tools.nsc.interpreter.IMain$WrappedRequest.loadAndRunReq(IMain.scala:637)
	at scala.tools.nsc.interpreter.IMain.interpret(IMain.scala:569)
	at scala.tools.nsc.interpreter.IMain.interpret(IMain.scala:565)
	at scala.tools.nsc.interpreter.ILoop.interpretStartingWith(ILoop.scala:807)
	at scala.tools.nsc.interpreter.ILoop.command(ILoop.scala:681)
	at scala.tools.nsc.interpreter.ILoop.processLine(ILoop.scala:395)
	at org.apache.spark.repl.SparkILoop$$anonfun$initializeSpark$1$$anonfun$apply$mcV$sp$1$$anonfun$apply$mcV$sp$2.apply(SparkILoop.scala:79)
	at org.apache.spark.repl.SparkILoop$$anonfun$initializeSpark$1$$anonfun$apply$mcV$sp$1$$anonfun$apply$mcV$sp$2.apply(SparkILoop.scala:79)
	at scala.collection.immutable.List.foreach(List.scala:381)
	at org.apache.spark.repl.SparkILoop$$anonfun$initializeSpark$1$$anonfun$apply$mcV$sp$1.apply$mcV$sp(SparkILoop.scala:79)
	at org.apache.spark.repl.SparkILoop$$anonfun$initializeSpark$1$$anonfun$apply$mcV$sp$1.apply(SparkILoop.scala:79)
	at org.apache.spark.repl.SparkILoop$$anonfun$initializeSpark$1$$anonfun$apply$mcV$sp$1.apply(SparkILoop.scala:79)
	at scala.tools.nsc.interpreter.ILoop.savingReplayStack(ILoop.scala:91)
	at org.apache.spark.repl.SparkILoop$$anonfun$initializeSpark$1.apply$mcV$sp(SparkILoop.scala:78)
	at org.apache.spark.repl.SparkILoop$$anonfun$initializeSpark$1.apply(SparkILoop.scala:78)
	at org.apache.spark.repl.SparkILoop$$anonfun$initializeSpark$1.apply(SparkILoop.scala:78)
	at scala.tools.nsc.interpreter.IMain.beQuietDuring(IMain.scala:214)
	at org.apache.spark.repl.SparkILoop.initializeSpark(SparkILoop.scala:77)
	at org.apache.spark.repl.SparkILoop.loadFiles(SparkILoop.scala:110)
	at scala.tools.nsc.interpreter.ILoop$$anonfun$process$1.apply$mcZ$sp(ILoop.scala:920)
	at scala.tools.nsc.interpreter.ILoop$$anonfun$process$1.apply(ILoop.scala:909)
	at scala.tools.nsc.interpreter.ILoop$$anonfun$process$1.apply(ILoop.scala:909)
	at scala.reflect.internal.util.ScalaClassLoader$.savingContextLoader(ScalaClassLoader.scala:97)
	at scala.tools.nsc.interpreter.ILoop.process(ILoop.scala:909)
	at org.apache.spark.repl.Main$.doMain(Main.scala:76)
	at org.apache.spark.repl.Main$.main(Main.scala:56)
	at org.apache.spark.repl.Main.main(Main.scala)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.spark.deploy.JavaMainApplication.start(SparkApplication.scala:52)
	at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:890)
	at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:192)
	at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:217)
	at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:137)
	at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
org.apache.hadoop.ipc.RemoteException: Cannot create file/logs/spark/local-1602756907934.inprogress. Name node is in safe mode.
The reported blocks 3 has reached the threshold 0.9990 of total blocks 3. The number of live datanodes 1 has reached the minimum number 0. In safe mode extension. Safe mode will be turned off automatically in 20 seconds.
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.checkNameNodeSafeMode(FSNamesystem.java:1335)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFileInt(FSNamesystem.java:2474)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFile(FSNamesystem.java:2363)
	at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.create(NameNodeRpcServer.java:624)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.create(ClientNamenodeProtocolServerSideTranslatorPB.java:398)
	at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:616)
	at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:982)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2217)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2213)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1762)
	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2211)

  at org.apache.hadoop.ipc.Client.call(Client.java:1475)
  at org.apache.hadoop.ipc.Client.call(Client.java:1412)
  at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:229)
  at com.sun.proxy.$Proxy15.create(Unknown Source)
  at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.create(ClientNamenodeProtocolTranslatorPB.java:296)
  at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
  at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
  at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
  at java.lang.reflect.Method.invoke(Method.java:498)
  at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:191)
  at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:102)
  at com.sun.proxy.$Proxy16.create(Unknown Source)
  at org.apache.hadoop.hdfs.DFSOutputStream.newStreamForCreate(DFSOutputStream.java:1648)
  at org.apache.hadoop.hdfs.DFSClient.create(DFSClient.java:1689)
  at org.apache.hadoop.hdfs.DFSClient.create(DFSClient.java:1624)
  at org.apache.hadoop.hdfs.DistributedFileSystem$7.doCall(DistributedFileSystem.java:448)
  at org.apache.hadoop.hdfs.DistributedFileSystem$7.doCall(DistributedFileSystem.java:444)
  at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
  at org.apache.hadoop.hdfs.DistributedFileSystem.create(DistributedFileSystem.java:459)
  at org.apache.hadoop.hdfs.DistributedFileSystem.create(DistributedFileSystem.java:387)
  at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:911)
  at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:892)
  at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:789)
  at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:778)
  at org.apache.spark.scheduler.EventLoggingListener.start(EventLoggingListener.scala:120)
  at org.apache.spark.SparkContext.<init>(SparkContext.scala:522)
  at org.apache.spark.SparkContext$.getOrCreate(SparkContext.scala:2493)
  at org.apache.spark.sql.SparkSession$Builder$$anonfun$7.apply(SparkSession.scala:934)
  at org.apache.spark.sql.SparkSession$Builder$$anonfun$7.apply(SparkSession.scala:925)
  at scala.Option.getOrElse(Option.scala:121)
  at org.apache.spark.sql.SparkSession$Builder.getOrCreate(SparkSession.scala:925)
  at org.apache.spark.repl.Main$.createSparkSession(Main.scala:103)
  ... 55 elided
<console>:14: error: not found: value spark
       import spark.implicits._
              ^
<console>:14: error: not found: value spark
       import spark.sql
              ^
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.4
      /_/

Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_251)
Type in expressions to have them evaluated.
Type :help for more information.

scala> :quit
```

注意到这句话 ***Cannot create file/logs/spark/local-1602756907934.inprogress. Name node is in safe mode.*** 问题在于 namenode 的 safe mode

```sh
# 关闭 hadoop 的 safe mode, 之后重启 spark-shell, 问题解决
➜  hadoop-2.7.7 hadoop dfsadmin -safemode leave
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

20/10/15 18:17:10 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Safe mode is OFF

➜  spark-2.3.4 spark-shell
20/10/15 18:17:18 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://192.168.31.108:4040
Spark context available as 'sc' (master = local[*], app id = local-1602757047307).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.4
      /_/

Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_251)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

