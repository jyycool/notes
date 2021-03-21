# 【JVM】对象创建在堆、栈和方法区的分布和关系

通过一个简单的示例展示, Java 堆、方法区和 Java 栈之间的关系

```java
public class SimpleHeap {
  private int id;
  
  public SimpleHeap(int id) {
    this.id = id;
  }
  
  public void show() {
    System.out.println("My Id is " + id);
  }
  
  public static void main(String[] args) {
    SimpleHeap s1 = new SimpleHeap(101);
    SimpleHeap s2 = new SimpleHeap(201);
    s1.show;
    s2.show;
  }
}
```

上述代码中类, 对象和局部变量的存放关系如下图.

![堆栈方法区](../allPics/Jvm/%E5%A0%86%E6%A0%88%E6%96%B9%E6%B3%95%E5%8C%BA.png)

- 类信息存放在方法区(JDK8之后改叫元数据区)
- 局部变量s1, s2 存放于Java 栈中的main栈帧中的局部变量表里
- 栈帧局部变量表里的局部变量s1, s2 分别指向堆中的两个实例