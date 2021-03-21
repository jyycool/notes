## 【JVM】对象在内存中的布局

### 前言

引用openjdk中的jol-core.jar包

```xml
<dependency>
  <groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version>0.9</version>
</dependency>
```

### 当前JVM环境参数

```sh
sherlock-MBP:~ sherlock$ java -XX:+PrintCommandLineFlags -version
-XX:InitialHeapSize=134217728 -XX:MaxHeapSize=2147483648 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC
java version "1.8.0_161"
Java(TM) SE Runtime Environment (build 1.8.0_161-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.161-b12, mixed mode)
```

开启UseCompressedOops，默认会开启UseCompressedClassPointers，会压缩klass pointer 这部分的大小，由8字节压缩至4字节，间接的提高内存的利用率。

由于UseCompressedClassPointers的开启是依赖于UseCompressedOops的开启，因此，要使UseCompressedClassPointers起作用，得先开启UseCompressedOops，并且开启UseCompressedOops 也默认开启UseCompressedOops，关闭UseCompressedOops 默认关闭UseCompressedOops。

### 示例

#### 示例1

```java
package com.github.java.object;

import org.openjdk.jol.info.ClassLayout;

/**
 * Descriptor:
 * Author: sherlock
 */
public class ObjectSize {

    public static void main(String[] args) {
        Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
    }
}
```

运行打印结果:

```java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

> 3个object header的SIZE是4, 分别为mark word = 4 + 4 = 8byte, class pointer = 4byte, 对其填充 = 4byte, 所以对象在内存占用空间16byte



### 对象的内存布局

一个Java对象在内存中包括对象头、实例数据和补齐填充3个部分：

![对象内存布局](/Users/sherlock/Desktop/notes/allPics/Java/对象内存布局.png)

#### 对象头

- **Mark Word**：包含一系列的标记位，比如轻量级锁的标记位，偏向锁标记位等等。在32位系统占4字节，在64位系统中占8字节；
- **Class Pointer**：用来指向对象对应的Class对象（其对应的元数据对象）的内存地址。在32位系统占4字节，在64位系统中占8字节；
- **Length**：如果是数组对象，还有一个保存数组长度的空间，占4个字节；

#### 对象实际数据

对象实际数据包括了对象的所有成员变量，其大小由各个成员变量的大小决定，比如：byte和boolean是1个字节，short和char是2个字节，int和float是4个字节，long和double是8个字节，reference是4个字节（64位系统中是8个字节）。

| Primitive Type | Memory Required(bytes) |
| -------------- | ---------------------- |
| boolean        | 1                      |
| byte           | 1                      |
| short          | 2                      |
| char           | 2                      |
| int            | 4                      |
| float          | 4                      |
| long           | 8                      |
| double         | 8                      |

#### 对齐填充

Java对象占用空间是8字节对齐的，即所有Java对象占用bytes数必须是8的倍数。例如，一个包含两个属性的对象：int和byte，这个对象需要占用8+4+1=13个字节，这时就需要加上大小为3字节的padding进行8字节对齐，最终占用大小为16个字节。

注意：以上对64位操作系统的描述是未开启指针压缩的情况，关于指针压缩会在下文中介绍。

#### 对象头占用空间大小

这里说明一下32位系统和64位系统中对象所占用内存空间的大小：

- 在32位系统下，存放Class Pointer的空间大小是4字节，MarkWord是4字节，对象头为8字节;
- ***在64位系统下，存放Class Pointer的空间大小是8字节，MarkWord是8字节，对象头为16字节;***
- ***64位开启指针压缩的情况下，存放Class Pointer的空间大小是4字节，MarkWord是8字节，对象头为12字节***;
- ***数组对象的对象头的大小为：数组对象头8字节+数组长度4字节+对齐4字节=16字节。其中对象引用占4字节（未开启指针压缩的64位为8字节），数组MarkWord为4字节（64位未开启指针压缩的为8字节）***;
- 静态属性不算在对象大小内。

### 指针压缩

从上文的分析中可以看到，64位JVM消耗的内存会比32位的要多大约1.5倍，这是因为对象指针在64位JVM下有更宽的寻址。对于那些将要从32位平台移植到64位的应用来说，平白无辜多了1/2的内存占用，这是开发者不愿意看到的。

从JDK 1.6 update14开始，64位的JVM正式支持了 -XX:+UseCompressedOops 这个可以压缩指针，起到节约内存占用的新参数。

### 什么是OOP？

OOP的全称为：Ordinary Object Pointer，就是普通对象指针。启用CompressOops后，会压缩的对象：

- 每个Class的属性指针（静态成员变量）；
- 每个对象的属性指针；
- 普通对象数组的每个元素指针。

当然，压缩也不是所有的指针都会压缩，对一些特殊类型的指针，JVM是不会优化的，例如指向PermGen的Class对象指针、本地变量、堆栈元素、入参、返回值和NULL指针不会被压缩。

### 启用指针压缩

在Java程序启动时增加JVM参数：`-XX:+UseCompressedOops`来启用。

*注意：32位HotSpot VM是不支持UseCompressedOops参数的，只有64位HotSpot VM才支持。*

本文中使用的是JDK 1.8，默认该参数就是开启的。

### Shallow size和Retained size

Shallow size就是对象本身占用内存的大小，不包含其引用的对象。

- 常规对象（非数组）的Shallow size有其成员变量的数量和类型决定。
- 数组的shallow size有数组元素的类型（对象类型、基本类型）和数组长度决定

Retained size是该对象自己的shallow size，加上从该对象能直接或间接访问到对象的shallow size之和。换句话说，retained size是该对象被GC之后所能回收到内存的总和

### 查看对象的大小

接下来我们使用[http://www.javamex.com/](https://link.jianshu.com?t=http://www.javamex.com/)中提供的[classmexer.jar](https://link.jianshu.com?t=http://www.javamex.com/classmexer/classmexer-0_03.zip)来计算对象的大小。

运行环境：JDK 1.8，Java HotSpot(TM) 64-Bit Server VM

#### 基本数据类型

对于基本数据类型来说，是比较简单的，因为我们已经知道每个基本数据类型的大小。代码如下：

```java
import com.javamex.classmexer.MemoryUtil;

