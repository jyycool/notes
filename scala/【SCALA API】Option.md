## 【SCALA API】Option

Scala Option(选项)类型用来表示一个值是可选的（有值或无值)。

Option[T] 是一个类型为 T 的可选值的容器： 

- 如果值存在， Option[T] 就是一个 Some[T] 
- 如果不存在， Option[T] 就是对象 None 



接下来我们来看一段代码：

```scala
// 虽然 Scala 可以不定义变量的类型，不过为了清楚些，我还是
// 把他显示的定义上了
 
val myMap: Map[String, String] = Map("key1" -> "value")
val value1: Option[String] = myMap.get("key1")
val value2: Option[String] = myMap.get("key2")
 
println(value1) // Some("value1")
println(value2) // None
```

### Scala Option 常用方法

| 序号 | 方法及描述                                                   |
| :--- | :----------------------------------------------------------- |
| 1    | **def get: A**                                                                                                                                                       获取可选值 |
| 2    | **def isEmpty: Boolean**                                                                                                                                  检测可选类型值是否为 None，是的话返回 true，否则返回 false |
| 3    | **def productArity: Int**                                                                                                                                   返回元素个数， A(x_1, ..., x_k), 返回 k |
| 4    | **def productElement(n: Int): Any**                                                                                                               获取指定的可选项，以 0 为起始。即 A(x_1, ..., x_k), 返回 x_(n+1) ， 0 < n < k. |
| 5    | **def exists(p: (A) => Boolean): Boolean**                                                                                                    如果可选项中指定条件的元素存在且不为 None 返回 true，否则返回 false。 |
| 6    | **def filter(p: (A) => Boolean): Option[A]**                                                                                                  如果选项包含有值，而且传递给 filter 的条件函数返回 true， filter 会返回 Some 实例。 否则，返回值为 None 。 |
| 7    | **def filterNot(p: (A) => Boolean): Option[A]**                                                                                           如果选项包含有值，而且传递给 filter 的条件函数返回 false， filter 会返回 Some 实例。 否则，返回值为 None 。 |
| 8    | **def flatMap\[B](f: (A) => Option[B]): Option[B]**                                                                                     如果选项包含有值，则传递给函数 f 处理后返回，否则返回 None |
| 9    | **def foreach\[U](f: (A) => U): Unit**                                                                                                               如果选项包含有值，则将每个值传递给函数 f， 否则不处理。 |
| 10   | **def getOrElse\[B >: A](default: => B): B**                                                                                                   如果选项包含有值，返回选项值，否则返回设定的默认值。 |
| 11   | **def isDefined: Boolean**                                                                                                                                如果可选值是 Some 的实例返回 true，否则返回 false。 |
| 12   | **def iterator: Iterator[A]**                                                                                                                              如果选项包含有值，迭代出可选值。如果可选值为空则返回空迭代器。 |
| 13   | **def map\[B](f: (A) => B): Option[B]**                                                                                                          如果选项包含有值， 返回由函数 f 处理后的 Some，否则返回 None |
| 14   | **def orElse\[B >: A](alternative: => Option[B]): Option[B]**                                                                    如果一个 Option 是 None ， orElse 方法会返回传名参数的值，否则，就直接返回这个 Option。 |
| 15   | **def orNull**                                                                                                                                                       如果选项包含有值返回选项值，否则返回 null。 |