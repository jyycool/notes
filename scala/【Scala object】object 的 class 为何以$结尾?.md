# 【Scala object】object 的 class 为何以$结尾?

看一段示例代码:

```scala
object DollarExample {
  def main(args : Array[String]) : Unit = {
    printClass()
  }

  def printClass() {
    println(s"The class is ${getClass}")
    println(s"The class name is ${getClass.getName}")
  }
}
```

运行结果

```sh
The class is class com.me.myorg.example.DollarExample$
The class name is com.me.myorg.example.DollarExample$
```

为什么这里的 className 会以$结尾?

## Scala object

Scala 中没有静态的概念, 所以它使用了 object 来实现了单例, 而单例中的变量和方法, 语义效果上和静态确实是一样的。

```sh
# sherlock @ mbp in ~ [16:45:23]
$ scala
Welcome to Scala 2.12.11 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_271).
Type in expressions for evaluation. Or try :help.

scala> :paste -raw
// Entering paste mode (ctrl-D to finish)

object HelloWorld {
 def main(args: Array[String]): Unit = {
    print("Hello World")
  }
}

// Exiting paste mode, now interpreting.

scala> :javap -p -filter HelloWorld
Compiled from "<pastie>"
public final class HelloWorld {
  public static void main(java.lang.String[]);
}

scala> :javap -p -filter HelloWorld$
Compiled from "<pastie>"
public final class HelloWorld$ {
  public static HelloWorld$ MODULE$;
  public static {};
  public void main(java.lang.String[]);
  private HelloWorld$();
}
```

使用反编译, 我们可以看到 object HelloWorld, 被生成了两个 class 文件, 分别是 HelloWorld 和 HelloWorld$。

上面使用 javap 查看的反编译代码还是不够清除, 我们使用 jd-gui 工具查看反编译的 class 代码。可以看到 HelloWorld 被具体变编译了如下内容:

```java
public class HelloWorld {
  public static void main(String[] args) {
    HelloWorld$.MODULE$.main(args);
  }
}

final class HelloWorld${
  public static final HelloWorld$ MODULE$;
  static{
    MODULE$=new HelloWorld$();
  }
  public void main(String[] args) {
    System.out.println("HelloWorld");
  }
}
```

这样就很清楚的可以看出, Scala 实现单例的方式: 在 HelloWorld\$ 内部有一个变量 MODULE$, 并且在类加载初始化的时候就给这个变量 MODULE\$ 赋值, 然后将 HelloWorld\$ 的构造器私有化。这样就可以保证 HelloWorld\$ 的实例 MODULE\$ 一定是全局唯一的。

对于 HelloWorld\$ 中变量和方法的使用在 HelloWorld 类中具体表现为使用了 MODULE$ 来调用变量和方法。