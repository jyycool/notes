# 【SCALA】默认导入和包的声明以及导入

## 前言

Scala默认会为每个`.scala`文件导入以下3个包:

1. `java.lang._`
2. `scala._`
3. `scala.Predef._` (一般很多的隐式转换都在该包下)

## 包的声明

1. 支持和`java`一样的声明方式(基本这种使用)

   ```java
   package com.zhengkw.scala.day04.pack
   1
   ```

2. 支持多个`package`语句(很少碰到)

   ```java
   package com.zhengkw.scala.day04.pack
   package a.b
   12
   ```

   `com.zhengkw.scala.day04.pack.a.b.PackDemo`

3. 包语句(很少碰到)

   ```java
   package c{  // c其实是子包
       class A
   }
   123
   ```

## 包导入

1. 导入和`java`一样, 在文件最顶层导入, 整个文件的任何位置都可以使用(掌握)

   ```java
   import java.util.HashMap
   1
   ```

2. **在`scala`中其实在代码任何位置都可以导入.**

   ```java
   def main(args: Array[String]): Unit = {
       
       import java.io.FileInputStream
       // 只能在main函数中使用
       val is = new FileInputStream("c:/users.json")
   }
   123456
   ```

3. 导入类的时候, 防止和现有的冲突, 可以给类起别名 （FileInputStream别名为JFI）

   ```java
   import java.io.{FileInputStream => JFI}
   1
   ```

4. **如何批量导入**

```java
import java.io._  // 导入java.io包下所有的类   (java是*)
1
```

5. **屏蔽某个类**

```java
import java.io.{FileInputStream => _, _}  //屏蔽 FileInputStream 
1
```

6. 静态导入   

> `java`只能导入静态成员.

```java
import static java.lang.System.out;
out.println();
12
scala
```

> `scala`中没有静态的概念！

```java
   import java.lang.Math._	
1
```

7. **`scala`还支持导入对象的成员**

```java
   val u = new User
   // 把对象u的成员导入
   import u._
   foo()
   eat()
12345
```

## 公共方法的处理

`java`中一般搞工具类, 在工具类中写静态方法. 因为`java`中所有的方法都需要依附于类或者对象

`scala`中为了解决这个问题, 提供了一个**包对象**的, 将来在这个包内, 使用包对内的方法的时候, 就像使用自己定义的.

```java
package com.github.sherlock
//包对象
package object pack {
    def foo1() = {
        println("foo...")
    }
    def eat1() = {
        println("eat...")
    }
}
12345678910
```

在`com.github.sherlock`包下所有的类可以直接使用这些方法.

