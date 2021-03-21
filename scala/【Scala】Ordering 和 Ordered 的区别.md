# 【Scala】Ordering 和 Ordered 的区别

一般学习Scala语言的人，都是有Java语言基础的。

我们知道在Java语言中对象的比较有两个接口，分别是`Comparable`和`Comparator`。那么它们之间的区别是什么呢？

- 实现`Comparable`接口的类，重写`compareTo()`方法后，其对象自身就具有了可比较性；
- 实现`comparator`接口的类，重写了`compare()`方法后，则提供一个第三方比较器，用于比较两个对象。

由于Scala是基于Java语言实现的，所以在Scala语言中也引入了以上两种比较方法(scala.math包下)：

- `Ordered` 特质定义了相同类型间的比较方式，但这种内部比较方式是单一的；

```java
trait Ordered[A] extends Any with java.lang.Comparable[A]{......}
1
```

- `Ordering`则是提供第三方比较器，可以自定义多种比较方式，在实际开发中也是使用比较多的，灵活解耦合。

```java
trait Ordering[T] extends Comparator[T] with PartialOrdering[T] with Serializable {......}
1
```

以上就是两种比较方式的简单总结。