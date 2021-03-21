# 【Java Lambda】匿名类 vs Lambda

匿名类是没有名称的内部类，这意味着我们可以同时声明和实例化类。lambda 表达式是编写匿名类的一种简短形式。通过使用 lambda 表达式，我们可以声明没有任何名称的方法。

## Anonymous class vs Lambda Expression

- 匿名类对象在编译后生成一个单独的类文件，当lambda表达式被转换为私有方法时，该类文件会增加jar文件的大小。它使用 invokedynamic 字节码指令动态绑定该方法，节省了时间和内存。
- 我们使用这个关键字在 lambda 表达式中表示当前类，而在匿名类的情况下，这个关键字可以表示特定的匿名类。
- 匿名类可以用于多个抽象方法，而 lambda 表达式专门用于函数接口。
- 我们只需要在 lambda 表达式中提供函数体，而在匿名类的情况下，我们需要编写冗余类定义。

## Example

```java
public class ThreadTest {
  public static void main(String[] args) {
    Runnable r1 = new Runnable() { // Anonymous class
      @Override
      public void run() {
        System.out.println("Using Anonymous class");
      }
    };
    Runnable r2 = () -> { // lambda expression
      System.out.println("Using Lambda Expression");
    };
    new Thread(r1).start();
    new Thread(r2).start();
  }
}
```

## Output

```java
Using Anonymous class
Using Lambda Expression
```