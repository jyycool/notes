# 【Scalatest】异步测试

## Asynchronous style traits

ScalaTest支持异步非阻塞测试。给定`Future`您正在测试的代码返回的值，您无需阻塞，直到`Future`对它的值执行断言为止。您可以改为将这些断言映射到，`Future`然后将结果返回`Future[Assertion]`给ScalaTest。完成后，测试将异步`Future[Assertion]`完成。

以下是受支持的异步样式特征： 

- `AsyncFeatureSpec`
- `AsyncFlatSpec`
- `AsyncFreeSpec`
- `AsyncFunSpec`
- `AsyncFunSuite`
- `AsyncWordSpec`

异步 traits 特征遵循其同步的样式。例如，这是一个`AsyncFlatSpec`：

```scala
package org.scalatest.examples.asyncflatspec

import org.scalatest.AsyncFlatSpec
import scala.concurrent.Future

class AddSpec extends AsyncFlatSpec {

  def addSoon(addends: Int*): Future[Int] = Future { addends.sum }

  behavior of "addSoon"

  it should "eventually compute a sum of passed Ints" in {
    val futureSum: Future[Int] = addSoon(1, 2)
    // You can map assertions onto a Future, then return
    // the resulting Future[Assertion] to ScalaTest:
    futureSum map { sum => assert(sum == 3) }
  }

  def addNow(addends: Int*): Int = addends.sum

  "addNow" should "immediately compute a sum of passed Ints" in {
    val sum: Int = addNow(1, 2)
    // You can also write synchronous tests. The body
    // must have result type Assertion:
    assert(sum == 3)
  }
}
```

在 Scala 交互界面运行 `AddSpec`会产生如下结果

```
addSoon
  - should eventually compute a sum of passed Ints
  - should immediately compute a sum of passed Ints
```

从版本3.0.0开始，ScalaTest 断言和匹配器的结果类型为`Assertion`。因此，以上示例中第一个测试的结果类型为`Future[Assertion]`。为了清楚起见，以下是REPL会话中的相关代码：

```sh
scala> import org.scalatest._
import org.scalatest._

scala> import Assertions._
import Assertions._

scala> import scala.concurrent.Future
import scala.concurrent.Future

scala> import scala.concurrent.ExecutionContext
import scala.concurrent.ExecutionContext

scala> implicit val executionContext = ExecutionContext.Implicits.global
executionContext: scala.concurrent.ExecutionContextExecutor = scala.concurrent.impl.ExecutionContextImpl@26141c5b

scala> def addSoon(addends: Int*): Future[Int] = Future { addends.sum }
addSoon: (addends: Int*)scala.concurrent.Future[Int]

scala> val futureSum: Future[Int] = addSoon(1, 2)
futureSum: scala.concurrent.Future[Int] = scala.concurrent.impl.Promise$DefaultPromise@721f47b2

scala> futureSum map { sum => assert(sum == 3) }
res0: scala.concurrent.Future[org.scalatest.Assertion] = scala.concurrent.impl.Promise$DefaultPromise@3955cfcb
```

第二个测试的结果类型为`Assertion`：

```sh
scala> def addNow(addends: Int*): Int = addends.sum
addNow: (addends: Int*)Int

scala> val sum: Int = addNow(1, 2)
sum: Int = 3

scala> assert(sum == 3)
res1: org.scalatest.Assertion = Succeeded
```

当`AddSpec`被构造，第二个测试将被注册和隐式转换为`Future[Assertion]`。隐式转换是从`Assertion`到`Future[Assertion]`，因此您必须在ScalaTest assertion或 matcher 表达式中结束同步测试。如果一个测试中不是以`Assertion`类型作为输出，则可以将`succeed`放在测试的末尾。`succeed`，是trait `Assertions`中的一个字段，返回`Succeeded`例子：

```sh
scala> succeed
res2: org.scalatest.Assertion = Succeeded
```

因此，放置`succeed`在测试主体的末端将满足类型检查器的要求：

```scala
"addNow" should "immediately compute a sum of passed Ints" in {
  val sum: Int = addNow(1, 2)
  assert(sum == 3)
  println("hi") // println has result type Unit
  succeed       // succeed has result type Assertion
}
```

