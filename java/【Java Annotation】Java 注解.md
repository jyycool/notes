# 【Java Annotation】Java 注解



Java1.5引入了注解，当前许多java框架中大量使用注解，如Hibernate、Jersey、Spring。注解作为程序的元数据嵌入到程序当中。注解可以被一些解析工具或者是编译工具进行解析。我们也可以声明注解在编译过程或执行时产生作用。

### 创建Java自定义注解

创建自定义注解和创建一个接口相似，但是注解的interface关键字需要以@符号开头。我们可以为注解声明方法。我们先来看看注解的例子，然后我们将讨论他的一些特性。

```java
package cgs.java.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

 
@Documented
@Target(ElementType.METHOD)
@Inherited
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodInfo{
  String author() default 'Pankaj';
  String date();
  int revision() default 1;
  String comments();
}
```

- 注解方法不能带有参数；
- 注解方法返回值类型限定为：基本类型、String、Enums、Annotation或者是这些类型的数组；
-  注解方法可以有默认值；
-  注解本身能够包含元注解，元注解被用来注解其它注解。



### 元注解

1. **@Documented** 

   指明拥有这个注解的元素可以被 javadoc 此类的工具文档化。这种类型应该用于注解那些影响客户使用带注释的元素声明的类型。如果一种声明使用 **Documented** 进行注解，这种类型的注解被作为被标注的程序成员的公共API。

2. **@Target** 

   指明该类型的注解可以注解的程序元素的范围。该元注解的取值可以为 ***TYPE, METHOD, CONSTRUCTOR, FIELD*** 等。***如果 Target 元注解没有出现，那么定义的注解可以应用于程序的任何元素。***

   ElementType.TYPE     							接口、类、枚举、注解

   ElementType.FIELD    							字段、枚举的常量

   ElementType.METHOD 				 		方法

   ElementType.PARAMETER      	 		方法参数

   ElementType.CONSTRUCTOR 	 		构造函数

   ElementType.LOCAL_VARIABLE  		局部变量

   ElementType.ANNOTATION_TYPE 	 注解

   ElementType.PACKAGE      				   包

3. **@Inherited** 

   指明该注解类型被自动继承。如果用户在当前类中查询这个元注解类型并且当前类的声明中不包含这个元注解类型，那么也将自动查询当前类的父类是否存在**Inherited**元注解，这个动作将被重复执行知道这个标注类型被找到，或者是查询到顶层的父类。

4. **@Retention** 

   指明了该Annotation被保留的时间长短。RetentionPolicy取值为***SOURCE, CLASS, RUNTIME***

   ***RetentionPolicy.SOURCE*** 注解仅存在于源码中，在class字节码文件中不包含

   ***RetentionPolicy.CLASS*** 默认的保留策略，注解会在class字节码文件中存在，但运行时无法获得

   ***RetentionPolicy.RUNTIME*** 注解会在class字节码文件中存在，在运行时可以通过反射获取到

　　

### Java内建注解

Java提供了三种内建注解。

1. **@Override**

   当我们想要复写父类中的方法时，我们需要使用该注解去告知编译器我们想要复写这个方法。这样一来当父类中的方法移除或者发生更改时编译器将提示错误信息。

2. **@Deprecated**

   当我们希望编译器知道某一方法不建议使用时，我们应该使用这个注解。Java在javadoc 中推荐使用该注解，我们应该提供为什么该方法不推荐使用以及替代的方法。

3. **@SuppressWarnings**

   这个仅仅是告诉编译器忽略特定的警告信息，例如在泛型中使用原生数据类型。它的保留策略是SOURCE（译者注：在源文件中有效）并且被编译器丢弃。



### Java注解解析

我们将使用反射技术来解析java类的注解。那么注解的 ***RetentionPolicy*** 应该设置为 ***RUNTIME*** 否则java类的注解信息在执行过程中将不可用，那么我们也不能从中得到任何和注解有关的数据。

