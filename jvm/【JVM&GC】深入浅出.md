

# 【JVM&GC】深入浅出



## 1. JVM简介

JVM是java的核心和基础，在java编译器和os平台之间的虚拟处理器。它是一种利用软件方法实现的抽象的计算机基于下层的操作系统和硬件平台，可以在上面执行java的字节码程序。

java编译器只要面向JVM，生成JVM能理解的代码或字节码文件。Java源文件经编译成字节码程序，通过JVM将每一条指令翻译成不同平台机器码，通过特定平台运行。

### 1.1 运行过程

Java语言写的源程序通过Java编译器，编译成与平台无关的‘字节码程序’(.class文件，也就是0，1二进制程序)，然后在OS之上的Java解释器中解释执行。

C++以及Fortran这类编译型语言都会通过一个静态的编译器将程序编译成CPU相关的二进制代码。

PHP以及Perl这列语言则是解释型语言，只需要安装正确的解释器，它们就能运行在任何CPU之上。当程序被执行的时候，程序代码会被逐行解释并执行。

------

### 1.2 JIT

1. 编译型语言的优缺点：
   - 速度快：因为在编译的时候它们能够获取到更多的有关程序结构的信息，从而有机会对它们进行优化。
   - 适用性差：它们编译得到的二进制代码往往是CPU相关的，在需要适配多种CPU时，可能需要编译多次。

2. 解释型语言的优缺点：
   - 适应性强：只需要安装正确的解释器，程序在任何CPU上都能够被运行
   - 速度慢：因为程序需要被逐行翻译，导致速度变慢。同时因为缺乏编译这一过程，执行代码不能通过编译器进行优化。

3. Java的做法是找到编译型语言和解释性语言的一个中间点：
   - Java代码会被编译：被编译成Java字节码，而不是针对某种CPU的二进制代码。
   - Java代码会被解释：Java字节码需要被java程序解释执行，此时，Java字节码被翻译成CPU相关的二进制代码。
   - JIT编译器的作用：在程序运行期间，将Java字节码编译成平台相关的二进制代码。正因为此编译行为发生在程序运行期间，所以该编译器被称为Just-In-Time编译器。

![JVM编译](/Users/sherlock/Desktop/notes/allPics/Java/JVM编译.png)

![JVM编译运行](/Users/sherlock/Desktop/notes/allPics/Java/JVM编译运行.png)



## 2. class文件的结构













## 3. JVM结构

![JMM](/Users/sherlock/Desktop/notes/allPics/Java/JMM.png)

> java是基于一门虚拟机的语言，所以了解并且熟知虚拟机运行原理非常重要。

### 3.1 方法区

方法区，Method Area， 对于习惯在HotSpot虚拟机上开发和部署程序的开发者来说，很多人愿意把方法区称为“永久代”（Permanent Generation），本质上两者并不等价，仅仅是因为HotSpot虚拟机的设计团队选择把GC分代收集扩展至方法区，或者说使用永久代来实现方法区而已。对于其他虚拟机（如BEA JRockit、IBM J9等）来说是不存在永久代的概念的。

主要存放已被虚拟机加载的***<u>类信息、常量、静态变量、即时编译器编译后的代码</u>***等数据（比如spring 使用IOC或者AOP创建bean时，或者使用cglib，反射的形式动态生成class信息等）。

>注意：JDK 6 时，String等字符串常量的信息是置于方法区中的，但是到了JDK 7 时，已经移动到了Java堆。所以，方法区也好，Java堆也罢，到底详细的保存了什么，其实没有具体定论，要结合不同的JVM版本来分析。

> ***OOM***
>
> 当方法区无法满足内存分配需求时，将抛出OutOfMemoryError。
>  运行时常量池溢出：比如一直往常量池加入数据，就会引起OutOfMemoryError异常。

#### 3.1.1 类信息

