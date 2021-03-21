## 【Scalatest】同步测试

ScalaTest支持不同的测试样式，每种样式都旨在满足特定的需求。

官方建议为单元测试选择一种主要样式，为验收测试选择另一种主要样式。对单元测试和验收测试使用不同的样式可以帮助开发人员在低级单元测试到高级验收测试之间“切换”。您可能还希望选择要在特殊情况下使用的特定样式，例如 [`PropSpec`用于测试矩阵](http://doc.scalatest.org/3.1.2/#org.scalatest.PropSpec@testMatrix)。我们通常以与单元测试相同的方式编写集成测试（涉及数据库等子系统的测试）。

官方建议将 `FlatSpec`用于单元和集成测试, 将 `FeatureSpec`用于验收测试。以及将`FlatSpec`作为默认选择，因为它像大多数开发人员所熟悉的XUnit测试一样是扁平的（未嵌套），但是可以指导您编写具有描述性的规范样式名称的重点测试。

```scala
abstract class UnitSpec extends FunSpec
		with Matchers
		with OptionValues
		with Inside
		with Inspectors
```



### 0. Style trait use cases

下面将快速概述每种样式特征的优缺点。

> 日常使用, 个人建议熟悉 `FunSuite`、`FlatSpec`、`FunSpec` 这三种即可

#### FunSuite

`FunSuite`会感到既舒适又熟悉，同时仍然可以享受 ***BDD(行为驱动开发)*** 的一些好处：`FunSuite`可以轻松编写描述性的测试名称，自然而然地编写重点测试，并生成类似于规范的输出，以促进利益相关者之间的交流。

```scala
import org.scalatest.FunSuite

class SetSuite extends FunSuite {

  test("An empty Set should have size 0") {
    assert(Set.empty.size == 0)
  }

  test("Invoking head on an empty Set should produce NoSuchElementException") {
    assertThrows[NoSuchElementException] {
      Set.empty.head
    }
  }
}
```

#### FlatSpec

对于希望从xUnit迁移到BDD的团队来说，这是很好的第一步，`FlatSpec`的结构像xUnit一样平坦，非常简单和熟悉，但是测试名称必须以规范样式编写：“ X应该Y”，“ A必须B“ *等*。

```scala
import org.scalatest.FlatSpec

class SetSpec extends FlatSpec {

  "An empty Set" should "have size 0" in {
    assert(Set.empty.size == 0)
  }

  it should "produce NoSuchElementException when head is invoked" in {
    assertThrows[NoSuchElementException] {
      Set.empty.head
    }
  }
}
```

#### FunSpec

对于来自Ruby的RSpec工具的团队，他们对`FunSpec`会感到非常熟悉。一般而言，对于任何喜欢BDD的团队， `FunSpec`的嵌套和柔和的结构化指南（带有`describe`和`it`）为编写规范样式的测试提供了绝佳的通用选择。

```scala
import org.scalatest.FunSpec

class SetSpec extends FunSpec {

  describe("A Set") {
    describe("when empty") {
      it("should have size 0") {
        assert(Set.empty.size == 0)
      }

      it("should produce NoSuchElementException when head is invoked") {
        assertThrows[NoSuchElementException] {
          Set.empty.head
        }
      }
    }
  }
  
}
```

#### WordSpec

对于来自spec或spec2的团队，`WordSpec`会感到很熟悉，并且通常是将specN测试移植到ScalaTest的最自然的方法。`WordSpec`对于必须如何编写文本，它具有非常明确的规定，因此非常适合希望在其规范文本上强加严格纪律的团队。

```scala
import org.scalatest.WordSpec

class SetSpec extends WordSpec {

  "A Set" when {
    "empty" should {
      "have size 0" in {
        assert(Set.empty.size == 0)
      }

      "produce NoSuchElementException when head is invoked" in {
        assertThrows[NoSuchElementException] {
          Set.empty.head
        }
      }
    }
  }
}
```

#### FreeSpec

因为它为应该如何编写规范文本提供了绝对的自由（并且没有指导），`FreeSpec`所以对于具有BDD经验并且能够就如何构造规范文本达成共识的团队来说，这是一个不错的选择。

```scala
import org.scalatest.FreeSpec

class SetSpec extends FreeSpec {

  "A Set" - {
    "when empty" - {
      "should have size 0" in {
        assert(Set.empty.size == 0)
      }

      "should produce NoSuchElementException when head is invoked" in {
        assertThrows[NoSuchElementException] {
          Set.empty.head
        }
      }
    }
  }
}
```

#### PropSpec

`PropSpec` 非常适合希望只根据属性检查编写测试的团队。当选择不同的样式特征作为主要单元测试样式时，它也是编写偶尔的测试矩阵的不错选择。

```scala
import org.scalatest._
import prop._
import scala.collection.immutable._

class SetSpec extends PropSpec with TableDrivenPropertyChecks with Matchers {

  val examples =
    Table(
      "set",
      BitSet.empty,
      HashSet.empty[Int],
      TreeSet.empty[Int]
    )

  property("an empty Set should have size 0") {
    forAll(examples) { set =>
      set.size should be (0)
    }
  }

  property("invoking head on an empty set should produce NoSuchElementException") {
    forAll(examples) { set =>
       a [NoSuchElementException] should be thrownBy { set.head }
    }
  }
}
```

#### FeatureSpec

特性`FeatureSpec`主要用于验收测试，和促进程序员与非程序员一起定义接受要求的过程。

```scala
import org.scalatest._

class TVSet {
  private var on: Boolean = false
  def isOn: Boolean = on
  def pressPowerButton() {
    on = !on
  }
}

class TVSetSpec extends FeatureSpec with GivenWhenThen {

  info("As a TV set owner")
  info("I want to be able to turn the TV on and off")
  info("So I can watch TV when I want")
  info("And save energy when I'm not watching TV")

  feature("TV power button") {
    scenario("User presses power button when TV is off") {

      Given("a TV set that is switched off")
      val tv = new TVSet
      assert(!tv.isOn)

      When("the power button is pressed")
      tv.pressPowerButton()

      Then("the TV should switch on")
      assert(tv.isOn)
    }

    scenario("User presses power button when TV is on") {

      Given("a TV set that is switched on")
      val tv = new TVSet
      tv.pressPowerButton()
      assert(tv.isOn)

      When("the power button is pressed")
      tv.pressPowerButton()

      Then("the TV should switch off")
      assert(!tv.isOn)
    }
  }
}
```

#### RefSpec（JVM only）

`RefSpec`允许您将测试定义为方法，与将测试表示为函数的样式类相比，每个测试节省一个函数文字。较少的函数字面量可以缩短编译时间，并减少生成的类文件的数量，从而有助于最大程度地减少构建时间。因此，`Spec`对于大型项目而言，使用是一个不错的选择，因为大型项目需要考虑构建时间，以及通过静态代码生成器以编程方式生成大量测试时。

```scala
import org.scalatest.refspec.RefSpec

class SetSpec extends RefSpec {

  object `A Set` {
    object `when empty` {
      def `should have size 0` {
        assert(Set.empty.size == 0)
      }

      def `should produce NoSuchElementException when head is invoked` {
        assertThrows[NoSuchElementException] {
          Set.empty.head
        }
      }
    }
  }
}
```

### 1. Defining base classes

官方建议为项目创建抽象基类，这些基类将最常使用的功能混合在一起，从而避免重复地将相同特征混合在一起来复制代码。例如，可以为`UnitSpec`类的单元测试创建一个类（不是特质，以加快编译速度）（下面示例代码中使用了 ***FlatSpec***,具体使用时可以换成自己熟悉的, 如 FunSpec等）

```scala
package com.mycompany.myproject

import org.scalatest._

abstract class UnitSpec extends FlatSpec with Matchers with
  OptionValues with Inside with Inspectors
```

然后，我们可以使用自定义基类为项目编写单元测试，如下所示：

```scala
package com.mycompany.myproject

import org.scalatest._

class MySpec extends UnitSpec {
  // Your tests here
}
```

### 2. Using assertions

默认情况下，ScalaTest在任何样式特征中提供了三个通用的断言。您可以使用：

- `assert` 用于一般性断言；
- `assertResult` 区分期望值与实际值；
- `assertThrows` 以确保一些代码会引发预期的异常。

ScalaTest的断言定义在trait`Assertions`中，该特征由trait `Suite`扩展为所有样式trait。trait`Assertions`还提供：

- `assume`有条件地*取消*测试；
- `fail` 无条件地使测试失败；
- `cancel` 无条件取消测试；
- `succeed` 无条件地使测试成功；
- `intercept` 确保少量代码引发预期的异常，然后对异常进行断言；
- `assertDoesNotCompile` 确保部分代码不会编译；
- `assertCompiles` 确保可以编译一些代码；
- `assertTypeError` 确保由于类型（非解析）错误而导致部分代码无法编译；
- `withClue` 添加有关故障的更多信息。

所有这些构建体描述如下。

#### assert

在任何Scala程序中，都可通过调用`assert`并传递一个`Boolean`表达式来编写断言，如：

```scala
val left = 2
val right = 1
assert(left == right)

// where a is 1, b is 2, c is 3, d is 4, xs is List(a, b, c), and num is 1.0:

assert(a == b || c >= d)
// Error message: 1 did not equal 2, and 3 was not greater than or equal to 4

assert(xs.exists(_ == 4))
// Error message: List(1, 2, 3) did not contain 4

assert("hello".startsWith("h") && "goodbye".endsWith("y"))
// Error message: "hello" started with "h", but "goodbye" did not end with "y"

assert(num.isInstanceOf[Int])
// Error message: 1.0 was not instance of scala.Int

assert(Some(2).isEmpty)
// Error message: Some(2) was not empty

assert(None.isDefined)
// Error message: scala.None.isDefined was false

assert(xs.exists(i => i > 10))
// Error message: xs.exists(((i: Int) => i.>(10))) was false
```

#### assertResult

随着操作数变长，代码的可读性也会降低。此外，为`==`和`===`比较而生成的错误消息不能区分实际值和期望值。这些操作数仅被称为`left`和`right`，因为如果一个被命名`expected`而另一个被命名`actual`，则人们将很难记住哪个是哪个。为了帮助断言的这些限制，请`Suite`包括一种`assertResult`可以称为的替代方法`assert`。要使用`assertResult`，请将预期值放在括号后`assertResult`，后跟花括号，其中包含应产生期望值的代码。例如：

```scala
val a = 5 
val b = 2 
assertResult（2）{ 
  a-b 
}
```

#### fail

如果只需要测试失败，则可以编写：

```scala
fail()
```

或者，如果您希望测试失败并显示一条消息，则可以编写：

```scala
fail("I've got a bad feeling about this")
```

#### assertThrows和intercept

有时，您需要测试在某些情况下，例如何时将无效参数传递给该方法时，该方法是否引发预期的异常。

为了使这种常见用例更易于表达和阅读，ScalaTest提供了两种方法： `assertThrows`和`intercept`。使用方法`assertThrows`如下：

```scala
val s = “ hi” 
assertThrows [ IndexOutOfBoundsException ] { //结果类型：断言 
  s.charAt（-1）
}
```

如果 charAt 抛出一个 IndexOutOfBoundsException， assertThrows 将返 Succeeded。但是，如果 charAt 正常完成，或引发其他异常，***assertThrows*** 则会突然出现 ***TestFailedException***。

intercept 方法的作用和 assertThrows 一样，不同的是 intercept 不是返回 Succeeded，而是返回捕获的异常，这样就可以进一步检查它。例如，您可能需要确保异常中包含的数据具有期望值。这是一个例子：

```scala
val s = "hi"
val caught =
  intercept[IndexOutOfBoundsException] { // Result type: IndexOutOfBoundsException
    s.charAt(-1)
  }
assert(caught.getMessage.indexOf("-1") != -1)
```

#### Getting a clue

如果希望此特性的方法默认提供更多信息,  `assert`和`assertResult`提供直接包含线索的方法, 而`intercept`则没有。这是提供线索的示例：

```scala
assert(1 + 1 === 3, "this is a clue")
assertResult(3, "this is a clue") { 1 + 1 }
```

要在失败的`assertThrows`呼叫引发的异常的详细消息中获得相同的线索，需要使用`withClue`：

```java
withClue("this is a clue") {
  assertThrows[IndexOutOfBoundsException] {
    "hi".charAt(-1)
  }
}
```

### 3. Sharing fixtures

当多个测试需要使用相同的fixture进行工作时，要避免在这些测试之间重复创建fixture代码。您在测试中重复执行的代码越多，测试在重构实际生产代码上的阻力就越大。

ScalaTest建议使用三种技术来消除这种代码重复：

- 使用Scala重构
- 覆写 `withFixture`
- 混合*之前和之后的*特征

#### Calling get-fixture methods

如果您需要在多个测试中创建相同的可变fixture对象，并且在使用它们后无需清理它们，则最简单的方法是编写一个或多个*get-fixture*方法。每次调用get-fixture方法时，都会返回所需fixture对象（或包含多个夹具对象的支架对象）的新实例。

```scala
package org.scalatest.examples.flatspec.getfixture

import org.scalatest.FlatSpec
import collection.mutable.ListBuffer

class ExampleSpec extends FlatSpec {

  def fixture =
    new {
      val builder = new StringBuilder("ScalaTest is ")
      val buffer = new ListBuffer[String]
    }

  "Testing" should "be easy" in {
    val f = fixture
    f.builder.append("easy!"
    assert(f.builder.toString === "ScalaTest is easy!")
    assert(f.buffer.isEmpty)
    f.buffer += "sweet"
  }

  it should "be fun" in {
    val f = fixture
    f.builder.append("fun!")
    assert(f.builder.toString === "ScalaTest is fun!")
    assert(f.buffer.isEmpty)
  }
}
```

#### Instantiating fixture-context objects

当不同的测试需要 fixtrue 对象的不同组合时，另一种特别有用的技术是将fixtrue对象定义为实例化测试主体的 ***fixture-context*** 的实例变量。

要使用此技术，您需要在trait和/或class中定义由fixtrue对象初始化的实例变量，然后在每个测试中实例化一个仅包含测试所需fixtrue对象的对象。trait允许您仅将每个测试所需的fixtrue对象混合在一起，而class则允许您通过构造函数传递数据以配置fixtrue对象。

```java
package org.scalatest.examples.flatspec.fixturecontext

import collection.mutable.ListBuffer
import org.scalatest.FlatSpec

class ExampleSpec extends FlatSpec {

  trait Builder {
    val builder = new StringBuilder("ScalaTest is ")
  }

  trait Buffer {
    val buffer = ListBuffer("ScalaTest", "is")
  }

  // This test needs the StringBuilder fixture
  "Testing" should "be productive" in new Builder {
    builder.append("productive!")
    assert(builder.toString === "ScalaTest is productive!")
  }

  // This test needs the ListBuffer[String] fixture
  "Test code" should "be readable" in new Buffer {
    buffer += ("readable!")
    assert(buffer === List("ScalaTest", "is", "readable!"))
  }

  // This test needs both the StringBuilder and ListBuffer
  it should "be clear and concise" in new Builder with Buffer {
    builder.append("clear!")
    buffer += ("concise!")
    assert(builder.toString === "ScalaTest is clear!")
    assert(buffer === List("ScalaTest", "is", "concise!"))
  }
}
```

