# 【JUC】从volatile引出的CPU锁机制

#Lang/Java

众所周知, Java多线程操作共享资源时会引出三个问题: 共享资源的可见性, 有序性和原子性

### 悲观锁

一般情况下，我们采用synchronized同步锁(独占锁、互斥锁、悲观锁)，即同一时间只有一个线程能够修改共享变量，其他线程必须等待。但是这样的话就相当于单线程，体现不出来多线程的优势。并且该线程获取到资源的锁时会使其他需要使用该资源但未获取锁的线程暂时挂起, 直到该线程处理完资源并释放锁, 其他线程才会继续竞争资源的锁

> 悲观锁的缺点:

1. 线程加锁和释放锁会引发较多的上下文的切换以及延时调度, 性能很差
2. 一个线程持有锁时会导致其他需要该锁的线程挂起
3. 如果一个优先高的线程等待优先级低的线程释放锁会引起优先级颠倒问题

volatile是不错的机制，但是volatile不能保证原子性。因此对于同步最终还是要回锁机制上来。

我们先不讨论如何解决悲观锁的缺点, 我们先看下volatile在底层的实现原理

### Lock指令

Java 的 volatile 其实是通过汇编指令，在 volatile 修饰的变量操作前加上 lock 前缀进行底层处理的，在 CPU 的底层其实是不存在 volatile 这种概念的。

x86汇编中，如果对一个指令加“lock”前缀，会发生什么，这里稍微详细解释一下

