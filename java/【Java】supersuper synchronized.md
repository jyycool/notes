# 【Java】super

说到, super关键字, 但凡入门 Java 的都知道, 这有啥可说的, super 不就是在类中对于当前类的父类的引用么? 是的, 一点没错, 那么我来看看下面这个例子。

```java
public class SuperClass {

  public synchronized void method() throws InterruptedException {
    System.out.print("SuperClass#method [");
    TimeUnit.SECONDS.sleep(20);
    System.out.print("]\n");
  }
}

public class SubClass extends SuperClass {

  @Override
  public synchronized void method() throws InterruptedException {
    System.out.print("SubClass#method [");
    TimeUnit.SECONDS.sleep(20);
    System.out.print("]\n");
    super.method();
  }

  public static void main(String[] args) throws InterruptedException {

    SubClass subClass = new SubClass();
    subClass.method();
  }
}
/*
	SubClass#method []
	SuperClass#method []
 */
```

执行结果中 "[]" 总是成对的出现。加锁是成功的, 那么问题来了, super.method() 这个方法被调用的时候, 获取的锁是父类对象的锁还是子类对象的锁?

ok, 我们通过工具 jstack 来查看:

```sh
"main" #1 prio=5 os_prio=31 tid=0x00007fef0180d000 nid=0xf03 waiting on condition [0x0000700002d21000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at cgs.java.sync.SuperClass.method(SuperClass.java:12)
	- locked <0x00000007956a72b8> (a cgs.java.sync.SubClass) ⑥
	at cgs.java.sync.SubClass.method(SubClass.java:17)
	- locked <0x00000007956a72b8> (a cgs.java.sync.SubClass) ⑧
	at cgs.java.sync.SubClass.main(SubClass.java:23)
```

⑥ 和 ⑧ 处说明了, 在 main 线程中代码 12 和 17 行处有两个锁, 锁住的是同一个对象: `cgs.java.sync.SubClass` 对象