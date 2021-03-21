# 32|Future和并发编程

随着多核处理器的大量普及，人们对并发编程也越来越关注 。 Java 提供了围绕着共享内存和锁构建的并发支持。 虽然支持是完备的，这种并发方案在实 际过程中却很难做对 。Scala 标准类库提供了另一种能够规避这些难点的选择， 将程序员的精力集中在***不可变状态的异步变换上***： 也就是 Future 。

虽然 Java 也提供了 Future ，它跟 Scala 的 Future 非常不同 。

两种 Future 都是用来表示某个异步计算的结果 (它更像是一个异步计算结果的占位符)，但 Java 的 Future 要求通过阻塞 的 get 方法来访问这个结果 。 虽然可以在调用 get 之前先调用 isDone 来判断某个 Java 的 Future 是否已经完成，从而避免阻塞，你却必须等到 Java 的 Future 完成之后才能继续用这个结果做进一步的计算 。

Scala 的 Future 则不同 , 不论计算是否完成，你都可以指定对它的变换逻辑。 每一个变换都产生新的 Future 来表示原始的 Future 经过给定的函数 变换后产生的异步结果。 执行计算的线程由隐式给出的执行上下文（ ExecutionContext ）决定 。 ***这使得你可以将异步的计算描述成一系列的对不可变值的变换， 完全不需要考虑共享内存和锁***。

>所以 Scala 的 Future 本质是对不可变值的转换, 所以它就不需要考虑共享内存和锁机制这些问题.



## 32.1 天堂里的烦恼

在 Java 平台上，每个对象都关联了一个逻辑监视器 （ monitor ），可以用来控制对数据的多线程访问。 使用这种模型需要由你来决定哪些数据将被多个线程共享，并将访问共享数据或控制对这些共享数据访问的代码段标记为 “synchronized”。 Java 运行时将运用一种锁的机制来确保同一时间只有一个线程进入有同一个锁控制的同步代码段，从而让你可以协同共享数据的多线程访问 。

为了兼容， Scala 提供了对 Java 并发原语（ concurrency primitives ）的访问。 我们可以用 Scala 调用 wait 、 notify 和 notifyAll 等方法，它们的含义 跟 Java 中的方法一样。 从技术上讲 Scala 并没有 synchronized 关键字，不过它有一个 synchronized 方法，可以像这样来调用 ：

```scala
var counter = 0 
synchronized {
	//这里每次都只有一个线程在执行
	counter = counter + 1 
}
```

不幸的是，程序员们发现要使用共享数据和锁模型来有把握地构建健壮的、多线程的应用程序十分困难。 这当中的问题是，在程序中的每一点，你都必须推断出哪些你正在修改或访问的数据可能会被其他线程修改或访问，以及在这一点上你握有哪些锁。 每次方法调用，你都必须推断出它将会尝试握有哪些锁， 并说服自己它这样做不会死锁。而在你推断中的这些锁，并不是在编译期就固 定下来的，这让问题变得更加复杂，因为程序可以在运行时的执行过程中任意创建新的锁。 

更糟的是，对于多线程的代码而言，测试是不可靠的 。 由于线程是非确定性的，你可能测试 1000 次都是成功的，而程序第一次在客户的机器上运行就 出问题。 对共享数据和锁，你必须通过推断来把程序做对，别无他途。

不仅如此，你也无法通过过度的同步来解决问题。同步一切可能并不比什么都不同步更好。 这中间的问题是尽管新的锁操作去掉了争用状况的可能，但同时也增加了新的死锁的可能 。 一个正确的使用锁的程序既不能存在争用状况，也不能有死锁，因此你不论往哪个方向做过头都是不安全的 。

java.util.concurrent 类库提供了并发编程的更高级别的抽象。 使用并发工具包来进行多线程编程比你用低级别的同步语法制作自己的抽象可能会带来的问题要少得多。 尽管如此 ，并发工具包也是基于共享数据和锁的，因而并没有从根本上解决使用这种模型的种种困难。



## 32.2 异步执行和Try

虽然并非银弹（ silver bullet ), Scala 的 Future 提供了一种可以减少（甚至免去）对共享数据和所进行推理的方式。 当你调用 Scala 方法时，它 “在你等待的过程中”执行某项计算并返回结果。 如果结果是一个 Future ，那么这个 Future 就表示另一个将被异步执行的计算，而该计算通常是由另一个完全不同的线程来完成的。 因此 ，对 Future 的许多操作都需要一个隐式的执行上下文（ execution context ）来提供异步执行函数的策略。 例如，如果你试着通过 Future.apply 工厂方法创建一个 future 但又不提供隐式的执行上下文（ scala.concurrent.ExecutionContext 的实例），你将得到一个编译错误：

