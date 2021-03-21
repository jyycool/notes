### 【SCALA】闭包

#### 摘要

本文通过 Scala 语言来实现一个简单的闭包，并且通过 Opcode 来深入理解 Scala 中闭包的实现原理。

#### 闭包

闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量。

再简单一些的说函数体内部引用了函数外部的一个或多个变量, 那么这个函数就称为一个闭包。 

下面我们来通过一个简单的例子实现 Scala 中的闭包，代码如下：

```ruby
object Closures {
  
  def main(args: Array[String]): Unit = {
    val addOne = makeAdd(1)
    val addTwo = makeAdd(2)
    
    println(addOne(1))
    println(addTwo(1))
  }
  
  def makeAdd(more: Int) = (x: Int) => x + more 
  def normalAdd(a: Int, b: Int) = a + b
}
```

我们定义了一个函数 makeAdd，输入参数是 Int 类型，返回的是一个函数（其实可以看成函数，后面我们会深入去研究到底是什么），同样我们定义了一个普通的函数 normalAdd 来进行比较，main 方法中，首先我们通过调用 makeAdd 来定义了两个 val：addOne 和 addTwo 并分别传入 1 和 2，然后执行并打印 addOne(1) 和 addTwo(2)，运行的结果是 2 和 3。



#### 分析

接下来我们来详细的分析一下上面这个例子的 Opcode，通过 javap 命令来查看 Closures.class 的字节码：

