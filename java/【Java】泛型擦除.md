# 【Java】泛型擦除

### 什么是类型擦除

Java在编译后的字节码(.class)文件中是不包含泛型中的类型信息的，使用泛型的时候加上的类型参数，会被编译器在编译的时候去掉，这个过程就称为类型擦除。如在代码中定义的`List<Object>`和`List<String>`等类型，在编译之后都会变成List，JVM看到的只是List，而由泛型附加的类型信息对JVM来说是不可见的。 因此，对于JVM来说，`List<Object>`和`List<String>`就是同一个类，所以，泛型实际上是Java语言的一个语法糖，又被叫做**伪泛型**。

### 类型擦除带来的特性

1.泛型类并没有自己独有的Class类对象。比如并不存在`List<String>.class`或是`List<Integer>.class`，而只有`List.class`，例如，下面的代码输出结果为true。

```java
public static void main(String[] args) {  
    List<String> a = new ArrayList<String>();  
    List<Integer> b = new ArrayList<Integer>();  
    System.out.println(a.getClass() == b.getClass());  
}
12345
```

2.静态变量是被泛型类的所有实例所共享的。对于声明为`MyClass<T>`的类，访问其中的静态变量的方法仍然是`MyClass.StaticVar`。不管是通过`new MyClass<String>`，还是`new MyClass<Integer>`创建的对象，都是共享一个静态变量。
 3.泛型的类型参数不能用在Java异常处理的catch语句中。因为异常处理是由JVM在运行时刻来进行的。由于类型信息被擦除，JVM是无法区分两个异常类型`MyException<String>`和`MyException<Integer>`的。对于JVM来说，它们都是`MyException`类型的，也就无法执行与异常对应的catch语句。

### 类型擦除的过程

类型擦除的基本过程也比较简单，首先是找到用来替换类型参数的具体类。这个具体类一般是Object。如果指定了类型参数的上界的话，则使用这个上界。把代码中的类型参数都替换成具体的类。同时去掉出现的类型声明，即去掉<>的内容。比如`T get()`方法声明就变成了`Object get()`，`List<String>`就变成了`List`。接下来就可能需要生成一些桥接方法（bridge method）。这是由于擦除了类型之后的类可能缺少某些必须的方法。比如考虑下面的代码，当类型信息被擦除之后，上述类的声明变成了`Class MyString implements Comparable`。但是这样的话，类`MyString`就会有编译错误，因为没有实现接口`Comparable`声明的`int compareTo(Object)`方法。这个时候就由编译器来动态生成这个方法。

```java
class MyString implements Comparable<String> {
    public int compareTo(String str) {        
        return 0;    
    }
} 
12345
```

### 实例分析

了解了类型擦除机制之后，就会明白编译器承担了全部的类型检查工作。编译器禁止某些泛型的使用方式，正是为了确保类型的安全性。以上面提到的`List<Object>`和`List<String>`为例来具体分析：

```java
public void inspect(List<Object> list) {    
    for (Object obj : list) {        
        System.out.println(obj);    
    }    
    list.add(1); 
}
public void test() {    
    List<String> strs = new ArrayList<String>();    
    inspect(strs); //编译错误 
}
12345678910
```

这段代码中，`inspect()`方法接受`List<Object>`作为参数，当在`test()`方法中试图传入`List<String>`的时候，会出现编译错误。假设这样的做法是允许的，那么在`inspect()`方法就可以通过`list.add(1)`来向集合中添加一个数字。这样在`test()`方法看来，其声明为`List<String>`的集合中却被添加了一个Integer类型的对象。这显然是违反类型安全的原则的，在某个时候肯定会抛出`ClassCastException`。因此，编译器禁止这样的行为。编译器会尽可能的检查可能存在的类型安全问题。对于确定是违反相关原则的地方，会给出编译错误。当编译器无法判断类型的使用是否正确的时候，会给出警告信息。