/**
 * Descriptor:
 * Author: sherlock
 * mvn install:install-file -DgroupId=classmexer -DartifactId=classmexer -Dversion=0.03 -Dpackaging=jar -Dfile=/Users/sherlock/devTools/sher/classmexer-0_03/classmexer.jar
 *
 * VM options:
 * -javaagent:/Users/sherlock/devTools/sher/classmexer-0_03/classmexer.jar
 * -XX:+UseCompressedOops
 */
public class ObjectSize2 {

    int a;
    long b;
    static int c;

    public static void main(String[] args) {

        ObjectSize2 objectSize2 = new ObjectSize2();
        // 打印对象的shallow size
        System.out.println("Shallow Size: " + MemoryUtil.memoryUsageOf(objectSize2) + " bytes");
        // 打印对象的 retained size
        System.out.println("Retained Size: " + MemoryUtil.deepMemoryUsageOf(objectSize2) + " bytes");

    }
}
```

运行查看结果：

```xml
Shallow Size: 24 bytes
Retained Size: 24 bytes
```

根据上文的分析可以知道，64位开启指针压缩的情况下：

- 对象头大小 = `ClassPointer(4byte)` + `MarkWord(8byte)` = 12 byte；
- 实际数据大小 = `int(4byte)` + `long(8byte)` = 12 byte（静态变量不在计算范围之内）

这里计算后大小为12 + 12 = 24 byte, 不需要padding补齐

#### 数组类型

```java
package com.github.java.object;

import com.javamex.classmexer.MemoryUtil;
import org.openjdk.jol.info.ClassLayout;

/**
 * Descriptor:
 * Author: sherlock
 * mvn install:install-file -DgroupId=classmexer -DartifactId=classmexer -Dversion=0.03 -Dpackaging=jar -Dfile=/Users/sherlock/devTools/sher/classmexer-0_03/classmexer.jar
 *
 * VM options:
 * -javaagent:/Users/sherlock/devTools/sher/classmexer-0_03/classmexer.jar
 * -XX:+UseCompressedOops
 */
public class ObjectSize2 {

    int a;
//    long b;
//    static int c;
    char[] arr = new char[50];

