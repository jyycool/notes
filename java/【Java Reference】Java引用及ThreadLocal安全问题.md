# 【Java Reference】Java引用及ThreadLocal安全问题

#Lang/Java

### OverView

Java中的基本引用: 

1. ***强引用 (normal reference)***
2. ***软引用 (soft reference)***
3. ***弱引用 (weak reference)***
4. ***虚引用 (phantom reference)***

```java
package cgs.reference;

/**
 * Descriptor: 当一个对象被回收时, 它的 finalize() 方法会被调用
 * Author: sherlock
 */
public class M {

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("into finalize......");
    }
}
```

我们都知道一个对象如果没有了任何引用，java虚拟机就认为这个对象没什么用了，就会对其进行垃圾回收，但是如果这个对象包含了finalize函数，性质就不一样了。怎么不一样了呢？

java虚拟机在进行垃圾回收的时候，一看到这个对象类含有finalize函数，就把这个函数交给FinalizerThread处理，而包含了这个finalize的对象就会被添加到FinalizerThread的执行队列，并使用一个链表，把这些包含了finalize的对象串起来。

它的影响在于只要finalize没有执行，那么这些对象就会一直存在堆区



### 强引用

是指创建一个对象并把这个对象赋给一个引用变量。

也是最常见的引用: ***Object o = new Object();*** 

```java
package cgs.reference;

import java.io.IOException;

/**
 * Descriptor: 强引用
 * Author: sherlock
 *
 * ps: JVM垃圾回收的根对象(gc root根)的范围有以下几种：
 * （1）虚拟机（JVM）栈中引用对象
 * （2）方法区中的类静态属性引用对象
 * （3）方法区中常量引用的对象（final 的常量值）
 * （4）本地方法栈JNI的引用对象
 */
public class NormalRef {

    public static void main(String[] args) throws IOException {

        M m = new M();
        m = null;
        System.gc();
        /*
            垃圾回收不是在 main 线程中执行的,
            所以必须保证主线程不退出, 给gc线程时间来启动执行gc
         */
        System.in.read();

    }

}
```



### 软引用

如果一个对象具有软引用，内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。

软引用可用来实现内存敏感的高速缓存, 比如网页缓存、图片缓存等。使用软引用能防止内存泄露，增强程序的健壮性。 

SoftReference的特点是它的一个实例保存对一个Java对象的软引用， 该软引用的存在不妨碍垃圾收集线程对该Java对象的回收。

也就是说，一旦SoftReference保存了对一个Java对象的软引用后，在垃圾线程对 这个Java对象回收前，SoftReference类所提供的get()方法返回Java对象的强引用。

另外，一旦垃圾线程回收该Java对象之 后，get()方法将返回null。

```java
package cgs.reference;

import  java.lang.ref.SoftReference;
import java.util.concurrent.TimeUnit;

/**
 * Descriptor: -Xmx15m
 * Author: sherlock
 */
public class SoftRef {

    public static void main(String[] args) {

        SoftReference<byte[]> ref = new SoftReference<>(new byte[1024 * 1024 * 10]);
//        ref = null;
        System.out.println(ref.get());
        System.gc();
        try {
            TimeUnit.MILLISECONDS.sleep(500L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(ref.get());
      	//当插入这个10mb的数组时, 内存空间不够, 且这个引用是强引用,就会gc回收软引用
        byte[] bytes = new byte[1024 * 1024 * 10];
        System.out.println(ref.get());
    }
}

执行结果:
[B@61bbe9ba
[B@61bbe9ba
null
```

ref是强引用, 但是 SoftReference 中指向的 byte[ ]数组的引用就称为 软引用, 当发生 gc 并且内存中插入一个较大对象, 内存空间不足的时候, 就会将软引用指向对象进行回收.

![soft](/Users/sherlock/Desktop/notes/allPics/Java/soft.png)



### 弱引用

弱引用也是用来描述非必需对象的，当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。在java中，用java.lang.ref.WeakReference类来表示。

