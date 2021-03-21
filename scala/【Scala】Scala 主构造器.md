# 【Scala】Scala 主构造器参数可见性

刚刚接触 Scala 的时候, 被 Scala 五花八门的构造器中参数定义搞得七荤八素, 你可能见过以下几种定义方式:

```scala
class Dude(name: String)
class Dude(var name: String)
class Dude(val name: String)
class Dude(private val name: String)
class Dude(@BeanProperty val name: String)
```

对于习惯了 Java 四平八稳的构造方式,  Scala 真的快把初学者逼疯了, 下面我们就来分析这几种构造方式.

## 1. 主构造中形参没有 var/val 修饰

这种情形对应的就是第一行代码

```scala
class Dude(name: String)
```

Ok,  首先我们先看下反编译后的 Java 代码:

```scala
➜  scala scalac Dude.scala
➜  scala javap -private Dude
Compiled from "Dude.scala"
public class Dude {
  public Dude(java.lang.String);
}
```

 上面代码可以发现, 这种情况下, 外部无法访问该字段, 即:

```scala
val dude = new Dude("jack")
dude.name // 这里是会报错的, 因为 name 仅仅是形参, 外部不可见
```



## 2. 主构造中形参有 var/val 修饰

这种情形对应的就是第二行代码

```scala
class Dude(var name: String)
```

我们再看下反编译后的 Java 代码:

```sh
➜  scala scalac Dude.scala
➜  scala javap -private Dude
Compiled from "Dude.scala"
public class Dude {
  private java.lang.String name;
  public java.lang.String name();
  public void name_$eq(java.lang.String);
  public Dude(java.lang.String);
}
```

我们可以发现, 多了一个名为 name 的 private class field, 并且还有了名为 name() 的

getter 方法和名为 name_$eq() 的 setter 方法, 如果我们再访问该字段

```scala
scala> val d = new Dude("mike")
d: Dude = Dude@642505c7

scala> d.name
res0: String = mike

scala> d.name = "jack"
d.name: String = jack

scala> d.name
res1: String = jack
```

这里有个有意思的地方, scala 中 getter/setter 方法默认都是直接使用字段名的方式来访问, 那么有人会问说好的 name_$eq 方法呢? 这个方法在 Java 代码中访问 scala class 可以看到.

同理可证:

```scala
class Dude(val name:String)
```

这个肯定就会生成一个 `private final String name`字段以及 getter 方法, 有兴趣的自己去试下.

## 3. 主构造中有 private 修饰形参

这种情形对应的就是第三行代码

```scala
class Dude(private val name: String)
```

我们还是看下反编译后的 Java 代码:

```sh
➜  scala scalac Dude.scala
➜  scala javap -private Dude
Compiled from "Dude.scala"
public class Dude {
  private final java.lang.String name;
  private java.lang.String name();
  public Dude(java.lang.String);
}
```

这里我们可以看到 getter 方法的访问权限变成了 private, 这样的话对外也是不可访问的, 这里剧透下, 这种情况在 当前class 和伴生对象中都是可以访问, 其他地方都无法访问该字段.

```scala
class Dude(private val name: String)

object Dude {
	def main(args: Array[String]): Unit = {
		val dude = new Dude("jack")
    // 伴生对象中可访问
		println(dude.name)	
	}
}

object Test {
	def main(args: Array[String]): Unit = {	
		val dude = new Dude("jack")
    // 编译期这里就会报错, name is unaccessible
		println(dude.name)
	}
}
```

针对这种情况, 我们可以在 class 内提供一个外部访问成员字段 name 的方法鸭, 我们试试看

```scala
class Dude(private val name: String) {
	// 编译期这里又报错了,这是因为 scala 构造参数中参数名和 getter/setter 方法名相同了. 
	def name: String = this.name
}
```

于是我们把参数名改为 _name

```scala
class Dude(private val _name: String) {
	def name: String = this._name
}

object Dude {
	def main(args: Array[String]): Unit = {		
		val dude = new Dude("jack")
		println(dude.name)
		
	}
}

object Test {
	def main(args: Array[String]): Unit = {
		val dude = new Dude("jack")
		println(dude.name)
	}
}
```

终于运行测试通过啦

这里我们可以发现在主构造器内使用 private 修饰, 然后方法内自定义外部访问方法, 有点多此一举的感觉, 实际中如果要隐藏某字段外部不可见可以这么使用.

在 Scala中使用单例模式需要将主构造器私有化:

```scala
class Single private {
  // ......
}
object Single {
  
  lazy private val single = new Single
  def getInstance = single
}
```



## 4. 主构造器内有@BeanProperty 修饰

这种情形对应的就是第四行代码

```scala
class Dude(@BeanProperty var name: String)
```

> Tips: 要使用@BeanProperty 需要 import scala.beans.BeanProperty

我们还是看下代码反编译后的 Java 代码:

```scala
➜  scala scalac Dude.scala
➜  scala javap -private Dude
Compiled from "Dude.scala"
public class Dude {
  private java.lang.String name;
  public java.lang.String name();
  public void name_$eq(java.lang.String);
  public java.lang.String getName();
  public void setName(java.lang.String);
  public Dude(java.lang.String);
}
```

依旧是生成了 `private String name` 成员字段, 此外还生成了 scala 和 java 风格下各自的 getter/setter 方法:

```scala
// scala getter/setter
public java.lang.String name();
public void name_$eq(java.lang.String);

// java getter/setter
public java.lang.String getName();
public void setName(java.lang.String);
```

## 5. 总结

Scala 中主构造器中字段的可见性

| 可见性                  | 可访问 | 可修改 |
| ----------------------- | ------ | ------ |
| var                     | 是     | 是     |
| val                     | 是     | 否     |
| 缺省可见性 (非 val/var) | 否     | 否     |
| private 修饰            | 否     | 否     |

当没有可见性的时候可以自己手动添加可见性方法, 但方法名不可以和参数名相同

