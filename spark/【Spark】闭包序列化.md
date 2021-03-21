## 【SPARK】闭包序列化

### scala闭包

具体详见 [scala闭包](/Users/sherlock/Desktop/notes/scala/[SCALA]闭包.md)



### 序列化

```scala
object Spark_closure_serial {

	def main(args: Array[String]): Unit = {

		val conf: SparkConf = new SparkConf()
				.setMaster("local[1]")
				.setAppName("serial")
		val sc: SparkContext = new SparkContext(conf)

		val intsRDD: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4))
		val num: Num = new Num
		intsRDD.foreach(i => println(s"total = ${num.figure + i}"))

		sc.stop()
	}
	
	class Num {
		val figure: Int = 10;
	}
}
```

上述代码在执行时会报错:

```scala
Exception in thread "main" org.apache.spark.SparkException: Task not serializable
	at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:403)
	at org.apache.spark.util.ClosureCleaner$.org$apache$spark$util$ClosureCleaner$$clean(ClosureCleaner.scala:393)
	at org.apache.spark.util.ClosureCleaner$.clean(ClosureCleaner.scala:162)
	at org.apache.spark.SparkContext.clean(SparkContext.scala:2326)
	at org.apache.spark.rdd.RDD$$anonfun$foreach$1.apply(RDD.scala:926)
	at org.apache.spark.rdd.RDD$$anonfun$foreach$1.apply(RDD.scala:925)
	at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:151)
	at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:112)
	at org.apache.spark.rdd.RDD.withScope(RDD.scala:363)
	at org.apache.spark.rdd.RDD.foreach(RDD.scala:925)
	at cgs.spark.core.rdd.operator.serial.Spark_closure_serial$.main(Spark_closure_serial.scala:22)
	at cgs.spark.core.rdd.operator.serial.Spark_closure_serial.main(Spark_closure_serial.scala)
Caused by: java.io.NotSerializableException: cgs.spark.core.rdd.operator.serial.Spark_closure_serial$Num
Serialization stack:
```

初步查看原因是因为没有序列化, task 内引用了外部不可序列化的变量, task 发送到 executor 端无法反序列化执行, 所以报错: 

将 ***class Num extends Serializable (或者使用 case class)*** , 之后可以正常执行, 

### 闭包

```scala
object Spark_closure_serial {

	def main(args: Array[String]): Unit = {

		val conf: SparkConf = new SparkConf()
				.setMaster("local[1]")
				.setAppName("serial")
		val sc: SparkContext = new SparkContext(conf)

		val intsRDD: RDD[Int] = sc.makeRDD(List())
		val num: Num = new Num
		intsRDD.foreach(i => println(s"total = ${num.figure + i}"))

		sc.stop()
	}

	class Num {
		val figure: Int = 10;
	}
}
```

上述代码中也会报异常 ***"Task not serializable"***,  但是 intRDD是空的 List 集合, foreach()方法根本不会执行到方法体内, 为何也会报错 ?

查看 foreach 源码

```scala
/**
* Applies a function f to all elements of this RDD.
*/
def foreach(f: T => Unit): Unit = withScope {
	val cleanF = sc.clean(f)
	sc.runJob(this, (iter: Iterator[T]) => iter.foreach(cleanF))
}
```

在提交 Job 前执行了 ***sc.clean(f)***, 这个方法在追踪堆栈信息后,它最后报错的地方是在:

```scala
private def ensureSerializable(func: AnyRef) {
    try {
      if (SparkEnv.get != null) {
        SparkEnv.get.closureSerializer.newInstance().serialize(func)
      }
    } catch {
      case ex: Exception => throw new SparkException("Task not serializable", ex)
    }
  }
```

所以在提交作业前, spark 会对算子内用户定义的方法做闭包清理和可序列化检查, 如果序列化检查不通过则会报 ***"Task not serializable"***

