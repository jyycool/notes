## 【SCALA GUIDE】01 | 提取器(Extractors)

[原文出处](http://danielwestheide.com/blog/2013/01/09/the-neophytes-guide-to-scala-part-8-welcome-to-the-future.html) 翻译: Google

### Overview

有超过5万位学员（译者注：现在已经大大超过这个数了）报名参加了 Martin Odersky’s的 “[Functional Programming Principles in Scala](https://www.coursera.org/learn/progfun1)” 课程. 考虑到有不少开发者是第一次接触Scala或者函数式编程，这是一个很大的人数。

正在读此文章的你也许是其中一员吧，又或者你已经开始学习Scala。不管怎样，只要你开始学习了Scala，你就会因深入研究这门优雅的语言而兴奋，当然你可能还是会对Scala有点感觉有点陌生或者迷糊，那么以此篇开始的系列文章就是为你而准备的。

即使Coursera的课程已经覆盖了很多关于Scala的必要知识，但受限于时间，它无法将所有知识点展开太细。结果对于初学者来说，有些Scala的特性看上去就像是魔术。你仍然可以去用这些特性，但是并没有知道所以然：他们是如何工作的，为什么要设计成这样。

在这个系列文章中，我将为你解开谜底以消除你大脑中的问号。我还会就我个人学习scala时苦于找不到好的文章讲解而磕磕绊绊难以掌握的Scala特性和类库加以解释。某些场景下，我还会试着给你一些如何以符合语言习惯的使用这些特性的指点。

前言就这么多。在开始之前还请注意到参加Coursera课程并不是读这个系列文章的前提条件（译者注：当然，Scala的基本语法还是应该很熟悉的，至少要能看懂Scala代码，最好有实际开发经验），当然从该课程里获取的Scala知识对看懂文章还是有帮助的，我也会时不时的引用到课程。

### 一、神奇的模式匹配如何工作

在Coursera的课程中，你接触到了Scala的一个非常强大的特性：模式匹配。它让你可以分解给定的数据结构，绑定构造的数值到变量。这并不是Scala独有的，它在其它一些语言中也发挥着重要价值，如Haskell，Erlang。

在视频教程中你注意到模式匹配可以用来分解很多类型的数据结构，包含 `list`，`stream`和`case class`的任何实例。那么可以被分解的数据类型是固定数量的吗？换句话说，我们可以让自定义的数据类型也能被模型匹配吗？首先的问题应该是，它是如何工作的？你可以像下面的例子一样舒服的用模式匹配是为什么呢？

```scala
case class User(firstName: String, lastName: String, score: Int)
def advance(xs: List[User]) = xs match {
  case User(_, _, score1) :: User(_, _, score2) :: _ => score1 - score2
  case _ => 0
}
```

当然这里没啥神奇的魔术，至少没那么多魔法。 上面那样的代码（不要去在意这段代码的具体价值）之所以行得通是因为有个叫做提取器（Extractor）的东东存在。

在最常用的场景下，提取器是构造器的反操作：构造器根据参数列表构造一个对象实例，而提取器从一个现有对象实例中将构造时的参数提取出来。

Scala类库里自带了一些提取器，等下我们来看一些。Case class有一点特别，因为Scala为每个case class自动构建一个伙伴对象（companion object），伙伴对象是一个单例对象，同时包含构建伙伴类实例的apply方法以及一个unapply方法，一个对象想要成为一个提取器就需要实现此方法。

### 二、我的第一个提取器

一个合法的`unapply`方法可以有很多种形式，我们从最常用的形式开始讲起。假定我们的User（前述代码中）类不再是一个case class，而是一个trait，有两个类实现它，并且它只包含一个字段：

```SCALA
trait SingleValUser {
	def name: String
}
class FreeSingleValUser(val name: String) extends SingleValUser
class PremiumSingleValUser(val name: String) extends SingleValUser
```

我们想要在 FreeUser 和 PremiumUser 的伙伴对象中实现各自的提取器，类似Scala为case class做的一样。如果提取器仅从给定对象实例中提取单个属性，那么unapply的形式会是这样的：

```scala
def unapply(object: S): Option[T]
```

这方法需要一个类型S的对象参数，返回一个T类型Option，T是要提取的参数的类型。Option是Scala里对null值的安全表达方式（译者注：也更优雅），后面会有专门篇幅讲解，目前你只需要知道这个unapply方法返回为Some[T]（如果能够成功从对象中提取出参数）或者None，None表示参数不能够被提取，由提取器的实现来制定是否能被提取的规则。

下面是提取器的实现：

```scala
trait SingleValUser {
	def name: String
}
class FreeSingleValUser(val name: String) extends SingleValUser
class PremiumSingleValUser(val name: String) extends SingleValUser

object FreeSingleValUser {
	def unapply(user: FreeSingleValUser) = Some(user.name)
}
object PremiumSingleValUser {
	def unapply(user: PremiumSingleValUser) = Some(user.name)
}
```

我们可以在REPL里测试下：

```scala
scala> FreeUser.unapply(new FreeUser("Daniel"))
res0: Option[String] = Some(Daniel) 
```

当然通常我们不直接呼叫 unapply 方法。 当用于提取模式场景时，Scala会帮我们呼叫提取器的unapply方法。 

如果unapply返回了Some[T]，意味着模式匹配了，提取出的值将被赋予模式中定义的变量。如果返回None，意味着模式不匹配，将会进入下一个case语句。

下面的方式将我们的提取器用于模式匹配：

```scala
object Test extends App {
	val user = new PremiumSingleValUser("miao")
	user match {
		case FreeSingleValUser(name) =>
			println (s"Hello ${name}")
		case PremiumSingleValUser(name) =>
			println (s"Welcome back, dear ${name}")
	}
}
```

你可能已经注意到，我们定义的两个提取器从不返回None。在这样的场景下倒不算是缺陷，因为如果你的对象是属于某个数据类型，你可以同时做类型检查和提取。就像上面的例子，因为user为PremiumUser类型，所以FreeUser的case子句将不会被匹配，将不会呼叫FreeUser的提取器，相应的第二个case将会匹配，user实例会传递给PremiumUser的伙伴对象（提取器）的unapply方法，该方法返回name的值。

我们会在本章节的后面看到不总是返回Some[T]类型的提取器。

### 三、提取多个值

假设我们想要进行匹配的类有多个属性：

```scala
trait MultiValUser {
	def name: String
	def score: Int
}
class FreeMultiValUser(val name: String, val score: Int, val upgradeProbability: Double) extends MultiValUser
class PremiumMultiValUser(val name: String, val score: Int) extends MultiValUser

```

如果提取器要能够解构给定的数据到多个参数，提取器的unapply方法应该是这样的类型：

```scala
def unapply(object: S): Option[(T1, ..., Tn)]
```

这方法需要一个类型为S的参数，返回一个TupleN的Option类型，N（译者注：从1到22）是提取的参数数量。
我们来修改下提取器以适用到修改后的类：

```scala
trait MultiValUser {
	def name: String
	def score: Int
}
class FreeMultiValUser(val name: String, val score: Int, val upgradeProbability: Double) extends MultiValUser
class PremiumMultiValUser(val name: String, val score: Int) extends MultiValUser

object FreeMultiValUser {
	def unapply(user: FreeMultiValUser) = {
		Some(user.name, user.score, user.upgradeProbability)
	}
}
object PremiumMultiValUser {
	def unapply(user: PremiumMultiValUser) = {
		Some(user.name, user.score)
	}
}
```

我们可以将这新的提取器用于模式匹配，就和之前的匹配例子差不多：

```scala
object TestMultiUser extends App {
	val user: FreeMultiValUser = new FreeMultiValUser("mi", 95, 0.9)
	user match {
		case FreeMultiValUser(name, _, upgradeProbability) =>
			if (upgradeProbability > 0.75) println(s"${name}, what can we do for you today?") else println(s"Hello ${name}")
		case PremiumMultiValUser(name, _) =>
			println(s"Welcome back, dear ${name}")
	}
}
```

### 四、Boolean型提取器

有时候你并不想要从匹配的数据中提取参数，你只想要做一个简单的boolean检查。在这样的场景下，第三种unapply的形式就可以帮到你了，这个unapply需要一个S类型的参数并返回一个Boolean型结果：

```scala
def unapply(object: S): Boolean
```

在模式匹配时，如果提取器返回true，则匹配到，否则继续下一个匹配测试。

在前面的例子中，我们在模式匹配时插入了一个判断逻辑来确定一个FreeUser是否有潜在的可能升级账号。我们接下来把这段逻辑放到提取器里去：

```scala
object premiumCandidate {
  def unapply(user: FreeUser): Boolean = user.upgradeProbability > 0.75
}
```

就像你看到的，提取器不必一定定义在类的伙伴对象中。下面的例子告诉你如何使用这个boolean型提取器：

```scala
val user: User = new FreeUser("Daniel", 2500, 0.8d)
user match {
  case freeUser @ premiumCandidate() => 			          				
  		initiateSpamProgram(freeUser)
  case _ => sendRegularNewsletter(user)
} 
```

在这个例子中，我们传递了空的参数给提取器，因为你不需要提取任何参数并赋值到变量。

这例子里看起来还有点奇怪：我假定虚构的initiateSpamProgram函数需要传入一个FreeUser实例作为参数因为我不想给付费用户（PremiumUser）发送广告。因为user是User类型的数据，在没有模式匹配时，我非得用难看的类型转换才能将user传递给initiateSpamProgram函数。

好在Scala的模式匹配支持将匹配的变量赋给一个新变量，新变量的类型和匹配到的提取器所期待的数据类型一致。通过@符号来赋值。我们的premiumCandidate期待一个FreeUser类型的实例作为参数，那么freeUser就会是一个满足匹配条件的FreeUser实例。

我个人不是很常用boolean型提取器，不过知道有这样的用法还是必须的，说不定你什么时候就会发现这种用法的好处。

### 五、中间操作符模式

如果你参加了Coursera的那个Scala课程，你应该还记得你可以和构造一个list或Stream类似的方式用连接操作符（***list用`::`，stream用`#::`***）来解构它们：

```scala
val xs = 58 #:: 43 #:: 93 #:: Stream.empty
xs match {
  case first #:: second #:: _ => first - second
  case _ => -1
}
```

你是否好奇为啥可以这么整呢。答案是我们目前所看到的提取模式匹配的语法有一个替代写法，Scala可以把提取器当成中间操作符。

所以，像e(p1,p2)的中，e为提取器，p1，p2是从给定参数中提取出的两个值，你可以写成p1 e p2。

所以，上面代码中，中间操作模式 ***head #:: tail*** 可以写成 ***#::(head, tail)***，***PremiumUser提取器也可以被这样使用name PremiumUser score***。当然我不推荐这样的写法。中间操作符模式的写法更多应该用于那些看起来像是操作符的提取器（译者注：即那些名称以符号表达的提取器），就像 List 和 Stream 的组合操作符一样(***::，#::***)，PremiumUser看上去显然不像是个操作符。


### 六、进一步了解Stream提取器

虽然在模式匹配中使用#::并没有太多特别之处，我还是想再仔细解读一下上面的模式匹配代码。这段代码也是通过检查传入的数据来决定是否返回None表示不匹配（译者注：而不是仅仅检查数据类型是否一致）的一个例子。

下面是摘录自Scala 2.9.2的#::提取器的源代码：

```scala
object #:: {
  def unapplyA: Option[(A, Stream[A])] =
    if (xs.isEmpty) None
    else Some((xs.head, xs.tail))
}
```


如果传入的Stream实例为空，就返回None。也就是说case head #:: tail不会匹配空的stream，否则返回一个Tuple2类型的数据（译者注：封装在Option中），Tuple2中的第一个元素是stream的首个元素值，Tuple2的第二个元素是stream的tail，stream的tail也是一个Stream。因而case head #:: tail 将会匹配一个有一个或多个元素的stream。如果stream只有一个元素，它的tail将返回一个空stream。

为了更好的理解这段代码，我们来把模式匹配部分从内嵌操作匹配改成另一种写法：

```scala
val xs = 58 #:: 43 #:: 93 #:: Stream.empty
xs match {
  case #::(first, #::(second, _)) => first - second
  case _ => -1
}
```

首先，xs被传入提取器进行匹配，提取器返回Some(xs.head,xs.tail)，结果是first被赋予58，xs的剩余部分被再次传给被包含的提取器进行匹配，这次提取器还是返回一个Tuple2包含xs.tail的首元素和xs.tail的tail，也就是second会被赋予Tuple2的第一个元素43，Tuple2的第二个元素被赋予通配符_，也即被丢弃。

### 七、使用提取器

那么既然你可以轻易的从case class中获得一些有用的提取器，在什么场景下仍然需要用到自定义提取器呢？

有人认为使用case class的模式匹配破坏了封装原则，它耦合了数据和数据表达实现，这种批评通常是源于面向对象的出发点。从Scala的函数式编程方面出发，你把case class当成一个只包含数据而没有任何行为的代数数据类型(ADTs)（译者注：对这个术语我比较陌生，原文是lgebraic data types (ADTs)）仍然不失为一个好主意。

通常，仅在你需要从你无法控制的数据类型中提取些东西或者你想从指定数据中进行额外的模式匹配时才有必要实现自定义的提取器。例如，提取器的一种常见用法是从一些字串中提取有意义的值。

> 留个回家作业吧：实现一个叫做URLExtractor的提取器并使用它，传入String，进行URL匹配。