1. 类型全限定名。
2. 类型的直接超类的全限定名（除非这个类型是java.lang.Object，它没有超类）。
3. 类型是类类型还是接口类型。
4. 类型的访问修饰符（public、abstract或final的某个子集）。
5. 任何直接超接口的全限定名的有序列表。
6. 类型的常量池。
7. 字段信息。
8. 方法信息。
9. 除了常量意外的所有类（静态）变量。
10. 一个到类ClassLoader的引用。
11. 一个到Class类的引用。

#### 3.1.2 常量池

##### 3.1.2.1 Class文件中常量池

在Class文件结构中，最头的4个字节用于存储Magic Number，用于确定一个文件是否能被JVM接受，再接着4个字节用于存储版本号，前2个字节存储次版本号，后2个存储主版本号，再接着是用于存放常量的常量池，由于常量的数量是不固定的，所以常量池的入口放置一个U2类型的数据(constant_pool_count)存储常量池容量计数值。

常量池主要用于存放两大类常量：***字面量(Literal)***和***符号引用量(Symbolic References)***，字面量相当于Java语言层面常量的概念，如文本字符串，声明为final的常量值等，符号引用则属于编译原理方面的概念，包括了如下三种类型的常量：

- 类和接口的全限定名
- 字段名称和描述符
- 方法名称和描述符

> 符号引用; 就将它简单理解为符号的特定解释, 类似于说我们看到WC就认为是卫生间一个道理

##### 3.1.1.2 运行时常量池

Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。

运行时常量池相对于Class文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入CLass文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用比较多的就是String类的intern()方法。

##### 2.1.1.3 常量池的好处

常量池是为了避免频繁的创建和销毁对象而影响系统性能，其实现了对象的共享。

例如字符串常量池，在编译阶段就把所有的字符串文字放到一个常量池中。

- （1）节省内存空间：常量池中所有相同的字符串常量被合并，只占用一个空间。
- （2）节省运行时间：比较字符串时，==比equals()快。对于两个引用变量，只用==判断引用是否相等，也就可以判断实际值是否相等。

> 双等号==的含义
>
> - 基本数据类型之间应用双等号，比较的是他们的数值。
> - 复合数据类型(类)之间应用双等号，比较的是他们在内存中的存放地址。

##### 3.1.1.4 基本类型的包装类和常量池

java中基本类型的包装类的大部分都实现了常量池技术，即Byte,Short,Integer,Long,Character,Boolean。

这5种包装类默认创建了数值[-128，127]的相应类型的缓存数据，但是超出此范围仍然会去创建新的对象。 两种浮点数类型的包装类Float,Double并没有实现常量池技术。

###### Integer与常量池

```csharp
Integer i1 = 40;
Integer i2 = 40;
Integer i3 = 0;
Integer i4 = new Integer(40);
Integer i5 = new Integer(40);
Integer i6 = new Integer(0);
 
System.out.println("i1=i2   " + (i1 == i2));
System.out.println("i1=i2+i3   " + (i1 == i2 + i3));
System.out.println("i1=i4   " + (i1 == i4));
System.out.println("i4=i5   " + (i4 == i5));
System.out.println("i4=i5+i6   " + (i4 == i5 + i6));  
System.out.println("40=i5+i6   " + (40 == i5 + i6));
 
 
i1=i2   true
i1=i2+i3   true
i1=i4   false
i4=i5   false
i4=i5+i6   true
40=i5+i6   true
```

***解释：***

> 1. Integer i1=40；Java在编译的时候会直接将代码封装成Integer i1=Integer.valueOf(40);，从而使用常量池中的对象。
>
> 2. Integer i1 = new Integer(40);这种情况下会创建新的对象。
> 3. 语句i4 == i5 + i6，因为+这个操作符不适用于Integer对象，首先i5和i6进行自动拆箱操作，进行数值相加，即i4 == 40。然后Integer对象无法与数值进行直接比较，所以i4自动拆箱转为int值40，最终这条语句转为40 == 40进行数值比较。

###### String与常量池