```objectivec
  Last modified Feb 14, 2017; size 1311 bytes
  MD5 checksum 02722b1fa1195c63a6dd0c8615db8c26
  Compiled from "Closures.scala"
public final class com.learn.scala.Closures$
  minor version: 0
  major version: 50
  flags: ACC_PUBLIC, ACC_FINAL, ACC_SUPER
Constant pool:
   #1 = Utf8               com/learn/scala/Closures$
   #2 = Class              #1             // com/learn/scala/Closures$
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             // java/lang/Object
   #5 = Utf8               Closures.scala
   #6 = Utf8               MODULE$
   #7 = Utf8               Lcom/learn/scala/Closures$;
   #8 = Utf8               <clinit>
   #9 = Utf8               ()V
  #10 = Utf8               <init>
  #11 = NameAndType        #10:#9         // "<init>":()V
  #12 = Methodref          #2.#11         // com/learn/scala/Closures$."<init>":()V
  #13 = Utf8               main
  #14 = Utf8               ([Ljava/lang/String;)V
  #15 = Utf8               makeAdd
  #16 = Utf8               (I)Lscala/Function1;
  #17 = NameAndType        #15:#16        // makeAdd:(I)Lscala/Function1;
  #18 = Methodref          #2.#17         // com/learn/scala/Closures$.makeAdd:(I)Lscala/Function1;
  #19 = Utf8               scala/Predef$
  #20 = Class              #19            // scala/Predef$
  #21 = Utf8               Lscala/Predef$;
  #22 = NameAndType        #6:#21         // MODULE$:Lscala/Predef$;
  #23 = Fieldref           #20.#22        // scala/Predef$.MODULE$:Lscala/Predef$;
  #24 = Utf8               scala/Function1
  #25 = Class              #24            // scala/Function1
  #26 = Utf8               apply$mcII$sp
  #27 = Utf8               (I)I
  #28 = NameAndType        #26:#27        // apply$mcII$sp:(I)I
  #29 = InterfaceMethodref #25.#28        // scala/Function1.apply$mcII$sp:(I)I
  #30 = Utf8               scala/runtime/BoxesRunTime
  #31 = Class              #30            // scala/runtime/BoxesRunTime
  #32 = Utf8               boxToInteger
  #33 = Utf8               (I)Ljava/lang/Integer;
  #34 = NameAndType        #32:#33        // boxToInteger:(I)Ljava/lang/Integer;
  #35 = Methodref          #31.#34        // scala/runtime/BoxesRunTime.boxToInteger:(I)Ljava/lang/Integer;
  #36 = Utf8               println
  #37 = Utf8               (Ljava/lang/Object;)V
  #38 = NameAndType        #36:#37        // println:(Ljava/lang/Object;)V
  #39 = Methodref          #20.#38        // scala/Predef$.println:(Ljava/lang/Object;)V
  #40 = Utf8               this
  #41 = Utf8               args
  #42 = Utf8               [Ljava/lang/String;
  #43 = Utf8               addOne
  #44 = Utf8               Lscala/Function1;
  #45 = Utf8               addTwo
  #46 = Utf8               com/learn/scala/Closures$$anonfun$makeAdd$1
  #47 = Class              #46            // com/learn/scala/Closures$$anonfun$makeAdd$1
  #48 = Utf8               (I)V
  #49 = NameAndType        #10:#48        // "<init>":(I)V
  #50 = Methodref          #47.#49        // com/learn/scala/Closures$$anonfun$makeAdd$1."<init>":(I)V
  #51 = Utf8               more
  #52 = Utf8               I
  #53 = Utf8               normalAdd
  #54 = Utf8               (II)I
  #55 = Utf8               a
  #56 = Utf8               b
  #57 = Methodref          #4.#11         // java/lang/Object."<init>":()V
  #58 = NameAndType        #6:#7          // MODULE$:Lcom/learn/scala/Closures$;
  #59 = Fieldref           #2.#58         // com/learn/scala/Closures$.MODULE$:Lcom/learn/scala/Closures$;
  #60 = Utf8               Code
  #61 = Utf8               LocalVariableTable
  #62 = Utf8               LineNumberTable
  #63 = Utf8               Signature
  #64 = Utf8               (I)Lscala/Function1<Ljava/lang/Object;Ljava/lang/Object;>;
  #65 = Utf8               SourceFile
  #66 = Utf8               InnerClasses
  #67 = Utf8               ScalaInlineInfo
  #68 = Utf8               Scala
{
  public static final com.learn.scala.Closures$ MODULE$;
    descriptor: Lcom/learn/scala/Closures$;
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL

  public static {};
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: new           #2                  // class com/learn/scala/Closures$
         3: invokespecial #12                 // Method "<init>":()V
         6: return

  public void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=4, args_size=2
         0: aload_0
         1: iconst_1
         2: invokevirtual #18                 // Method makeAdd:(I)Lscala/Function1;
         5: astore_2
         6: aload_0
         7: iconst_2
         8: invokevirtual #18                 // Method makeAdd:(I)Lscala/Function1;
        11: astore_3
        12: getstatic     #23                 // Field scala/Predef$.MODULE$:Lscala/Predef$;
        15: aload_2
        16: iconst_1
        17: invokeinterface #29,  2           // InterfaceMethod scala/Function1.apply$mcII$sp:(I)I
        22: invokestatic  #35                 // Method scala/runtime/BoxesRunTime.boxToInteger:(I)Ljava/lang/Integer;
        25: invokevirtual #39                 // Method scala/Predef$.println:(Ljava/lang/Object;)V
        28: getstatic     #23                 // Field scala/Predef$.MODULE$:Lscala/Predef$;
        31: aload_3
        32: iconst_1
        33: invokeinterface #29,  2           // InterfaceMethod scala/Function1.apply$mcII$sp:(I)I
        38: invokestatic  #35                 // Method scala/runtime/BoxesRunTime.boxToInteger:(I)Ljava/lang/Integer;
        41: invokevirtual #39                 // Method scala/Predef$.println:(Ljava/lang/Object;)V
        44: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      45     0  this   Lcom/learn/scala/Closures$;
            0      45     1  args   [Ljava/lang/String;
            6      38     2 addOne   Lscala/Function1;
           12      32     3 addTwo   Lscala/Function1;
      LineNumberTable:
        line 6: 0
        line 7: 6
        line 9: 12
        line 10: 28

  public scala.Function1<java.lang.Object, java.lang.Object> makeAdd(int);
    descriptor: (I)Lscala/Function1;
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=2, args_size=2
         0: new           #47                 // class com/learn/scala/Closures$$anonfun$makeAdd$1
         3: dup
         4: iload_1
         5: invokespecial #50                 // Method com/learn/scala/Closures$$anonfun$makeAdd$1."<init>":(I)V
         8: areturn
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/learn/scala/Closures$;
            0       9     1  more   I
      LineNumberTable:
        line 13: 0
    Signature: #64                          // (I)Lscala/Function1<Ljava/lang/Object;Ljava/lang/Object;>;

  public int normalAdd(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: ireturn
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       4     0  this   Lcom/learn/scala/Closures$;
            0       4     1     a   I
            0       4     2     b   I
      LineNumberTable:
        line 15: 0
}
SourceFile: "Closures.scala"
InnerClasses:
     public final #47; //class com/learn/scala/Closures$$anonfun$makeAdd$1
Error: unknown attribute
  ScalaInlineInfo: length = 0x18
   01 01 00 04 00 0A 00 09 01 00 0D 00 0E 01 00 0F
   00 10 01 00 35 00 36 01
Error: unknown attribute
  Scala: length = 0x0
```



