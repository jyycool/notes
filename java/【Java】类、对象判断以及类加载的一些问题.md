# 【Java】类、对象判断以及类加载的一些问题

在学习<<深入理解 Java虚拟机>>时了解到, 如果一个类被不同的 `ClassLoader` 加载到内存, 那么生成的 `.class` 对象是不同的;

下面是测试代码 (`Foo.class` 是一个无成员变量和成员方法的普通类)

```java
package cgs.guice.test;

import java.io.IOException;
import java.io.InputStream;

/**
 * @Description TODO
 * @Author sherlock
 * @Date
 */
public class ClassLoaderTest {

	static final String className = "cgs.guice.test.Foo";

  public static void main(String[] args) throws Exception 	{

  	ClassLoader myLoader = new ClassLoader(){
    	@Override
      public Class<?> loadClass(String name) throws ClassNotFoundException {
      	try {
        	String className = name.substring(name.lastIndexOf(".") + 1);
          String classFileName = className + ".class";
          InputStream inputStream = getClass()
            		.getResourceAsStream(classFileName);
          if (inputStream == null) 
          	return super.loadClass(name);
          byte[] bytes = new byte[inputStream.available()];
          inputStream.read(bytes);
          return defineClass(name, bytes, 0, bytes.length);
				}catch (IOException e){
					throw new ClassNotFoundException(name);
        }
     }
	};

		Object instance = myLoader.loadClass(className).newInstance();
		System.out.println(instance.getClass());
		System.out.println(instance instanceof Foo);
	}
}
```

结果自然为 `false`, 因为 instace 对象是使用自定构造器加载的类生成的实例, 它当然不是 使用默认加载器加载的 Foo 的实例

于是在这里牵扯出了 `instanceof` 方法, 又联想到 `isInstance` 以及 `isAssignableFrom`

## 1. instanceof

`instanceof` 严格来说是 Java 中的一个双目运算符，用来测试一个对象是否为一个类的实例，用法为：

```java
boolean result = obj instanceof Class
```

　　其中 obj 为一个对象，Class 表示一个类或者一个接口，当 obj 为 Class 的对象，或者是其直接或间接子类，或者是其接口的实现类，结果result 都返回 true，否则返回false。

　　注意：编译器会检查 obj 是否能转换成右边的class类型，如果不能转换则直接报错，如果不能确定类型，则通过编译，具体看运行时定。

### 1.1 obj 必须为引用类型，不能是基本类型

```java
int i = 0;
System.out.println(i instanceof Integer); //编译不通过System.out.println(i instanceof Object);//编译不通过
```

`instanceof` 运算符只能用作对象的判断。

### 1.2 obj 为 null

```java
System.out.println(null instanceof Object);//false
```