```dart
String str1 = "abcd";
String str2 = new String("abcd");
System.out.println(str1==str2);//false
// 这是两段代码
String str1 = "str";
String str2 = "ing";
String str3 = "str" + "ing";
String str4 = str1 + str2;
System.out.println(str3 == str4);//false
  
String str5 = "string";
System.out.println(str3 == str5);//true
```

***解释：***

> 1. new String("abcd")是在常量池中拿对象，"abcd"是直接在堆内存空间创建一个新的对象。只要使用new方法，便需要创建新的对象。
> 2. 连接表达式 +
>    只有使用***引号包含文本的方式***创建的String对象之间使用“+”连接产生的新对象才会被加入字符串池中。上述代码中的str3
>    对于所有包含new方式新建对象（包括null）的“+”连接表达式，它所产生的新对象都不会被加入字符串池中。

```tsx
public static final String A; // 常量A
public static final String B;    // 常量B
static {  
   A = "ab";  
   B = "cd";  
}  
public static void main(String[] args) {  
	// 将两个常量用+连接对s进行初始化  
  String s = A + B;  
  String t = "abcd";  
  if (s == t) {  
      System.out.println("s等于t，它们是同一个对象");  
    } else {  
      System.out.println("s不等于t，它们不是同一个对象");  
    }  
}
```

***解释：***

> s不等于t，它们不是同一个对象。
>
> A和B虽然被定义为常量，但是它们都没有马上被赋值。在运算出s的值之前，他们何时被赋值，以及被赋予什么样的值，都是个变数。因此A和B在被赋值之前，性质类似于一个变量。***那么s就不能在编译期被确定，而只能在运行时被创建了***。

```dart
String s1 = new String("xyz"); //创建了几个对象？
```

***解释：***

> 2个
>
> 考虑类加载阶段和实际执行时。
>
> 1. 类加载对一个类只会进行一次。”xyz”在类加载时就已经创建并驻留了（如果该类被加载之前已经有”xyz”字符串被驻留过则不需要重复创建用于驻留的”xyz”实例）。驻留的字符串是放在全局共享的字符串常量池中的。
> 2. 在这段代码后续被运行的时候，”xyz”字面量对应的String实例已经固定了，不会再被重复创建。所以这段代码将常量池中的对象复制一份放到heap中，并且把heap中的这个对象的引用交给s1 持有。

```tsx
public static void main(String[] args) {
  String s1 = new String("计算机");
  String s2 = s1.intern();
  String s3 = "计算机";
  System.out.println("s1 == s2? " + (s1 == s2)); // false
  System.out.println("s3 == s2? " + (s3 == s2)); // true
}
```

***解释：***

> String的`intern()`方法会查找在常量池中是否存在一份equal相等的字符串,如果有则返回该字符串的引用,如果没有则添加自己的字符串进入常量池。

```csharp
public class Test {
  public static void main(String[] args) {
   	String hello = "Hello"
    String lo = "lo";
  	System.out.println((hello == "Hello") + " "); //true
   	System.out.println((Other.hello == hello) + " "); //true
   	System.out.println((other.Other.hello == hello) + " "); //true
   	System.out.println((hello == ("Hel"+"lo")) + " "); //true
   	System.out.println((hello == ("Hel"+lo)) + " "); //false
   	System.out.println(hello == ("Hel"+lo).intern()); //true
	}
}
 
class Other {
 static String hello = "Hello";
}
 
package other;
public class Other {
 public static String hello = "Hello";
} 
```

***解释：***

> 在同包同类下,引用自同一String对象.
>
> 在同包不同类下,引用自同一String对象.
>
> 在不同包不同类下,依然引用自同一String对象.
>
> 在编译成.class时能够识别为同一字符串的,自动优化成常量,引用自同一String对象.
>
> 在运行时创建的字符串具有独立的内存地址,所以不引用自同一String对象.

### 3.2 堆

Heap（堆）是JVM的内存数据区。

