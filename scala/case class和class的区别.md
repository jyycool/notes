## case class和class的区别

在 [Scala](https://www.iteblog.com/archives/tag/scala/) 中存在***case class***，它其实就是一个普通的 ***class***。但是它又和普通的 ***class*** 略有区别

1. ***初始化的时候可以不用 new，当然你也可以加上，普通类一定需要加 new ( 不考虑伴生对象apply )***

   ```scala
   scala> case class Cat(name:String)
   defined class Cat
   
   scala> Cat("miao")
   res0: Cat = Cat(miao)
   ```

2. ***toString的实现更漂亮***

   ```scala
   scala> case class Cat(name:String)
   defined class Cat
   
   scala> Cat("miao")
   res0: Cat = Cat(miao)
   
   scala> res0
   res2: Cat = Cat(miao)
   ```

3. **默认实现了equals 和hashCode**

   ```scala
   scala> case class Cat(name:String)
   defined class Cat
   
   scala> Cat("miao")
   res0: Cat = Cat(miao)
   
   scala> Cat("miao")
   res3: Cat = Cat(miao)
   
   scala> res0 == res3
   res4: Boolean = true
   
   scala> res3.hashCode
   res6: Int = 1157011823
   ```

4. ***默认是可以序列化的，也就是实现了Serializable***

   ```scala
   scala> case class Cat(name:String)
   defined class Cat
   
   scala> Cat("miao")
   res3: Cat = Cat(miao)
   
   scala> import java.io._
   import java.io._
   
   scala> val bos = new ByteArrayOutputStream()
   bos: java.io.ByteArrayOutputStream =
   
   scala> val ops = new ObjectOutputStream(bos)
   ops: java.io.ObjectOutputStream = java.io.ObjectOutputStream@5bf0d49
   
   scala> ops.writeObject(res3)
   
   // class A 默认是没有实现序列化, 所以 writebject方法报错
   scala> class A
   defined class A
   
   scala> val a = new A()
   a: A = A@3e6ef8ad
   
   scala> ops.writeObject(a)
   java.io.NotSerializableException: A
     at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1184)
     at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:348)
     ... 32 elided
   ```

   

5. ***自动从scala.Product中继承一些函数***

6. ***case class构造函数的参数是public级别的，我们可以直接访问***

   ```scala
   scala> res3.name
   res11: String = miao
   ```

   

7. ***支持模式匹配***

   > 其实感觉case class最重要的特性应该就是支持模式匹配。这也是我们定义case class的唯一理由，难怪[Scala](https://www.iteblog.com/archives/tag/scala/)官方也说：**It makes only sense to define case classes if pattern matching is used to decompose data structures.** 

   ```scala
   object TermTest extends scala.App {
   
     def printTerm(term: Term) {
       term match {
         case Var(n) => print(n)
         case Fun(x, b) =>
         	print("^" + x + ".")
           printTerm(b)
         case App(f, v) =>
           print("(")
           printTerm(f)
           print(" ")
           printTerm(v)
           print(")")
       }
     }
   
     def isIdentityFun(term: Term): Boolean = term match {
       case Fun(x, Var(y)) if x == y => true
       case _ => false
     }
   
     val id = Fun("x", Var("x"))
     val t = Fun("x", Fun("y", App(Var("x"), Var("y"))))
     printTerm(t)
     println
     println(isIdentityFun(id))
     println(isIdentityFun(t))
   }
   ```

   

