# 【Java 8】lambda使用局部变量的限制

Java 8 引入的新特性里最重要的就是 `lambda` 表达式了, 它的出现表明了 Java 也进入了函数式编程的时代.

Java 8 中的常用的函数式接口

| 函数式接口          | 函数描述符        | 原始类型特化                                                 |
| ------------------- | ----------------- | ------------------------------------------------------------ |
| Predicate\<T>       | T -> Boolean      | IntPredicate<br>LongPredicate <br>DoublePredicate            |
| Consumer\<T>        | T -> void         | IntConsumer<br>LongConsumer<br>DoubleConsumer                |
| Function<T, R>      | T -> R            | IntFunction\<R><br>IntToDoubleFunction<br>IntToLongFunction<br>LongFunction\<R><br>LongToDoubleFunction<br>LongToIntFunction<br>DoubleFunction\<R><br>ToIntFunction\<T><br>ToDoubleFunction\<T><br>ToLongFunction\<T> |
| Supplier\<T>        | () -> T           | BooleanSupplier<br>IntSupplier<br>LongSupplier<br>DoubleSupplier |
| UnaryOperator\<T>   | T -> T            | IntUnaryOperator<br>LongUnaryOperator<br/>DoubleUnaryOperator |
| BinaryOperator\<T>  | (T, T) -> T       | IntBinaryOperator<br>LongBinaryOperator<br/>DoubleBinaryOperator |
| BiPredicate<L, R>   | (L, R) -> boolean |                                                              |
| BiConsumer<T, U>    | (T, U) -> void    | ObjIntConcumer<br>ObjLongConcumer<br/>ObjDoubleConcumer      |
| BiFunction<T, U, R> | (T, U) -> R       | ToIntBiFunction<T, U><br>ToLongBiFunction<T, U><br/>ToDoubleBiFunction<T, U> |

但在使用 `Java 8` 新特性 `lambda` 的时候, 如果 `lambda` 表达是内部引用了外部的局部变量, 并且这个变量不是 `final` 的,此时会报错. why???

开门见山的说, 这是 Java 为了防止数据不同步而规定的, 也就是为了防止你在 `lambda` 内部使用的外层局部变量被代码修改了, 而 `lambda` 内部无法同步这个修改

那么为什么你在外层修改了局部变量而 `lambda` 内部却感知不到呢 ? 这个问题就要回到 `lambda` 的本质了.

## lambda 本质

lambda 表达式的本质是什么 ? 也就是匿名内部类嘛. 或者在准确一些的说, 它就是一个函数式接口的实现的实例嘛.

## 外部变量的传递

那么你再想一下, 把一个外部变量拿到这个实例中去使用, 这个变量如何传递进去呢?  最简单的方法, 构造器嘛. `lambda` 表达式实例化的时候, 编译器会创建一个新的 `class` 文件(仔细想一下你是不是在工程编译后见到过类似于 `Main$1.class` 的文件), 该文件就是 `lambda` 实例化的类的字节码文件,  在文件中, 编译器帮我们创建了一个构造器, 这个构造器的入参中就包含了你要是用的外层局部变量, 所以本质上外层局部变量的传递就是通过 `lambda` 构造器传入实例内部供其使用的.

## 值传递和引用传递

那么这个时候就很好理解, 为什么 `lambda` 内部使用外层变量必须是 `final` 的了, 你想嘛, 既然是通过构造器传递, 本质上也就是方法传递, 如果这个变量是基本数据类型, 那么方法中基本数据类型的传递方法就是值传递, 通俗一点说, 方法内部其实就是 copy 了一份这个基本类型变量的副本, 所以你在外层改变了这个变量的值, `lambda` 只是引用了副本, 肯定无法感知这个变化嘛. 不过如果是引用传递, 这个问题问题就不存在了嘛, 因为引用传递的就是变量在内存的地址值, 并不会传递引用对象内部的属性值.

## lambda 只是声明

`lambda` 只是声明, 怎么理解呢? 无论是 `lambda` 还是匿名内部类, 在写 `lambda` 表达式的时候都不会去直接执行表达式的, `lambda` 只是一种声明, 和声明变量一样, 你声明一个变量 `int x`, 这仅仅是一个声明, 可能你会在很多行代码后才去执行这个表达式, 例如:

```java
Thread t1 = new Thread(() -> System.out.println("call..."));
System.out.println("main call ...");
// 这里经过了多含代码
t1.start();  
```

在执行第一行代码的时候, 声明了一个 `lambda` 表达式, 但此时并未执行表达式, 在第 4 行代码才会执行 `lambda` 表达式, 但是假如在代码第一行 `lambda` 内部使用了外层变量 x, 其初始值是 10, 之后的多行代码中修改了这个变量 x 的值, 在第四行去执行 `lambda` 的时候, 其内部这个变量 x 的值仍然为 10, 这就造成了数据不同步的问题了.