一个虚拟机实例只对应一个堆空间，堆是线程共享的。堆空间是存放对象实例的地方，几乎所有对象实例都在这里分配。堆也是垃圾收集器管理的主要区域(也被称为GC堆)。堆可以处于物理上不连续的内存空间中，只要逻辑上相连就行。

Heap 的管理很复杂，每次分配不定长的内存空间，专门用来保存对象的实例。在Heap 中分配一定的内存来保存对象实例，实际上也只是保存对象实例的属性值，属性的类型和对象本身的类型标记等，并不保存对象的方法（方法是指令，保存在Stack中）。而对象实例在Heap中分配好以后，需要在Stack中保存一个4字节的Heap 内存地址，用来定位该对象实例在Heap 中的位置，便于找到该对象实例。

> ***OOM:***
>
> 堆中没有足够的内存进行对象实例分配时，并且堆也无法扩展时，会抛出OutOfMemoryError异常。

![young&old](/Users/sherlock/Desktop/notes/allPics/Java/young&old.png)

### 3.3 栈(Java栈)

Stack（栈）是JVM的内存指令区。

描述的是java方法执行的内存模型：每个方法被执行的时候都会同时创建一个栈帧，用于存放***局部变量表（基本类型、对象引用）、操作数栈、方法返回、常量池指针等信息***。 由编译器自动分配释放， 内存的分配是连续的。Stack的速度很快，管理很简单，并且每次操作的数据或者指令字节长度是已知的。所以Java 基本数据类型，Java 指令代码，常量都保存在Stack中。

虚拟机只会对栈进行两种操作，以帧为单位的入栈和出栈。Java栈中的每个帧都保存一个方法调用的局部变量、操作数栈、指向常量池的指针等，且每一次方法调用都会创建一个帧，并压栈。

> ***OOM***
>
> - 如果一个线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常， 比如递归调用。
> - 如果线程生成数量过多，无法申请足够多的内存时，则会抛出OutOfMemoryError异常。比如tomcat请求数量非常多时，设置最大请求数。

#### 3.3.1 栈帧

栈帧由三部分组成:

1. 局部变量区
2. 操作数栈
3. 帧数据区(指向常量池的一堆指针)

##### 3.3.1.1 局部变量区

包含方法的参数和局部变量。

###### Code

```csharp
package com.github.java.jvm.stack;

/**
 * Descriptor:
 * Author: sherlock
 */
public class StackTest {

    public static void main(String[] args) {
        StackTest s = new StackTest();
        s.method1();
    }
    void method1(){
        System.out.println("method1 start......");
        try {
            method2();
        } catch (Exception e){
            e.printStackTrace();
        }
        System.out.println("method1 end......");
    }
    void method2(){
        System.out.println("method2 start......");
        method3();
        System.out.println("method2 end......");
        System.out.println(10 / 0);
    }
    void method3(){
        System.out.println("method3 start......");
        System.out.println("method3 end......");
    }

}
```

IDEA使用***jclasslib***插件之后可以查看字节码指令如下：

![jclasslib1](/Users/sherlock/Desktop/notes/allPics/Java/jclasslib1.png)

接下来我们一一解释这里面的内容

1. ***Code***

   > ***Misc:***
   >
   > - Maximun local variables: 局部变量表的长度
   > - Code length: 编译后字节码指令的长度
   >
   > ***Bytecode:***
   >
   > - 图中红色数字为字节码指令索引也就是PC寄存器执行顺序的索引
   >
   > 
   >
   > ![jclasslib2](/Users/sherlock/Desktop/notes/allPics/Java/jclasslib2.png)
   >
   > ![jclasslib3](/Users/sherlock/Desktop/notes/allPics/Java/jclasslib3.png)