    public static void main(String[] args) {
        ObjectSize2 objectSize2 = new ObjectSize2();  		System.out.println(ClassLayout.parseInstance(objectSize2).toPrintable());

        // 打印对象的shallow size
        System.out.println("Shallow Size: " + MemoryUtil.memoryUsageOf(objectSize2) + " bytes");
        // 打印对象的 retained size
        System.out.println("Retained Size: " + MemoryUtil.deepMemoryUsageOf(objectSize2) + " bytes");

    }
}
```

运行打印结果:

```java
com.github.java.object.ObjectSize2 object internals:
 OFFSET  SIZE     TYPE DESCRIPTION                               VALUE
      0     4          (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4          (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4          (object header)                           61 c1 00 f8 (01100001 11000001 00000000 11111000) (-134168223)
     12     4      int ObjectSize2.a                             0
     16     4   char[] ObjectSize2.arr                           [, , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , , ]
     20     4          (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

Shallow Size: 24 bytes
Retained Size: 144 bytes
```

> 因为该类中有一个数组属性: 经过测试无论是int[], long[], char[], ***objectSize2.arr***的引用大小一直是4byte, 所以这个objectSize2对象本身的Shallow Size = mark word(8byte) + class pointer(4byte) + 初始数据大小(int4byte + 数组引用4byte = 8byte) + padding(4byte) = 24byte.
>
> 而 objectSize2.arr数组本身的Shallow Size = mark word(8byte) + class pointer(4byte) + length(4byte) + 实际数据大小(50 * 2byte) + 填充 0byte= 116byte
>
> objectSize2的Retained Size = objectSize2的Shallow Size + arr的Shallow Size  + padding = 116 + 24 + 4 = 144byte

#### 包装类型

包装类（Boolean/Byte/Short/Character/Integer/Long/Double/Float）占用内存的大小等于对象头大小加上底层基础数据类型的大小。

包装类型的Retained Size占用情况如下：

| Numberic Wrappers | +useCompressedOops | -useCompressedOops |
| ----------------- | ------------------ | ------------------ |
| Byte, Boolean     | 16 bytes           | 24 bytes           |
| Short, Character  | 16 bytes           | 24 bytes           |
| Integer, Float    | 16 bytes           | 24 bytes           |
| Long, Double      | 24 bytes           | 24 bytes           |

这里不做举例说明

#### String类型

***重头戏来了!!!!!!!***

在JDK1.7及以上版本中，`java.lang.String`中包含2个属性，一个用于存放字符串数据的char[], 一个int类型的hashcode, 部分源代码如下：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0
    ...
}
```

在开启指针压缩时，一个String对象的大小为：

- **Shallow Size=对象头大小12字节+int类型大小4字节+数组引用大小4字节+padding4字节=24字节**；
- **Retained Size=Shallow Size+char数组的Retained Size**。

##### 示例:

```java
package com.github.java.object;

import com.javamex.classmexer.MemoryUtil;
import org.openjdk.jol.info.ClassLayout;

/**
 * Descriptor:
 * Author: sherlock
 * mvn install:install-file -DgroupId=classmexer -DartifactId=classmexer -Dversion=0.03 -Dpackaging=jar -Dfile=/Users/sherlock/devTools/sher/classmexer-0_03/classmexer.jar
 *
 * VM options:
 * -javaagent:/Users/sherlock/devTools/sher/classmexer-0_03/classmexer.jar
 * -XX:+UseCompressedOops
 */
public class ObjectSize2 {

//    int a;
//    long b;
//    static int c;
//    char[] arr = new char[50];
    String name = "miao";


    public static void main(String[] args) {

        ObjectSize2 objectSize2 = new ObjectSize2();
        System.out.println(ClassLayout.parseInstance(objectSize2).toPrintable());

        // 打印对象的shallow size
        System.out.println("Shallow Size: " + MemoryUtil.memoryUsageOf(objectSize2) + " bytes");
        // 打印对象的 retained size
        System.out.println("Retained Size: " + MemoryUtil.deepMemoryUsageOf(objectSize2) + " bytes");

    }
}
```

运行打印结果;

```jade
com.github.java.object.ObjectSize2 object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           61 c1 00 f8 (01100001 11000001 00000000 11111000) (-134168223)
     12     4   java.lang.String ObjectSize2.name                          (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

Shallow Size: 16 bytes
Retained Size: 64 bytes
```

> 1. 首先objectSize2的Shallow Size = (12 + 4 +0) = 16byte
> 2. String也是一个对象, 它的Shallow Size = (12 + 4 + 4) = 24byte
> 3. String中有一个char[]数组, 它的Shallow Size = (12 + 4 + 2*4 + 0) = 24byte
>
> 所以objectSize2的**Retained Size** = 16 + 24 + 24 = 64byte