```scala
scala> import scala.concurrent.Future 
import scala.concurrent.Future

scala> val fut = Future {Thread.sleep(lOOOO); 21 + 21} <console>:ll: error: Cannot find an implicit ExecutionContext.
	You might pass an (implicit ec: ExecutionContext) parameter to your method or import scala.concurrent.ExecutionContext.Implicits.global.
		val fut= Fut ure { Thread.sleep(lOOOO); 21 + 21}

```

这个错误消息提示了解决该问题的一种方式 ： 引人 Scala 提供的一个全局的执行上下文。 ***对 JVM 而言，这个全局的执行上下文使用的是一个线程池***。一旦将隐式的执行上下文纳人到作用域，你就可以创建 future 了：

```scala
scala> import scala.concurrent.ExecutionContext.Implicits.global import scala.concurrent.ExecutionContext.Implicits.global

scala> val fut = Future {Thread.sleep(lOOOO); 21 + 21}
fut: scala.concurrent.Future[Int] = ...
```

前一例中创建的 future 利用 global 这个执行上下文异步地执行代码块， 然后计算出 42 这个值。 一旦开始执行，对应的线程会睡 10 秒钟。 因此这个 future 至少需要 10 秒才能完成。

Future 有两个方法可以让你轮询： ***isCompleted*** 和 ***value*** 。 对一个还未完成的 future 调用时， isCompleted 将返回 false ，而 value 将返回 None 。

```scala
scala> fut.isCompleted 
resO: Boolean = false

scala> fut.value 
resl: Option[scala. util. Try[Int]] = None
```

而一旦 future 完成（本例中意味着过了 10 秒钟）， isCompleted 将返回 true ，而 value 将返回一个 Some:

```scala
scala> fut.isCompleted 
res2: Boolean = true

scala> fut.value 
res3: Option[scala.util.Try[Int]] = Some(Success(42))
```

这里 value 返回的可选值包含一个 Try。 如图 32.1 所示， 一个 Try 要么是包含类型为 T 的值的 Success ，要么是包含一个异常（ java.lang.Throwable 的实例）的 Failure 。 Try 的目的是为异步计算提供一种与同步计算中 try 表达式类似的东西：允许你处理那些计算有可能异常终止而不是返回 结果的情况 。

> `scala.util.Try` 的结构与 `Either` 相似，`Try` 是一个 `sealed` 抽象类，具有两个子类，分别是 `Succuss` 和 `Failure`。前者的使用方式与 `Right` 类相似，它会保存正常的返回值 `Success[T]`。后者与 `Left` 相似，不过 `Failure` 总是保存 `Failure[Throwable]` 类型的值。

对同步计算而言，你可以用 try/catch 来确保调用某个方法的线程可以 捕获并处理由该方法抛出的异常。 不过对于异步计算来说，发起该计算的线程通常都转到别的任务去了。 在这之后如果异步计算因为某个异常失败了，原始的线程就不再能够用 catch 来处理这个异常。 因此，当处理表示异步活动的 Future 时，你要用 Try 来处理这种情况：该活动未能交出某个结果，而是异常终止了。 这里有一个展示了异步活动失败场景的例子 ：

```scala
scala> val fut = Future { Thread.sleep(lOOOO); 21 / 0}
fut:scala.concurrent.Future[Int] = .
scala> fut.value 
res4: Option [seal a. util. Try [Int]] = None

10 秒钟过后：

scala> fut.value 
res5: Option[scala.util.Try[Int]] = Some(Failure(java.lang.ArithmeticException: / by zero))
```



## 32.3 使用 Future

Scala 的 Future 让你对 Future 的结果指定变换然后得到一个新的 future 来表示这两个异步计算的组合：原始的计算和变换。

### 用 map对Future做变换

各种变换操作当中，最基础的操作是 map。 可以直接将下一个计算 map 到当前的 future ，而不是阻塞（等待结果）然后继续做另一个计算这样做的结果 将会是一个新的 future ，表示原始的异步计算结果经过传给 map 的函数异步变换后的结果。

例如，如下的 future 会在 10 秒钟后完成：

```scala
scala> val fut = Future {Thread.sleep(lOOOO); 21 + 21}
fut: scala.concurrent.Future[Int] = ...
```

对这个 future 映射一个增 1 的函数将会交出另一个 future。 这个新的 future 将表示由原始的加法和跟在后面的一次增 1 组成的计算：

