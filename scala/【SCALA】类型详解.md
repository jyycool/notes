# 【SCALA】类型详解

scala 是一个强类型的语言,但是在编程时可以省略对象的类型.

### java中对象类型(type)与类(class)信息

jdk1.5 前 类型与类是一一映射,类一致类型就一致.

1.5 后引入了泛型, jvm 选择运行时擦除类型, 类型不可以只通过类信息进行判断.
 比如: List\<String>,List\<Integer> 的class 都是 Class\<List>,然而他们的类型是不相同的,泛型是需要通过反射来进行获得,
 同时java通过增加 Type 来表达这种类型.

List\<String> 的类型由 类构造器 和 类型参数 组成. 和 List\<Integer> 完全不相同.

### scala中类型

scala 没有用 java 自己的类型接口, 而使用 `scala.reflect.runtime.universe.Type` 接口

1. 类获得类型或类信息

   ```scala
   // 获取类型信息
   scala> import scala.reflect.runtime.universe._
   import scala.reflect.runtime.universe._
   
   scala> class A
   defined class A
   
   scala> typeOf[A]
   res1: reflect.runtime.universe.Type = A
   
   scala> classOf[A]
   res2: Class[A] = class A
   ```

2. 对象获得类信息

   另外java 中对象获取类信息可以通过 getClass方法, scala继承了这个方法.

   ```scala
   scala> val a = new A()
   a: A = A@3b64f131
   
   scala> a.getClass
   res3: Class[_ <: A] = class A
   ```



### scala 类型与类区别

```scala
scala> class B {class C}
defined class B

scala> val b1 = new B
b1: B = B@775a5a67

scala> val b2 = new B
b2: B = B@485547ac

scala> val c1 = new b1.C
c1: b1.C = B$C@10c1682b

scala> val c2 = new b2.C
c2: b2.C = B$C@4ceb368b

scala> c1.getClass
res4: Class[_ <: b1.C] = class B$C

scala> c1.getClass == c2.getClass
res5: Boolean = true

scala> typeOf[b1.C] == typeOf[b2.C]
res6: Boolean = false

scala> typeOf[b1.C]
res7: reflect.runtime.universe.Type = b1.C
```

类(class)与类型(type)是两个不一样的概念,类型(type)比类(class)更”具体”，任何数据都有类型。***类是面向对象系统里对同一类数据的抽象，在没有泛型之前，类型系统不存在高阶概念，直接与类一一映射，而泛型出现之后，就不在一一映射了。***

比如定义`class List[T] {}`, 可以有`List[Int]` 和 `List[String]`等具体类型，它们的类是同一个List，但类型则根据不同的构造参数类型而不同。

```scala
scala> typeOf[List[String]] == typeOf[List[Int]]
res9: Boolean = false

scala> classOf[List[String]] == classOf[List[Int]]
res10: Boolean = true
```



类包括类型,类型一致他们的类信息也是一致的. 类型不一致,但是类可能一致, 类型是所有编程语言都有的概念，一切数据都有类型。类更多存在于面向对象语言，非面向对象语言也有“结构体”等与之相似的概念；类是对数据的抽象，而类型则是对数据的”分类”，类型比类更“具体”，更“细”一些。

### scala 内部依赖类型

```scala
scala> class B {
  |class C
  |def foo(c: C) = println(c)
	|}
defined class B

scala> val b1 = new B
b1: B = B@775a5a67

scala> val b2 = new B
b2: B = B@485547ac

scala> val c1 = new b1.C
c1: b1.C = B$C@10c1682b

scala> val c2 = new b2.C
c2: b2.C = B$C@4ceb368b

scala> c1.getClass == c2.getClass
res5: Boolean = true

scala> typeOf[b1.C] == typeOf[b2.C]
res6: Boolean = false
```

在 java 中, 内部类创建的对象是相同的, 但是 Scala 中 c1 和 c2 是不同的类型(Type).

```scala
b1.foo(c2)
<console>:17: error: type mismatch;
 found   : b2.C
 required: b1.C
       b1.foo(c2)
              ^
```

b1.foo方法接受的参数类型为：b1.C，而传入的c2 类型是 b2.C，两者不匹配。

> 路径依赖类型；比如上面的 B.this.C 就是一个路径依赖类型，C 前面的路径 B.this 随着不同的实例而不同，比如 b1 和 b2 就是两个不同的路径，所以b1.C 与 b2.C也是不同的类型。

1. 内部类定义在Object中: 路径 package.object.Inner
2. 内部类定义在class/trait 里:

```scala
class A {
    class B
    val b = new B // 相当于 A.this.B
}
```

类路径  A.B

```scala
class A { 
  class B 
}

class C extends A { 
  val x = new super.B // 相当于 C.this.A.this.B
} 
```

怎么让 a1.foo 方法可以接收 b2 参数 ？

#### 类型投影(type projection)

在scala里，内部类型(排除定义在object内部的)，想要表达所有的外部类A实例路径下的B类型，即对 a1.B 和 a2.B及所有的 an.B类型找一个共同的父类型，这就是类型投影，用 A#B的形式表示。

```python
def foo(b: A#B)
```

### 结构类型

结构类型(structural type)为静态语言增加了部分动态特性，使得参数类型不再拘泥于某个已命名的类型，只要参数中包含结构中声明的方法或值即可。

```scala
def free(res: { def close():Unit }){
	res.close 
}
	
free(new {def close() = println("closed")})
// closed
```

也可以使用Type定义别名

```ruby
type X = { def close():Unit }
defined type alias X
def free(res:X) = res.close
free(new { def close() = println("closed") })
closed
```

如果某一对象实现了close方法也可以直接传入

```scala
object A { 
  def close() {println("A closed")} 
}

free(A)
A closed
```

结构类型还可以用在稍微复杂一点的“复合类型”中，比如：

```ruby
scala> trait X1; trait X2;

scala> def test(x: X1 with X2 { def close():Unit } ) = x.close
```

上面声明 test 方法参数的类型为：`X1 with X2 { def close():Unit }`

表示参数需要符合特质X1和X2同时也要有定义close方法

### Type 定义类型

type S = String

可以给 String类型起一个别名 S

可以用于抽象类型

```scala
scala> trait A { type T; def foo(i:T) = print(i) }

scala> class B extends A { type T = Int }

scala> val b = new B

scala> b.foo(200)
200

scala> class C extends A { type T = String }

scala> val c = new C

scala> c.foo("hello")
hello
```

### 泛型子类型/父类型

在Java泛型里表示某个类型是Test类型的父类型/子类型，使用super/extends关键字：

```dart
<T super Test>
<T extends Test>

//或用通配符的形式：
<? super Test>
<? extends Test>
```

scala 中使用

```json
[T >: Test]
[T <: Test]

//或用通配符:
[_ >: Test]
[_ <: Test]
```

lower bound适用于把泛型对象当作数据的消费者的场景下:

### 数组类型

协变 : A 是 B 的子类型, List[A] 也是 List[B] 的子类型

逆变 : A 是 B 的父类型, List[A] 也是 List[B] 的父类型

在 java 中引用类型的数组类型是支持协变的, 即 String[] 类型是 Object[] 的子类型,

scala里不支持数组的协变，以尝试保持比java更高的纯粹性。即 Array[String]的实例对象不能赋给Array[Any]类型的变量。

不同于java里其他泛型集合的实现，数组类型中的类型参数在运行时是必须的，即 [Ljava/lang/String 与 [Ljava/lang/Object 是两个不同的类型，不像 Collection\<String> 与 Collection\<StringBuilder> 运行时参数类型都被擦拭掉，成为Collection\<Object>。