首先看一下手册的原文，见[https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf](https://link.zhihu.com/?target=https%3A//www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf)

![intel_lock](/Users/sherlock/Desktop/notes/allPics/Java/intel_lock.jpg)

文档中说的很清楚，对于 Lock 指令区分两种实现方法

> 对于早期的CPU，总是采用的是锁总线的方式。具体方法是，一旦遇到了Lock指令，就由仲裁器选择一个核心独占总线。其余的CPU核心不能再通过总线与内存通讯。从而达到“原子性”的目的。
>
> 具体做法是，某一个核心触发总线的“Lock#”那根线，让总线仲裁器工作，把总线完全分给某个核心。
>
> 这种方式的确能解决问题，但是非常不高效。为了个原子性结果搞得其他CPU都不能干活了。
>
> 因此从Intel P6 CPU开始就做了一个优化，改用***Ringbus + MESI协议***，也就是文档里说的cache conherence机制。这种技术被Intel称为“Cache Locking”。
>
> 根据文档原文：如果是P6后的CPU，并且数据已经被CPU缓存了，并且是要写回到主存的，则可以用cache locking处理问题。**否则还是得锁总线**。因此，lock到底用锁总线，还是用cache locking，完全是看当时的情况。当然能用后者的就肯定用后者。
>
> ```
> Intel P6是Intel第6代架构的CPU，其实也很老了，差不多1995年出的…… 比如Pentium Pro，Pentium II，Pentium III都隶属于P6架构。
> ```
>
> MESI大致的意思是：若干个CPU核心通过ringbus连到一起。每个核心都维护自己的Cache的状态。如果对于同一份内存数据在多个核里都有cache，则状态都为S（shared）。一旦有一核心改了这个数据（状态变成了M），其他核心就能瞬间通过ringbus感知到这个修改，从而把自己的cache状态变成I（Invalid），并且从标记为M的cache中读过来。同时，这个数据会被原子的写回到主存。最终，cache的状态又会变为S。
>
> 这相当于给cache本身单独做了一套总线（要不怎么叫ring bus），避免了真的锁总线。
>
> ```
> 但是有两种情况下处理器不会使用缓存锁定。
> 第一种情况是：当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行(cache line)，则处理器会调用总线锁定。
> 第二种情况是：有些处理器不支持缓存锁定。对于Inter486和奔腾处理器,就算锁定的内存区域在处理器的缓存行中也会调用总线锁定。
> ```

回到CAS。

### CAS原理

我们一般说的CAS在x86的大概写法是

```c#
lock cmpxchg a, b, c
```

贴一下x86的cmpxchg指定Opcode CMPXCHG: 

```c++
CPU: I486+ 
 Type of Instruction: User 

 Instruction: CMPXCHG dest, src 

 Description: Compares the accumulator with dest. If equal the "dest" 
 is loaded with "src", otherwise the accumulator is loaded 
 with "dest". 

 Flags Affected: AF, CF, OF, PF, SF, ZF 

 CPU mode: RM,PM,VM,SMM 

+++++++++++++++++++++++

 Clocks: 
 CMPXCHG reg, reg 6 
 CMPXCHG mem, reg 7 (10 if compartion fails) 
```

对于一致性来讲，“lock”前缀是起关键作用的指令。而cmpxchg是一个原子执行“load”，“compare”，“save“的操作。Intel规定lock只能对单条指令起作用。我不是很清楚为啥Intel是这么设计的，比如必须要lock和cmpxchg一起写。有没有可能cmpxchg可以单独用的时候？也许是单核处理器的时候可以不用lock了？( 确实是单核就没必要加lock了，例如在windowsX86中会先试用os::isMP方法看看是不是多核的，如果是就加lock )

CAS的特性使得它成为实现任何高层“锁”的必要的构建。几乎所有的“锁”，如Mutex，ReentrantLock等都得用CAS让线程先原子性的抢到一个东西（比如一个队列的头部），然后才能维护其他锁相关的数据。并且很有意思的是，如果一个竞争算法只用到了CAS，却没有让线程“等待”，就会被称为“无锁算法”。

### 乐观锁

假设不会发生并发冲突, 只有在最后更新共享资源的时候会判断一下在此期间有没有别的线程修改了这个共享资源。***如果发生冲突就重试，直到没有冲突，更新成功(整个这个操作不可以切分开直到更新成功才算是一个完整的原子操作)。***

乐观锁的底层是基于CAS来实现的, 而CAS底层是使用汇编指令`lock cmpxchg a, b, c`来实现的, 而lock指令在CPU硬件层面是通过***总线锁/MESI缓存一致性实现***的

CAS ， 就是CPU指令`lock cmpxchg v, a, b `，它是一种乐观锁技术，多个线程尝试更新同一个变量，其中一个线程更新这一个变量，其他线程操作失败，失败线程不会挂起，而是被通知竞争失败，再次尝试。 

CAS有3个操作数，内存值V，旧的预期值A，新的值B, 如果 V==B那么更新值变为:

```java
//CAS比较与交换代码
do{
    备份旧值A
    计算逻辑...
    构造新值B 
}while(!CAS(内存值V，旧值A，新值B))

public final long getAndAdd(long delta) {
  while (true) {
    long current = get();
    long next = current + delta;
    if (compareAndSet(current, next))
      return current;
	}
}
//

```

V与A比较如果V没有被修改，那么替换新值，如果被修改了内存中的值返回false，并放弃刚才的操作，然后重新再次执行，直到执行成功对CAS的支持 `AtomicInt/AtomicLong.incrementAndGet()`

现代的CPU提供了特殊的指令，可以自动更新共享数据，而且能够检测到其他线程的干扰，而 compareAndSet() 就用这些代替了锁定。

#### AtomicInteger

拿出AtomicInteger来研究在没有锁的情况下是如何做到数据正确性的。

```java
private volatile int value;
```

首先毫无疑问，在没有锁的机制下可能需要借助volatile原语，保证线程间的数据是可见的（共享的）。这样在获取变量的值的时候才能直接读取。

```java
public final int get() {
	return value;
}
```

然后来看看++i是怎么做到的。

```java
public final int incrementAndGet() {
   for (;;) {
     int current = get();
     int next = current + 1;
     if (compareAndSet(current, next))
       return next;
   }
 }
```

在这里采用了CAS操作，每次从内存中读取数据然后将此数据和+1后的结果进行CAS操作，如果成功就返回结果，否则重试直到成功为止。而`compareAndSet`利用JNI来完成CPU指令的操作。

```java
public final boolean compareAndSet(int expect, int update) {  
   return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

整体的过程就是这样子的，利用CPU的CAS指令，同时借助JNI来完成Java的非阻塞算法。其它原子操作都是利用类似的特性完成的。

其中

```java
unsafe.compareAndSwapInt(this, valueOffset, expect, update);
```

类似：

```java
if (this == expect) {
 	this = update
	return true;
} else {
	return false;
}
```

那么问题就来了，成功过程中需要2个步骤：

1. 比较 this == expect

2. 将this替换为update

3. sdsd 

   compareAndSwapInt如何这两个步骤的原子性呢？ 参考上面CAS的原理。



### 应用层锁和CPU锁的关系

CPU锁和应用层的锁要解决的问题不一样。

CPU锁主要解决的是***多个核心并发访问/修改同一块内存的问题***。所以有***锁总线***和***MESI缓存一致性***来做。对于上层主要的抽象就是CAS。主要的招数就是用CAS+循环来抢东西。如果抢不到就只能

- 继续循环下去玩命抢（这时会空耗CPU）
- 不抢了，回复给上层代码“抢不到”。

应用层的锁存在了“进程/线程“的概念（下文统一都说进程）。解决的是多个进程并发访问同一块内存的问题。比起CPU的层级来说，应用层的锁可以多一个招数，叫做“让给当前进程不可调度“。这个是OS提供的支持。

因此在应用层的层次上你可以定义一个高级的“锁”，大概执行这样一个抢锁流程

1. 尝试用CAS抢到锁
2. 如果抢不到，则回到1重试
3. 如果抢了几十次都还抢不到，就把当前进程（的信息）尝试挂到一个等待队列上（当然挂的过程还是要CAS）
4. 把当前进程设定为不可调度，这样OS就不会把当前进程调度给CPU执行。（这种情况因为需要做一次系统调用，所以有比较大的损耗，一般被称为“重量级锁”)

而当某个进程释放锁时，他就可以做释放锁的流程

1.  找到释放锁的那个等待队列
2. 把等待队列里第一个等待的进程信息取出来，并且告诉OS，这个进程程可以执行了（这里也要做一次系统调用）
3. 这个被复活了的进程一般需要在做一次循环尝试抢锁，然后就回到了上面的抢锁流程。



### CAS在JUC中的实现

java中提供了Unsafe类，它提供了三个函数，分别用来操作基本类型int和long，以及引用类型Object

```java
public final native boolean compareAndSwapObject
  (Object obj, long valueOffset, Object expect, Object update);

public final native boolean compareAndSwapInt
  (Object obj, long valueOffset, int expect, int update);

public final native boolean compareAndSwapLong
  (Object obj, long valueOffset, long expect, long update);
```

参数的意义：

> 1. obj 和 valueOffset：表示这个共享变量的内存地址。这个共享变量是obj对象的一个成员属性，valueOffset表示这个共享变量在obj类中的内存偏移量。所以通过这两个参数就可以直接在内存中修改和读取共享变量值。
> 2. expect: 表示预期原来的值。
> 3. update: 表示期待更新的值。



调用JUC并发框架下原子类的方法时，不需要考虑多线程问题。那么我们分析它是怎么解决多线程问题的。以***AtomicInteger.class***为例

#### AtomicInteger

```java
 // 通过它来实现CAS操作的。因为是int类型，所以调用它的compareAndSwapInt方法
private static final Unsafe unsafe = Unsafe.getUnsafe();

// value这个共享变量在AtomicInteger对象上内存偏移量，
// 通过它直接在内存中修改value的值，compareAndSwapInt方法中需要这个参数
private static final long valueOffset;

// 通过静态代码块，在AtomicInteger类加载时就会调用
static {
  try {
    // 通过unsafe类，获取value变量在AtomicInteger对象上内存偏移量
    valueOffset = unsafe.objectFieldOffset
      (AtomicInteger.class.getDeclaredField("value"));
  } catch (Exception ex) { throw new Error(ex); }
}

// 共享变量，AtomicInteger就保证了对它多线程操作的安全性。
// 使用volatile修饰，解决了可见性和有序性问题。
private volatile int value;
```

有三个重要的属性：

> 1. unsafe: 通过它实现CAS操作，因为共享变量是int类型，所以调用compareAndSwapInt方法。
> 2. valueOffset: 共享变量value在AtomicInteger对象上内存偏移量
> 3. value: 共享变量，使用volatile修饰，解决了可见性和有序性问题。

#### 重要方法

##### 1.  get与set方法

```csharp
 // 直接读取。因为是volatile关键子修饰的，总是能看到(任意线程)对这个volatile变量最新的写入
public final int get() {
  return value;
}

// 直接写入。因为是volatile关键子修饰的，所以它修改value变量也会立即被别的线程读取到。
public final void set(int newValue) {
  value = newValue;
}
```

因为value变量是volatile关键字修饰的，它总是能读取(任意线程)对这个volatile变量最新的写入。它修改value变量也会立即被别的线程读取到。

##### 2 compareAndSet方法

```java
// 如果value变量的当前值(内存值)等于期望值(expect)，那么就把update赋值给value变量，返回true。
// 如果value变量的当前值(内存值)不等于期望值(expect)，就什么都不做，返回false。
// 这个就是CAS操作，使用unsafe.compareAndSwapInt方法，保证整个操作过程的原子性
public final boolean compareAndSet(int expect, int update) {
  return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

通过调用unsafe的compareAndSwapInt方法实现CAS函数的。但是CAS函数只能保证比较并交换操作的原子性，但是更新操作并不一定会执行。比如我们想让共享变量value自增。
 共享变量value自增是三个操作，1.读取value值，2.计算value+1的值，3.将value+1的值赋值给value。分析这三个操作：

1. 读取value值,因为value变量是volatile关键字修饰的，能够读取到任意线程对它最后一次修改的值，所以没问题。

2. 计算value+1的值：这个时候就有问题了，可能在计算这个值的时候，其他线程更改了value值，因为没有加同步锁，所以其他线程可以更改value值。

3. 将value+1的值赋值给value: 使用CAS函数，如果返回false，说明在当前线程读取value值到调用CAS函数方法前，共享变量被其他线程修改了，那么value+1的结果值就不是我们想要的了，因为要重新计算。

##### 3 getAndAddInt方法

```kotlin
public final int getAndAddInt(Object obj, long valueOffset, int var) {
  int expect;
  // 利用循环，直到更新成功才跳出循环。
  do {
    // 获取value的最新值
    expect = this.getIntVolatile(obj, valueOffset);
    // expect + var表示需要更新的值，如果compareAndSwapInt返回false，说明value值被其他线程更改了。
    // 那么就循环重试，再次获取value最新值expect，然后再计算需要更新的值expect + var。直到更新成功
  } while(!this.compareAndSwapInt(obj, valueOffset, expect, expect + var));

  // 返回当前线程在更改value成功后的，value变量原先值。并不是更改后的值
  return expect;
}
```

这个方法在Unsafe类中，利用do_while循环，先利用当前值，计算更新值，然后通过compareAndSwapInt方法设置value变量，如果compareAndSwapInt方法返回失败，表示value变量的值被别的线程更改了，所以循环获取value变量最新值，再通过compareAndSwapInt方法设置value变量。直到设置成功。跳出循环，返回更新前的值。

```java 
// 将value的值当前值的基础上加1，并返回当前值
public final int getAndIncrement() {
  return unsafe.getAndAddInt(this, valueOffset, 1);
}

// 将value的值当前值的基础上加-1，并返回当前值
public final int getAndDecrement() {
  return unsafe.getAndAddInt(this, valueOffset, -1);
}


// 将value的值当前值的基础上加delta，并返回当前值
public final int getAndAdd(int delta) {
  return unsafe.getAndAddInt(this, valueOffset, delta);
}


// 将value的值当前值的基础上加1，并返回更新后的值(即当前值加1)
public final int incrementAndGet() {
  return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// 将value的值当前值的基础上加-1，并返回更新后的值(即当前值加-1)
public final int decrementAndGet() {
  return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
}

// 将value的值当前值的基础上加delta，并返回更新后的值(即当前值加delta)
public final int addAndGet(int delta) {
  return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}
```

都是利用unsafe.getAndAddInt方法实现的。

#### 示例

```cpp
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;

class Data {
    AtomicInteger num;

    public Data(int num) {
        this.num = new AtomicInteger(num);
    }

    public int getAndDecrement() {
        return num.getAndDecrement();
    }
}

class MyRun implements Runnable {

    private Data data;
    // 用来记录所有卖出票的编号
    private List<Integer> list;
    private CountDownLatch latch;

    public MyRun(Data data, List<Integer> list, CountDownLatch latch) {
        this.data = data;
        this.list = list;
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            action();
        }  finally {
            // 释放latch共享锁
            latch.countDown();
        }
    }

    // 进行买票操作，注意这里没有使用data.num>0作为判断条件，直到卖完线程退出。
    // 那么做会导致这两处使用了共享变量data.num，那么做多线程同步时，就要考虑更多条件。
    // 这里只for循环了5次，表示每个线程只卖5张票，并将所有卖出去编号存入list集合中。
    public void action() {
        for (int i = 0; i < 5; i++) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            int newNum = data.getAndDecrement();

            System.out.println("线程"+Thread.currentThread().getName()+"  num=="+newNum);
            list.add(newNum);
        }
    }
}

public class ThreadTest {


    public static void startThread(Data data, String name, List<Integer> list,CountDownLatch latch) {
        Thread t = new Thread(new MyRun(data, list, latch), name);
        t.start();
    }

    public static void main(String[] args) {
        // 使用CountDownLatch来让主线程等待子线程都执行完毕时，才结束
        CountDownLatch latch = new CountDownLatch(6);

        long start = System.currentTimeMillis();
        // 这里用并发list集合
        List<Integer> list = new CopyOnWriteArrayList();
        Data data = new Data(30);
        startThread(data, "t1", list, latch);
        startThread(data, "t2", list, latch);
        startThread(data, "t3", list, latch);
        startThread(data, "t4", list, latch);
        startThread(data, "t5", list, latch);
        startThread(data, "t6", list, latch);


        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 处理一下list集合，进行排序和翻转
        Collections.sort(list);
        Collections.reverse(list);
        System.out.println(list);

        long time = System.currentTimeMillis() - start;
        // 输出一共花费的时间
        System.out.println("\n主线程结束 time=="+time);
    }
}
```