```java
package cgs.reference;

import java.lang.ref.WeakReference;

/**
 * Descriptor: 弱引用遇到 gc 就会被回收
 * Author: sherlock
 */
public class WeakRef {

    public static void main(String[] args) {

        WeakReference<M> ref = new WeakReference<>(new M());

        System.out.println(ref.get());

        System.gc();
        System.out.println(ref.get());

//        ThreadLocal<M> threadLocal = new ThreadLocal<>();
//        threadLocal.set(new M());
//        threadLocal.remove();
        
    }

}

执行结果:
cgs.reference.M@61bbe9ba
null
into finalize......
```



### 虚引用

虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。在java中用 ***java.lang.ref.PhantomReference*** 类表示。如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收。

```java
package cgs.reference;

import java.lang.ref.PhantomReference;
import java.lang.ref.Reference;
import java.lang.ref.ReferenceQueue;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * Descriptor: -verbose:gc -Xmx5m
 * Author: sherlock
 */
public class PhantomRef {

    private static final List<Object> LIST = new LinkedList<>();
    private static final ReferenceQueue<Fin> QUEUE = new ReferenceQueue<>();

    public static void main(String[] args) {

        PhantomReference<Fin> ref = new PhantomReference<>(new Fin(), QUEUE);

        new Thread(() -> {
            while (true) {
                LIST.add(new byte[1024 * 1024]);
                try {
                    TimeUnit.SECONDS.sleep(1L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    Thread.currentThread().interrupt();
                }
                System.out.println(ref.get());
            }
        }).start();

        new Thread(() -> {
            while (true) {
                Reference<? extends Fin> poll = QUEUE.poll();
                if (poll != null) {
                    System.out.println("--- 虚引用对象被jvm回收了 ---" + poll);
                    System.out.println("--- 回收对象 ---- " + poll.get());
                }
            }
        }).start();

        try {
            Thread.currentThread().join();
        } catch (InterruptedException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }
}

执行结果:
[GC (Allocation Failure)  1024K->464K(5632K), 0.0010521 secs]
[GC (Allocation Failure)  1488K->552K(5632K), 0.0058033 secs]
into finalize......
[GC (Allocation Failure)  1576K->672K(5632K), 0.0022589 secs]
[GC (Allocation Failure)  2712K->1944K(5632K), 0.0010932 secs]
null
null
null
[GC (Allocation Failure)  4355K->4144K(5632K), 0.0026014 secs]
[GC (Allocation Failure)  4144K->4160K(5632K), 0.0008544 secs]
[Full GC (Allocation Failure)  4160K->4007K(5632K), 0.0130259 secs]
[GC (Allocation Failure)  4007K->4007K(5632K), 0.0005680 secs]
[Full GC (Allocation Failure)  4007K->3916K(5632K), 0.0102369 secs]
--- 虚引用对象被jvm回收了 ---java.lang.ref.PhantomReference@22f26843
--- 回收对象 ---- null
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at cgs.reference.PhantomRef.lambda$main$0(PhantomRef.java:25)
	at cgs.reference.PhantomRef$$Lambda$1/1607521710.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)
```

因为设置的虚拟机堆大小比较小，所以创建一个100k的对象时直接进入了老年代，等到发生Full GC时才会被扫描然后回收。

使用虚引用的目的就是为了得知对象被GC的时机，所以可以利用虚引用来进行销毁前的一些操作，比如说资源释放等。这个虚引用对于对象而言完全是无感知的，有没有完全一样，但是对于虚引用的使用者而言，就像是待观察的对象的把脉线，可以通过它来观察对象是否已经被回收，从而进行相应的处理。

***虚引用比较典型的存在是NIO操作直接内存的时候创建的那个对象如果被释放之后，直接内存要通过虚引用的Cleaner对象调用Unsafe的freememory方法释放那块直接内存。***



> 小结:
>
> - 虚引用是最弱的引用
> - 虚引用对对象而言是无感知的，对象有虚引用跟没有是完全一样的
> - 虚引用不会影响对象的生命周期
> - 虚引用可以用来做为对象是否存活的监控



### ThreadLocal

