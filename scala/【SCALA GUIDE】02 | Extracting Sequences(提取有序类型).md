## 【SCALA GUIDE】02 | Extracting Sequences(提取有序类型)

### overview

在 [第一篇](/Users/sherlock/Desktop/notes/scala/[SCALA GUIDE]01 | 提取器(Extractors).md) 中，我们知道了如何实现自己的提取器，如何在模式匹配中使用提取器。然而，我们仅仅讨论了从数据中提取固定数量参数的提取器，然而Scala可以针对一些序列数据类型提取任意数量的参数。

例如，你可以定义模式来匹配一个只包含两个元素或三个元素的list：

```java
val xs = 3 :: 6 :: 12 :: Nil
xs match {
  case List(a, b) => a * b
  case List(a, b, c) => a + b + c
  case _ => 0
}
```

另外，如果你不在乎要匹配的list的具体长度，还可以使用通配操作符：_*：

```java
val xs = 3 :: 6 :: 12 :: 24 :: Nil
xs match {
  case List(a, b, _*) => a * b
  case _ => 0
}
```

上面这段代码中，第一个case被匹配到，xs的前两个元素分别被赋予a和b变量，同时不管列表中还剩余多少成员都忽略掉。

显然，这种用途的提取器不能用我在首篇中介绍的方法来实现。我们需要一种方式可以来定义一个提取器，它需要指定类型的参数输入并解构参数的数据到一个序列中，而这个序列的长度在编译时是未知的。

现在轮到unapplySeq出场了，这个方法就是用来实现上述场景的。我们来看下这方法的一种形式：

```java
def unapplySeq(object: S): Option[Seq[T]]
```

它传入类型为S的参数，如果S完全不匹配则返回None，否则返回包含在Option中的类型为T的Seq。



### 例子：提取名字

我们来实现一个看上去不太实用的例子（译者注：这个例子只是为了说明如何实现unapplySeq，可以有更简单的方法实际想要的功能）。假设在我们的程序中，我接受字串形式保存的人名。

如果一个人有多个名字，字串里可能包含他的第二或第三名字。因此，可能存在的值会是`"Daniel"、` `"Catherina Johanna"或者` `"Matthew John Michael"。我们想要匹配这些名字，并且提取出每个名字并赋给变量。`

下面是用unapplySeq来实现的一个简单的提取器：

```java
object GivenNames {
  def unapplySeq(name: String): Option[Seq[String]] = {
    val names = name.trim.split(" ")
    if (names.forall(_.isEmpty)) None else Some(names)
  }
}
```

给定一个包含一个或多个名字的字串，会将这些名字分解提取成一个序列。如果给定的姓名连一个名字都不包含，提取器将返回None，因而，使用该提取器的模式将不被匹配。

现在来测试下这个提取器：

```java
def greetWithFirstName(name: String) = name match {
  case GivenNames(firstName, _*) => "Good morning, " + firstName + "!"
  case _ => "Welcome! Please make sure to fill in your name!"
}
```

这个简短的方法返回问候语，只提取第一个名字而忽略其它名字。`greetWithFirstName("Daniel")` 返回 `"Good morning, Daniel!"`, 而 `greetWithFirstName("Catherina Johanna")` 返回 `"Good morning, Catherina!"。`

