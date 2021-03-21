## 【JNI】数据类型和类型描述符

我们使用 javap, jmap 查看 java class 时,经常看到一些类似 `[B、[C、 [Ljava/lang/Object、 (ILjava/lang/Class;)J` ... 这样的符号, 初看时真的是一脸懵... 这些其实就是 JNI(Java Native Interface) 描述符

这里包含了两种:

1. JNI字段描述符
2. JNI方法描述符

## 一、简介

在 JNI 开发中，我们知道，Java 的数据类型并不是直接在 JNI 里使用的，例如 int 就是使用 jint 来表示。so，来认识一下 JNI 中的这些数据类型吧。

## 二、基本数据类型

| Java数据类型 | JNI本地类型 | C/C++数据类型  | 数据类型描述        |
| ------------ | ----------- | -------------- | ------------------- |
| boolean      | jboolean    | unsigned char  | C/C++无符号8为整数  |
| byte         | jbyte       | signed char    | C/C++有符号8位整数  |
| char         | jchar       | unsigned short | C/C++无符号16位整数 |
| short        | jshort      | signed short   | C/C++有符号16位整数 |
| int          | jint        | signed int     | C/C++有符号32位整数 |
| long         | jlong       | signed long    | C/C++有符号64位整数 |
| float        | jfloat      | float          | C/C++32位浮点数     |
| double       | jdouble     | double         | C/C++64位浮点数     |



## 三、引用数据类型

| Java的类类型        | JNI的引用类型 | 类型描述                                                     |
| ------------------- | ------------- | ------------------------------------------------------------ |
| java.lang.Object    | jobject       | 可以表示任何Java的对象，或者没有JNI对应类型的Java对象（*实例方法的强制参数*） |
| java.lang.String    | jstring       | Java的String字符串类型的对象                                 |
| java.lang.Class     | jclass        | Java的Class类型对象（*静态方法的强制参*数）                  |
| Object[]            | jobjectArray  | Java任何对象的数组表示形式                                   |
| boolean[]           | jbooleanArray | Java基本类型boolean的数组表示形式                            |
| byte[]              | jbyteArray    | Java基本类型byte的数组表示形式                               |
| char[]              | jcharArray    | Java基本类型char的数组表示形式                               |
| short[]             | jshortArray   | Java基本类型short的数组表示形式                              |
| int[]               | jintArray     | Java基本类型int的数组表示形式                                |
| long[]              | jlongArray    | Java基本类型long的数组表示形式                               |
| float[]             | jfloatArray   | Java基本类型float的数组表示形式                              |
| double[]            | jdoubleArray  | Java基本类型double的数组表示形式                             |
| java.lang.Throwable | jthrowable    | Java的Throwable类型，表示异常的所有类型和子类                |
| void                | void          | N/A                                                          |



## 四、数据类型描述符

### 4.1 什么是数据类型描述符

JVM虚拟机中, 存储数据类型名称时, 使用的是指定的描述符来存储, 而不是我们习惯的 int, long 等。

### 4.2 对照表

| Java类型     | 类型描述符                       |
| ------------ | -------------------------------- |
| int          | I                                |
| long         | J                                |
| byte         | B                                |
| short        | S                                |
| char         | C                                |
| float        | F                                |
| double       | D                                |
| boolean      | Z                                |
| void         | V                                |
| 其他引用类型 | L+类全名+; (这里注意一定要;结尾) |
| 数组         | [                                |
| 方法         | (参数)返回值                     |

### 4.3 示例

1. 表示一个 String 类

   一个 Java 类对应的描述符，就是 "L" 加 "全类名"，其中 "." 要换成 "/" ，最后末尾";"结束。

   ```java
   Java 类型：java.lang.String
   JNI 描述符：Ljava/lang/String;
   ```

2. 表示数组

   数组就是简单的在类型描述符前加 [ 即可，二维数组就是两个 [ ，以此类推。

   ```java
   Java 类型：String[]
   JNI 描述符：[Ljava/lang/String;
   
   Java 类型：int[][]
   JNI 描述符：[[I
   ```

3. 表示方法

   括号内是每个参数的类型符，括号外就是返回值的类型符。

   ```js
   Java 方法：long f (int n, String s, int[] arr);
   JNI 描述符：(ILjava/lang/String;[I)J
   
   Java 方法：void f ();
   JNI 描述符：()V
   ```

   