- 非阻塞
- 可以在`Future` 完成之前进行`Assertion`，也就是说，返回`Future[Assertion]`而不是`Assertion`
- 线程安全的
- 单线程串行执行上下文
- `Futures`按开始的顺序一个接一个执行和完成
- 用于将任务放入测试体中的线程也用于随后执行这些任务
- `Assertion`可以映射到 `Future`
- 不需要在测试体内部使用阻塞(例如, `Await`, `WhenReady`)
- 测试体中的返回值必须是`Future[Assertion]`
- 在测试主体中不支持多个断言
- 不能在测试体内部使用阻塞结构，因为它将永远挂起测试，因为它等待已入队但从未启动的任务

## ScalaFutures

```scala
class ScalaFuturesSpec extends FlatSpec with ScalaFutures {
  ...
  whenReady(Future(3) { v => assert(v == 3) }
  ...
}
```

- 阻塞
- 必须等待 `Future` 完成，然后才能返回`Assertion`
- 非线程安全
- 需要与`scala.concurrent.ExecutionContext.Implicits.global`一起使用。它是一个用于并行执行的线程池
- 在同一个测试主体中支持多个断言
- 测试主体中的返回值不必是 `Assertion`类型



## Eventually

```scala
class EventuallySpec extends FlatSpec with Eventually {
  ...
  eventually { assert(Future(3).value.contains(Success(3))) }
  ...
}
```

- 普通的测试工具，不单单是为了测试 `Future` 

- 这里的语义是重试按名称传递的任何类型的代码块，直到满足断言为止

- 在测试 `Future` 时，会使用`scala.concurrent.ExecutionContext.Implicits.global`

- 主要用于集成测试，针对响应时间不可预测的实际服务进行测试

## Single-threaded serial execution model vs. thread-pooled global execution model

给定如下测试代码

```scala
val f1 = Future {
  val tmp = mutableSharedState
  Thread.sleep(5000)
  println(s"Start Future1 with mutableSharedState=$tmp in thread=${Thread.currentThread}")
  mutableSharedState = tmp + 1
  println(s"Complete Future1 with mutableSharedState=$mutableSharedState")
}

val f2 = Future {
  val tmp = mutableSharedState
  println(s"Start Future2 with mutableSharedState=$tmp in thread=${Thread.currentThread}")
  mutableSharedState = tmp + 1
  println(s"Complete Future2 with mutableSharedState=$mutableSharedState")
}

for {
  _ <- f1
  _ <- f2
} yield {
  assert(mutableSharedState == 2)
}
```

我们主要比较 `AsyncSpec` 和`ScalaFuturesSpec`的输出

- `testOnly example.AsyncSpec`:

  ```scala
  Start Future1 with mutableSharedState=0 in thread=Thread[pool-11-thread-3-ScalaTest-running-AsyncSpec,5,main]
  Complete Future1 with mutableSharedState=1
  Start Future2 with mutableSharedState=1 in thread=Thread[pool-11-thread-3-ScalaTest-running-AsyncSpec,5,main]
  Complete Future2 with mutableSharedState=2
  ```

- `testOnly example.ScalaFuturesSpec`:

  ```scala
  Start Future2 with mutableSharedState=0 in thread=Thread[scala-execution-context-global-119,5,main]
  Complete Future2 with mutableSharedState=1
  Start Future1 with mutableSharedState=0 in thread=Thread[scala-execution-context-global-120,5,main]
  Complete Future1 with mutableSharedState=1
  ```

注意，在串行执行模型中，同一个线程是如何被使用的，`Futures`是如何按顺序完成的。另一方面，在全局执行模型中使用了多线程，Future2在Future1之前完成，这会导致共享可变状态的竞态条件，从而导致测试失败。

## 如何选择异步测试

在单元测试中，我们应该使用模拟子系统，在该子系统中，返还的`Futures`应几乎立即完成，因此`Eventually`单元中无需进行`Eventually`测试。 因此，选择是在`Asynchronous style traits`和`ScalaFutures`之间。 两者之间的主要区别在于，前者与后者不同，是非阻塞的。 如果可能的话，我们永远不要阻塞，所以我们应该更喜欢`AsyncFlatSpec`类的异步样式。 执行模型还有更大的不同。 默认情况下，异步样式使用自定义串行执行模型，该模型在共享的可变状态下提供线程安全性，这与`ScalaFutures`通常使用的全局线程池支持的执行模型`ScalaFutures` 。 总之，我的建议是除非有充分的理由, 不然都选择`Asynchronous style traits`。