我们先来看一下 makeAdd 部分（为了方便理解，下文中的图中只是简单的表示，类似常量池的部分并没有在图中表现出来）
 首先通过 new 实例化了一个 ***class com/learn/scala/Closures$$anonfun$makeAdd$1***，图中的 Heap 中简单的表示为 makeAdd；

![a1](/Users/sherlock/Desktop/notes/allPics/Scala/a1.png)



dup 命令就是复制一份上一步骤分配的空间的引用，并压入 operand stack 的栈底

![a2](/Users/sherlock/Desktop/notes/allPics/Scala/a2.png)



iload_1 就是将 LocalVariableTable 中的 more 压入 operand stack

![a3](/Users/sherlock/Desktop/notes/allPics/Scala/a3.png)



然后 invokespecial 将栈中的两个值 pop 出来执行 init 操作，也就是将 more 的具体值传入到 makeAdd 的初始化操作的函数中（可以认为是构造函数），然后将得到的新的实例化对象的引用压入 operand stack

![a4](/Users/sherlock/Desktop/notes/allPics/Scala/a4.png)

最后使用 areturn 返回这个引用，即将 initedmakeAddRef pop 出来

由此可以看出，Scala 中实际上是在 Heap 中创建了一个 makeAdd 的实例化对象，所以 more 变量在下次调用的时候依然可以使用，而普通方法的局部变量在调用的时候是压入 Operand stack 中的，计算完成之后就会 pop 出，所以在函数的调用完成后就不能在访问这个变量，下面我们来通过 main 方法中的具体执行来验证此结论。

我们只看关键的部分：
 iconst_1：将 1 压入 operand stack
 invokevirtual #18 将执行上面我们分析过的 makeAdd 函数调用的部分，调用的时候将 1 作为参数传入进行初始化，并将最终得到的实例对象的引用压入 operand stack
 astore_2 将上一步骤的引用 pop 出来并存储到 Local Variable Array 中
 至此执行了 main 方法中的 `val addOne = makeAdd(1)`

7、8、11 重复执行了上面的步骤，即代码中的第二行：`val addTwo = makeAdd(2)`
 此时，heap 有两个对象的实例，然后后面的 15、16、17、22、25 和 31、32、33、38、41 分别执行了`println(addOne(1))`和`println(addTwo(1))`的部分，比较关键的部分是 invokeinterface 的时候执行了具体对象的 apply 方法，并将 1 作为参数传入，其它部分在此不再详细说明。

实际上我们可以在 bin 目录下看到一个名为Closures$$anonfun$makeAdd$1的类文件，我们用 javap 命令可以看到，实际上是调用了实例对象的 apply 方法：



```cpp
public final int apply(int);
    descriptor: (I)I
    flags: ACC_PUBLIC, ACC_FINAL
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: iload_1
         2: invokevirtual #23                 // Method apply$mcII$sp:(I)I
         5: ireturn
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0  this   Lcom/learn/scala/Closures$$anonfun$makeAdd$1;
            0       6     1     x   I
      LineNumberTable:
        line 13: 0
```

至此，我们验证了我们的结论，Scala 实现闭包的方法是在 heap 中保存了使用不同参数初始化而产生的不同对象，对象中保存了变量的状态，然后调用具体对象的 apply 方法而最后产生不同的结果。

### 结论

与 Java 中使用内部类实现闭包相比，Scala 中为函数创建了一个对象 Function1 来保存变量的状态，然后具体执行的时候调用对应实例的 apply 方法，实现了函数作用域外也可以访问函数外部的变量。


