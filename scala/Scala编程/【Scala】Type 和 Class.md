# 【Scala】type 和 class

由于在scala中非常强调泛型或者说类型系统 ，Java 和 scala 是基于jvm的，java1.5 以前具体对象的类型与class一一对应，后来引入泛型，例如***字符串数组***或***整数数组***都是数组 ，但其实类型是不一样的，在虚拟机内部，并不关心泛型或类型系统。对泛型支持是基于运行时角度考虑的，在虚拟机中泛型被编译运行时是被擦除的， 在运行时泛型是通过反射方式获取。

## 一、type 与 class 概念的区别

scala 中 type 是一种更具体的类型, 任何数据都有类型 。 

class 其实是 一种数据结构和基于该数据结构的一种抽象。

**在没有泛型之前，类型系统不存在高阶概念，直接与类一一映射，而泛型出现之后，就不在一一映射了**

比如定义`class List[T] {}`, 可以有 `List[Int]` 和 `List[String]`等具体类型，它们的类是同一个 `List`，但类型则根据不同的构造参数类型而不同。

类型一致的对象, 它们的类也是一致的，反过来，类一致的，其类型不一定一致.

> 通俗的说, type 一致则 class 一定一致, 而 class 一致 type 却不一定一致
>
> 这和 hashcode 与 equals 有点像, equals 相等的两个对象,其 hashcode 一定相等, 而相反 hashcode 相等的两个对象, 其 equals 不一定相对

```scala
scala> classOf[List[Int]] == classOf[List[String]]
res1: Boolean = true

scala> import scala.reflect.runtime.universe._
import scala.reflect.runtime.universe._

scala> typeOf[List[Int]] == typeOf[List[String]]
res5: Boolean = false
```

## 二、typeOf与 classOf 的区别

```scala
package com.ifly.edu.scala.BestPractice
 
/**
 *  calss 和 type  区别
 */
import scala.reflect.runtime.universe._
 
class Spark
 
trait Hadoop
 
object Flink
 
class Java {
 
  class Scala
 
}

```

### 1. class(类)

```scala
println(typeOf[Spark])
println(classOf[Spark])
```

运行结果

```
com.ifly.edu.scala.BestPractice.Spark 
class com.ifly.edu.scala.BestPractice.Spark
```



### 2. trait(特质)

```scala
println(typeOf[Hadoop])
println(classOf[Hadoop])
```

运行结果 

```
com.ifly.edu.scala.BestPractice.Hadoop 
interface com.ifly.edu.scala.BestPractice.Hadoop
```



### 3. object(对象)

```scala
println(typeOf[Flink])   //静态对象是没有 typeof
println(classOf[Flink])  //静态对象是没有 classof
```

 

### 4. 嵌套类

```scala
val java1 = new Java
val java2 = new Java
val scala1 = new java1.Scala
val scala2 = new java2.Scala

println(typeOf[java1.Scala])
println(classOf[java1.Scala])
```

运行结果 

```
java1.Scala 
class com.ifly.edu.scala.BestPractice.Java$Scala
```

## 三、classOf和getClass区别

`classOf` 获取运行时的类型。`classOf[T]` 相当于 java 中的 `T.class`

```scala
scala> class A
defined class A

scala> val a = new A
a: A = A@4c79ca55

scala> a.getClass
res6: Class[_ <: A] = class A

scala> classOf[A]
res7: Class[A] = class A
```

上面显示了两者的不同，`getClass` 方法得到的是 `Class[A]` 的某个子类，而 `classOf[A]` 得到是正确的 `Class[A]`，但是去比较的话，这两个类型是equals 为true的

```scala
scala> a.getClass == classOf[A]
res8: Boolean = true
```

这种细微的差别，体现在类型赋值时，因为 java 里的 `Class[T]`是不支持协变的，所以无法把一个 `Class[_ < : A]` 赋值给一个 `Class[A]`

```scala
scala> val clazz: Class[A] = a.getClass
<console>:16: error: type mismatch;
 found   : Class[T] where type T <: A
 required: Class[A]
Note: T <: A, but Java-defined class Class is invariant in type T.
You may wish to investigate a wildcard type such as `_ <: A`. (SLS 3.2.10)
       val clazz: Class[A] = a.getClass
                               ^
```