```scala
scala> val result = fut.map(x => x + 1)
result: scala.concurrent.Future[Int] = .

scala> result.value 
res5: Option [seala.util.Try [Int]] = None
```

一旦原始的 future 完成，并且该函数被应用到其结果后， map 方法返回的那个future 也会完成 ：

```scala
scala> result.value 
res6: Option[scala.util.Try[Int]] = Some(Success(43))
```

注意，本例中执行的操作（创建 future 、计算 21 + 21 的和，以及 42 + 1) 有可能分别被三个不同的线程执行。

### 用 for表达式对Future做变换

由于 Scala 的 Future 还声明了 flatMap 方法，可以用 for 表达式来对 future 做变换。 例如，考虑下面两个 future ，它们分别在 10 秒后产出 42 和 46:

```scala
scala> val futl =Future { Thread.sleep(lOOOO); 21 + 21}
futl:scala.concurrent.Future[Int] = ...

scala> val fut2 =Future { Thread.sleep(lOOOO); 23 + 23}
fut2:scala.concurrent.Future[Int] = ...
```

有了这两个 future ，可以得到一个新的表示它们结果的异步和新 future,

就像这样：

```scala
scala> for { 
  		 	x <- futl 
  			y <- fut2 
			 } yield x + y 
res7 : scala.concurrent.Future[Int] = ...
```

一旦原始的两个 future 完成，并且后续的和计算也完成后，你将会看到如 下结果：

```scala
scala> res7.value 
res8: Option[scala.util.Try[Int]] = Some(Success(88))
```

由于 for 表达式会串行化它们的变换， 如果你不在 for 表达式之前创建 future ，它们就不会并行运行 。 例如，虽然前面的 for 表达式需要大约 10 秒钟 完成，下面这个 for 表达式至少需要 20 秒：

```scala
scala> for { 
  			x <-Future { Thread.sleep(lOOOO); 21 + 21} 
  			y <-Future { Thread.sleep(lOOOO); 23 + 23} 
			 } yield x + y 
res9: scala.concurrent.Future[Int]

scala> res9.value 
res27: Option[scala. util. Try[Int]] = None

scala> //至少需要 20 秒完成

scala> res9.value 
res28: Option[scala.util.Try[Int]] = Some(Success(88))
```



### 创建 Future: Future.failed、 Future.successful、Future.fromTry 和 Promise

除了我们在之前的例子中用来创建 Future 的 apply 方法之外， Future 伴生对象也提供了三个创建已然完成的 future 的工厂方法： successful , failed 和 fromTry。 ***这些工厂方法并不需要 ExecutionContext 。***

successful 这个工厂方法将创建一个已经成功完成的 future :

```scala
scala> Future.successful { 21 + 21 } 
res2: scala.concurrent.Future[Int] = ...
```

failed 方法将创建一个已经失败的 future :

```scala
scala> Future.failed(new Exception (” bummer !”)) 
res3: scala.concurrent.Future[Nothing] = ...
```

from Try 方法将从给定的 Try 创建一个已经完成的 future :

```scala
scala> import scala.util.{Success,Failure} 
import scala.util.{Success, Failure}

scala> Future.fromTry(Success{ 21 + 21 }) 
res4: scala.concurrent.Future[Int] = ...

scala> Future.fromTry(Failure(new Exception (” bummer !”))) resS: scala.concurrent.Future[Nothing] = .

```

***创建 future 最一般化的方式是使用 Promise。 给定一个 promise，可以得到一个由这个 promise 控制的 future。*** 当你完成 promise 时，对应的 future 也会完成。 

参考下面的例子:

```scala
scala> val pro = Promise[Int]
pro: scala.concurrent.Promise[Int]

scala> val fut = pro.future
fut: scala.concurrent.Future[Int]

scala> fut.value 
res8: Option[scala. util. Try[Int]] = None
```

可以用名为 success、failure 和 Complete 的方法来完成 promise。

Promise 的这些方法跟前面我们介绍过的构造已然完成的 future 的方法很像。 例如， success 方法将成功地完成 future :

```scala
scala> pro.success(42)
res13: pro.type = Future(Success(42))

scala> fut.value
res14: Option[scala.util.Try[Int]] = Some(Success(42))
```

failure 方法接收一个会让 future 因为它失败的异常 。 complete 方法接收一个 Try。 还有一个接收 future 的 completeWith 方法，这个方法将使得该 promise 的 future 的完成状态跟你传人的 future 保持同步。

### 过滤: filter和collect