2. ***LineNumTable***和***LocalVariableTable***

   >介绍完***code***，再继续介绍下面两个内容,这两个内容要结合在一起理解：
   >
   >***LineNumTable***:
   >
   >- Start PC: 字节码指令的索引, PC寄存器的索引
   >- Line Number: 代码中的行号
   >
   >***LocalVariableTable***:
   >
   >- Start PC: 字节码指令的索引, PC寄存器的索引
   >- Length: 在整个字节码指令中的作用范围
   >
   >![jclasslib4](/Users/sherlock/Desktop/notes/allPics/Java/jclasslib4.png)
   >
   >![jclasslib5](/Users/sherlock/Desktop/notes/allPics/Java/jclasslib5.png)
   >
   >***<u>TIPS:</u>***
   >
   >​	***Misc***属性中的***Code Length*** = ***LocalVariableTable***中的***Length***

###### Slot

Java栈帧里的局部变量表有很多的槽位组成，每个槽最大可以容纳32位的数据类型, 

| 数据类型  | 占用slot数量 |
| --------- | ------------ |
| byte      | 1            |
| boolean   | 1            |
| byte      | 1            |
| short     | 1            |
| int       | 1            |
| float     | 1            |
| long      | 2            |
| double    | 2            |
| reference | 1            |

1. 在编译好了的字节码指令中, 静态方法的参数索引都是从0开始的, 每个类型的局部变量占用slot数如上

   > ```java
   > public static int method(int i, long l, float f, Object o, byte b){
   > 	double d = 0.9;
   >   String s = "miao";
   >   return 0;
   > }
   > ```
   >
   > 各局部变量所占用slot数量如下:
   >
   > ```csharp
   > 0 int int i							---> 占1个slot
   > 1 long long l						---> 占2个slot
   > 3 float float f					---> 占1个slot
   > 4 reference Object o		---> 占1个slot
   > 5 int byte b						---> 占1个slot
   > 6 double double d				---> 占2个slot
   > 8 reference String s		---> 占1个slot
   > ```

2. 实例方法的局部变量表和静态方法唯一区别就是实例方法在局部变量表里第一个槽位（0位置）存的是一个this引用（当前对象的引用），后面就和静态方法的一样了。

   >```java
   >public int method(int i, long l, float f, Object o, byte b){
   >	double d = 0.9;
   >  String s = "miao";
   >  return 0;
   >}
   >```
   >
   >这个方法是非static方法, 各局部变量占用slot数量如下:
   >
   >```java
   >0 reference StackTest this	---> 占1个slot
   >1 int int i									---> 占1个slot
   >2 long long l								---> 占2个slot
   >4 float float f							---> 占1个slot
   >5 reference Object o				---> 占1个slot
   >6 int byte b								---> 占1个slot
   >7 double double d						---> 占2个slot
   >9 reference String s				---> 占1个slot
   >```

3. slot槽位是可以复用的

   > 当一个局部变量过了其作用域，那么在其作用域之后申明的新的局部变量就很可能会复用过期局部变量的槽位，从而达到节省资源的目的
   >
   > ```java
   > void method2() {
   >   int a = 0;
   >   {
   >     int b = 0;
   >   	b = a + 1;
   >   }
   >   // 上面代码块执行结束, 局部变量b已经被销毁, 就有slot空出来
   >   int c = a + 1;
   > }
   > ```
   >
   > ![jclasslib6](/Users/sherlock/Desktop/notes/allPics/Java/jclasslib6.png)
   >
   > 上图中b开始位于index=2的slot中, 当局部变量c出现后, 意味着代码块执行结束了, 变量b被销毁, c复用了b的slot

   

##### 3.3.1.2 操作数栈

操作数栈也常被称为操作栈，它是一个后入先出（Last In First Out，LIFO) 栈。同局部变量表一样，操作数栈的最大深度也在编译的时候被写入到Code属性的max_stacks数据项之中。操作数栈的每一个元素可以是任意的Java数据类型，包括long和double。32位数据类型所占的栈容量为l,64位数据类型所占的栈容量为2在方法执行的任何时候，操作数栈的深度都不会超过在max_stacks数据项中设定的最大值。