关于 `null` 类型的描述在 [官方文档](https://docs.oracle.com/javase/specs/jls/se7/html/jls-4.html#jls-4.1) 有一些介绍。一般我们知道 Java 分为两种数据类型，一种是基本数据类型，有八个分别是 `byte` `short` `int` long `float` double `char` `boolean`,一种是引用类型，包括类，接口，数组等等。而Java中还有一种特殊的 `null` 类型，该类型没有名字，所以不可能声明为 `null` 类型的变量或者转换为 `null` 类型，`null` 引用是 `null` 类型表达式唯一可能的值，`null` 引用也可以转换为任意引用类型。我们不需要对 null 类型有多深刻的了解，我们只需要知道 `null` 是可以成为任意引用类型的**特殊符号**。

在 [JavaSE规范](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.20.2) 中对 `instanceof` 运算符的规定就是：如果 obj 为 `null`，那么将返回 `false`。

### 1.3 obj 为 class 类的实例对象

```java
Integer integer = new Integer(1);
System.out.println(integer instanceof Integer);//true
```

这没什么好说的，最普遍的一种用法。

### 1.4 obj 为 class 接口的实现类

了解 Java 集合的，我们知道集合中有个上层接口 `List`，其有个典型实现类 `ArrayList`

```java
public class ArrayList<E> extends AbstractList<E>  implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

所以我们可以用 `instanceof` 运算符判断 某个对象是否是 `List` 接口的实现类，如果是返回 `true`，否则返回 `false`

```java
ArrayList arrayList = new ArrayList();
System.out.println(arrayList instanceof List); //true
```

或者反过来也是返回 true

```java
List list = new ArrayList();
System.out.println(list instanceof ArrayList);//true
```

### 1.5 obj 为 class 类的直接或间接子类

我们新建一个父类 `Person.class`，然后在创建它的一个子类 `Man.class`

```java
public class Person { }
```

`Man.class`

```java
public class Man extends Person{ }
```

测试：

```java
Person p1 = new Person();
Person p2 = new Man();
Man m1 = new Man();
System.out.println(p1 instanceof Man); //false
System.out.println(p2 instanceof Man); //true
System.out.println(m1 instanceof Man); //true
```

注意第一种情况， `p1 instanceof Man` ，Man 是 Person 的子类，Person 不是 Man 的子类，所以返回结果为 false。

### 1.6 问题

前面我们说过**编译器会检查 obj 是否能转换成右边的class类型，如果不能转换则直接报错，如果不能确定类型，则通过编译，具体看运行时定。**

看如下几个例子：

```java
Person p1 = new Person();
System.out.println(p1 instanceof String);//编译报错System.out.println(p1 instanceof List);	//false
System.out.println(p1 instanceof List<?>);//false
System.out.println(p1 instanceof List<Person>);//编译报错
```

按照我们上面的说法，这里就存在问题了，Person 的对象 p1 很明显不能转换为 String 对象，那么自然 Person 的对象 `p1 instanceof String` 不能通过编译，但为什么 `p1 instanceof List` 却能通过编译呢？而 `instanceof List<Person>` 又不能通过编译了？

### 1.7 深究原理

我们可以看Java语言规范 [Java SE 8](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.20.2)

![img](https://images2018.cnblogs.com/blog/1120165/201803/1120165-20180302000448613-26394231.png)

如果用伪代码描述：

```java
boolean result;
if (obj == null) {
	result = false;
} else {
	try{
		T temp = (T) obj; // checkcast 
		result = true;
	} catch (ClassCastException e) {
		result = false;
	}
}
```

也就是说有表达式 `obj instanceof T`，`instanceof` 运算符的 obj 操作数的类型必须是引用类型或空类型; 否则，会发生编译时错误。 

如果 obj 强制转换为 T 时发生编译错误，则关系表达式的 instanceof 同样会产生编译时错误。 在这种情况下，表达式实例的结果永远为false。

在运行时，如果 T 的值不为 `null`，并且 obj 可以转换为 T 而不引发`ClassCastException`，则 `instanceof` 运算符的结果为true。 否则结果是错误的

简单来说就是：**如果 obj 不为 null 并且 (T) obj 不抛 ClassCastException 异常则该表达式值为 true ，否则值为 false 。**

所以对于上面提出的问题就很好理解了，为什么 p1 instanceof String 编译报错，因为 `(String)p1` 是不能通过编译的，而 `(List)p1` 可以通过编译。

### 1.8 instanceof 的实现策略

JavaSE 8 `instanceof` 的[实现算法](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.instanceof)

　　![img](https://images2018.cnblogs.com/blog/1120165/201803/1120165-20180302002919162-2045599504.png)

 

1. obj 如果为 null，则返回 false；否则设S 为 obj 的类型对象，剩下的问题就是检查S是否为T的子类型；

2. 如果S == T，则返回 true；

3. 接下来分为3种情况，之所以要分情况是因为instanceof要做的是“子类型检查”，而Java语言的类型系统里数组类型、接口类型与普通类类型三者的子类型规定都不一样，必须分开来讨论。

   - S是数组类型：如果 T 是一个类类型，那么T必须是Object；如果 T 是接口类型，那么 T 必须是由数组实现的接口之一；
   - 接口类型：对接口类型的 instanceof 就直接遍历S里记录的它所实现的接口，看有没有跟T一致的；

   - 类类型：对类类型的 instanceof 则是遍历S的super链（继承链）一直到Object，看有没有跟T一致的。遍历类的super链意味着这个算法的性能会受类的继承深度的影响。



## 2. isInstance

这个方法是属于 `Class` 类的方法

```java
class.isInstance(obj)
```

obj 是被测试的对象，如果 obj 是调用这个方法的 class或接口 的实例，则返回 true。这个方法是 `instanceof` 运算符的动态等价。 

它的属性及使用情况和 `instanceof` 完全一样



## 3. isAssignableFrom

这个方法也是属于 `Class` 类的方法

```java
class.iaAssignableFrom(clazz)
```

如果调用这个方法的 `class` 或 接口 与参数 `clazz` 表示的类或接口相同，或者是参数 `clazz` 表示的类或接口的父类，则返回 true。 

简单些说就是当 `class` 与 clazz 相同 或者 `class` 是 `clazz` 的父类/父接口, 则返回 true

> 但是这里要注意不同 classloader 加载的情况



## 4.Scala 中对应的方法

一张表就能说明

| **Scala**           | **Java**         |
| ------------------- | ---------------- |
| obj.isInstanceOf[C] | obj instanceof C |
| obj.asInstanceOf[C] | (C)obj           |
| classOf[C]          | C.class          |