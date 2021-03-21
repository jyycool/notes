# 22 | Executor与线程池：如何创建正确的线程池？

![](https://static001.geekbang.org/resource/image/b4/da/b4ba2177ffc160cb9489985170fa81da.jpg)

虽然在 Java 语言中创建线程看上去就像创建一个对象一样简单，只需要 new Thread() 就可以了，但实际上创建线程远不是创建一个对象那么简单。创建对象，仅仅是在 JVM  的堆里分配一块内存而已；而创建一个线程，却需要调用操作系统内核的 API，然后操作系统要为线程分配一系列的资源，这个成本就很高了，所以线程是一个重量级的对象，应该避免频繁创建和销毁。

那如何避免呢？应对方案估计你已经知道了，那就是线程池。

线程池的需求是如此普遍，所以 Java SDK 并发包自然也少不了它。但是很多人在初次接触并发包里线程池相关的工具类时，多少会都有点蒙，不知道该从哪里入手，我觉得根本原因在于线程池和一般意义上的池化资源是不同的。一般意义上的池化资源，都是下面这样，当你需要资源的时候就调用 acquire() 方法来申请资源，用完之后就调用 release()  释放资源。若你带着这个固有模型来看并发包里线程池相关的工具类时，会很遗憾地发现它们完全匹配不上，Java 提供的线程池里面压根就没有申请线程和释放线程的方法。

```java
class XXXPool{
  // 获取池化资源
  XXX acquire() {
  }
  // 释放池化资源
  void release(XXX x){
  }
}  
```

## 线程池是一种生产者-消费者模式

为什么线程池没有采用一般意义上池化资源的设计方法呢？如果线程池采用一般意义上池化资源的设计方法，应该是下面示例代码这样。你可以来思考一下，假设我们获取到一个空闲线程 T1，然后该如何使用 T1 呢？你期望的可能是这样：通过调用 T1 的 execute() 方法，传入一个 Runnable  对象来执行具体业务逻辑，就像通过构造函数 Thread(Runnable target) 创建线程一样。可惜的是，你翻遍 Thread  对象的所有方法，都不存在类似 execute(Runnable target) 这样的公共方法。

```java
//采用一般意义上池化资源的设计方法
class ThreadPool{
  // 获取空闲线程
  Thread acquire() {
  }
  // 释放线程
  void release(Thread t){
  }
} 
//期望的使用
ThreadPool pool；
Thread T1=pool.acquire();
//传入Runnable对象
T1.execute(()->{
  //具体业务逻辑
  ......
});
```

所以，线程池的设计，没有办法直接采用一般意义上池化资源的设计方法。那线程池该如何设计呢？目前业界线程池的设计，普遍采用的都是生产者 - 消费者模式。线程池的使用方是生产者，线程池本身是消费者。在下面的示例代码中，我们创建了一个非常简单的线程池 MyThreadPool，你可以通过它来理解线程池的工作原理。

```java
//简化的线程池，仅用来说明工作原理
class MyThreadPool{
  //利用阻塞队列实现生产者-消费者模式
  BlockingQueue<Runnable> workQueue;
  //保存内部工作线程
  List<WorkerThread> threads = new ArrayList<>();
  // 构造方法
  MyThreadPool(
    int poolSize, 
    BlockingQueue<Runnable> workQueue){
    
    this.workQueue = workQueue;
    // 创建工作线程
    for(int idx=0; idx<poolSize; idx++){
      WorkerThread work = new WorkerThread();
      work.start();
      threads.add(work);
    }
  }
  
  // 提交任务
  void execute(Runnable command){
    workQueue.put(command);
  }
  
  // 工作线程负责消费任务，并执行任务
  class WorkerThread extends Thread{
    
    @Override
    public void run() {
      //循环取任务并执行
      while(true){ ①
        Runnable task = workQueue.take();
        task.run();
      } 
    }
  }  
}

/** 下面是使用示例 **/
// 创建有界阻塞队列
BlockingQueue<Runnable> workQueue = new LinkedBlockingQueue<>(2);
// 创建线程池  
MyThreadPool pool = new MyThreadPool(10, workQueue);
// 提交任务  
pool.execute(()->{
    System.out.println("hello");
});
```

在 MyThreadPool 的内部，我们维护了一个阻塞队列 workQueue 和一组工作线程，工作线程的个数由构造函数中的 poolSize 来指定。用户通过调用 execute() 方法来提交 Runnable 任务，execute() 方法的内部实现仅仅是将任务加入到 workQueue 中。MyThreadPool 内部维护的工作线程会消费 workQueue 中的任务并执行任务，相关的代码就是代码①处的 while 循环。线程池主要的工作原理就这些，是不是还挺简单的？

## 如何使用 Java 中的线程池

Java 并发包里提供的线程池，远比我们上面的示例代码强大得多，当然也复杂得多。Java 提供的线程池相关的工具类中，最核心的是 ThreadPoolExecutor，通过名字你也能看出来，它强调的是  Executor，而不是一般意义上的池化资源。ThreadPoolExecutor 的构造函数非常复杂，如下面代码所示，这个最完备的构造函数有 7 个参数。

```java
ThreadPoolExecutor(
  int corePoolSize,
  int maximumPoolSize,
  long keepAliveTime,
  TimeUnit unit,
  BlockingQueue<Runnable> workQueue,
  ThreadFactory threadFactory,
  RejectedExecutionHandler handler) 
```

下面我们一一介绍这些参数的意义，你可以把线程池类比为一个项目组，而线程就是项目组的成员。- 

- `corePoolSize`

  表示线程池保有的最小线程数。有些项目很闲，但是也不能把人都撤了，至少要留 corePoolSize 个人坚守阵地。

- `maximumPoolSize`

  表示线程池创建的最大线程数。当项目很忙时，就需要加人，但是也不能无限制地加，最多就加到 maximumPoolSize 个人。当项目闲下来时，就要撤人了，最多能撤到 corePoolSize 个人。

- `keepAliveTime & unit`

  上面提到项目根据忙闲来增减人员，那在编程世界里，如何定义忙和闲呢？很简单，一个线程如果在一段时间内，都没有执行任务，说明很闲，keepAliveTime 和 unit 就是用来定义这个“一段时间”的参数。也就是说，如果一个线程空闲了keepAliveTime &  unit这么久，而且线程池的线程数大于 corePoolSize  ，那么这个空闲的线程就要被回收了。

- `workQueue`

  工作队列，和上面示例代码的工作队列同义。

- `threadFactory`

  通过这个参数你可以自定义如何创建线程，例如你可以给线程指定一个有意义的名字。

- `handler`

  通过这个参数你可以自定义任务的拒绝策略。如果线程池中所有的线程都在忙碌，并且工作队列也满了（前提是工作队列是有界队列），那么此时提交任务，线程池就会拒绝接收。

  至于拒绝的策略，你可以通过 handler 这个参数来指定。ThreadPoolExecutor 已经提供了以下 4 种策略。

  - CallerRunsPolicy：提交任务的线程自己去执行该任务。
  - AbortPolicy：默认的拒绝策略，会 throws RejectedExecutionException。
  - DiscardPolicy：直接丢弃任务，没有任何异常抛出。
  - DiscardOldestPolicy：丢弃最老的任务，其实就是把最早进入工作队列的任务丢弃，然后把新任务加入到工作队列。

Java 在 1.6 版本还增加了 allowCoreThreadTimeOut(boolean value) 方法，它可以让所有线程都支持超时，这意味着如果项目很闲，就会将项目组的成员都撤走。

## 使用线程池要注意些什么

考虑到 ThreadPoolExecutor 的构造函数实在是有些复杂，所以 Java 并发包里提供了一个线程池的静态工厂类 Executors，利用 Executors 你可以快速创建线程池。不过目前大厂的编码规范中基本上都不建议使用 Executors 了，所以这里我就不再花篇幅介绍了。

不建议使用 Executors 的最重要的原因是：Executors 提供的很多方法默认使用的都是无界的  LinkedBlockingQueue，高负载情境下，无界队列很容易导致 OOM，而 OOM 会导致所有请求都无法处理，这是致命问题。所以强烈建议使用有界队列。

使用有界队列，当任务过多时，线程池会触发执行拒绝策略，线程池默认的拒绝策略会 throw RejectedExecutionException 这是个运行时异常，对于运行时异常编译器并不强制 catch 它，所以开发人员很容易忽略。因此默认拒绝策略要慎重使用。如果线程池处理的任务非常重要，建议自定义自己的拒绝策略；并且在实际工作中，自定义的拒绝策略往往和降级策略配合使用。

使用线程池，还要注意异常处理的问题，例如通过 ThreadPoolExecutor 对象的 execute() 方法提交任务时，如果任务在执行的过程中出现运行时异常，会导致执行任务的线程终止；不过，最致命的是任务虽然异常了，但是你却获取不到任何通知，这会让你误以为任务都执行得很正常。虽然线程池提供了很多用于异常处理的方法，但是最稳妥和简单的方案还是捕获所有异常并按需处理，你可以参考下面的示例代码。

```java
try {
  //业务逻辑
} catch (RuntimeException x) {
  //按需处理
} catch (Throwable x) {
  //按需处理
} 
```

## 总结

线程池在 Java  并发编程领域非常重要，很多大厂的编码规范都要求必须通过线程池来管理线程。线程池和普通的池化资源有很大不同，线程池实际上是生产者 -  消费者模式的一种实现，理解生产者 - 消费者模式是理解线程池的关键所在。

创建线程池设置合适的线程数非常重要，这部分内容，你可以参考《10 | Java 线程（中）：创建多少线程才是合适的？》的内容。另外《Java 并发编程实战》的第 7 章《取消与关闭》的 7.3  节“处理非正常的线程终止” 详细介绍了异常处理的方案，第 8 章《线程池的使用》对线程池的使用也有更深入的介绍，如果你感兴趣或有需要的话，建议你仔细阅读。

## 线程的取消与关闭

在大多数时候, 我们都会让线程运行到结束, 或者让它自行停止。但是, 有时我们希望任务提前结束, 或许因为用户取消了操作, 或许应用程序需要被快速关闭。

在 Java 中使线程安全、快速、可靠的停止下来, 并不是一件容易的事。Java 没有提供任何机制来安全的终止线程。但它提供了中断(Interruption), 这是一种协作机制, 能够使一个线程终止另一个线程的当前工作。

### 任务取消

如果外部代码能够在某个操作正常完成之前将其置入"完成"状态, 那么这个操作就可以称为是可取消的(Cancellable)。取消某个操作的原因有很多种:

1. 用户请求取消
2. 有限时间的操作
3. 错误
4. 关闭

Java 中没有一种很安全的抢占方式来停止线程, 因此也就没有安全的方式来停止任务。只有一些协作式的机制, 使请求取消的任务和代码都遵循一种协商好的协议。

其中一种协作机制能设置某个"已请求取消(Cancellation Requested)"标志, 而任务将定期的查看该标志。如果设置了这个标志, 那么任务将提前结束。

```java
public class PrimeGenerator implements Runnable {

  @GuardedBy("this")
  private final List<BigInteger> primes = new ArrayList<>();
  private volatile boolean cancelled;

  @Override
  public void run() {
    BigInteger p = BigInteger.ONE;
    while (!cancelled) {
      p = p.nextProbablePrime();
      synchronized (this) {
        primes.add(p);
      }
    }
  }

  public void cancell(){
    cancelled = true;
  }

  public List<BigInteger> get() {
    return Collections.unmodifiableList(primes);
  }

  public static void main(String[] args) {

    PrimeGenerator generator = new PrimeGenerator();
    new Thread(generator).start();
    try {
      TimeUnit.SECONDS.sleep(1);
    }catch (InterruptedException e) {
      e.printStackTrace();
    }finally {
      generator.cancelled = true;
    }

    System.out.println(generator.get());
  }

}
```

上面代码给出了这个类的使用示例。即让素数生成器运行 1 秒后取消任务。但实际任务不会刚刚好运行 1 秒钟停止, 因为在请求取消的时刻和 run 中的方法循环下一次检查中间可能存在延迟。cancel方法由 finally 块调用, 确保即使在调用 sleep 时被中断也能取消素数生成器的执行。

一个可取消的任务必须拥有取消策略(Cancellation Policy), 在这个策略中将详细的定义取消操作的"How", "When" 以及 "What", 即其他代码如何(How)请求取消该任务, 任务在何时(When)检查是否已经取消了请求, 以及在响应取消请求时应该执行哪些(What)操作

### 中断

`PrimeGenerator` 中取消机制最终会使得搜搜素数的任务退出, 但在退出过程中需要花费一定的时间。然而, 如果使用这种方法的任务调用了一个阻塞方法, 例如 `BlockingQueue.put`, 那么可能会产生一个更严重的问题--任务可能永远不会检查取消标志, 因此永远不会结束。

线程中断是一种协作机制, 线程可以通过这种机制来通知另一个线程, 告诉他在合适的时间或者可能的情况下停止当前工作,转而执行其他的工作。

> 在 Java 的 API 或者与严规范中, 并没有将中断与任何取消语义关联起来, 但实际上, 如果在取消之外的其他操作中使用中断, 那么都是不合适的, 并且很难支撑其更大的应用。

每个线程中都有一个 boolean 类型的中断状态。当线程中断时, 这个线程的中断状态将被置位 true。在 Thread 中包含了中断线程以及查询线程中断状态的方法,如下:

```java
public class Thread{
	public void interrupt(){......}
  public boolean isInterrupted(){......}
  public static boolean interrupted(){......}
}
```

- `interrupt()`方法能中断目标线程
- `isInterrupted()`方法能返回目标线程的中断状态
- `interrupted()`方法将清除当前线程的中断状态, 并返回它之前的值, 这也是清除中断状态的唯一方法

阻塞库的方法, 如 `Thread.sleep()` 和 `Object.wait()`等, 都会检查线程何时中断, 并在发现中断时提前返回。它在响应中断时执行的操作包括: 清除中断状态, 抛出`InterruptedException`, 表示阻塞操作由于中断而提前结束。JVM 并不保证阻塞方法检测到中断的速度, 但在实际情况中响应的速度还是非常快的。

对中断操作的正确理解是: 他并不会真正的中断一个正在运行的线程, 而只是发出中断请求, 然后由线程在下一个合适的时刻中断自己。(这个时刻也被称为取消点)。有些方法, 例如 `wait`, `sleep` 和 `join`, 将严格的处理这种请求, 当他们收到中断请求或者在开始执行时发现某个已经被设置好的中断状态时, 将抛出一个异常。

使用 static interrupted 时应该特别小心, 因为它会清除当前线程的中断状态。如果在调用 interrupted 时返回 true, 除非想屏蔽这个中断, 否则必须对它进行处理。

> 中断是实现取消的最合适的方式

```java
public class PrimeProducer implements Runnable {

  private Thread currentThread;
  private final BlockingQueue<BigInteger> queue;

  public PrimeProducer(BlockingQueue<BigInteger> queue) {
    this.queue = queue;
  }

  @Override
  public void run() {
    currentThread = Thread.currentThread();
    try {
      BigInteger p = BigInteger.ONE;
      while (!Thread.currentThread().isInterrupted()) {
        queue.put(p = p.nextProbablePrime());
      }
    } catch (InterruptedException e) {
      System.out.printf("Thread:[%s] was interrupted......\n", currentThread.getName());
    }
  }

  public void cancel() {
    if (currentThread != null) {
      System.out.printf("call cancel() in Thread:[%s]\n", currentThread.getName());
      currentThread.interrupt();
    }
  }

  public static void main(String[] args) {

    BlockingQueue<BigInteger> queue = new ArrayBlockingQueue<BigInteger>(50);
    PrimeProducer primeProducer = new PrimeProducer(queue);
    new Thread(primeProducer, "miao-miao").start();

    try {
      TimeUnit.SECONDS.sleep(1);
    }catch (InterruptedException e) {
      e.printStackTrace();
    }finally {
      primeProducer.cancel();
    }

    System.out.println(queue);
  }
}
```

### 中断策略

中断策略规定线程如何解释某个中断请求--当发现中断请求时, 应该做哪些工作, 哪儿工作单元对于中断来说是原子操作, 以及以对快的速度来响应中断。

最合理的中断策略是以某种形式的线程级取消操作或服务级取消操作: 尽快退出, 在必要的时候进行清理, 通知某个所有者该线程已经退出。此外还可以建立其他中断策略, 例如: 暂停服务或重新开始服务

### 响应中断

当调用可中断的阻塞函数库时, 如 sleep, wait, 有两种实用策略可用于处理 `InterruptedException` :

- 传递异常(可能在执行某个特定于任务的清除操作之后), 从而使你的方法也成为可中断的阻塞方法
- 恢复中断状态, 从而使调用栈中的上层代码能够对其进行处理













## 课后思考

使用线程池，默认情况下创建的线程名字都类似pool-1-thread-2这样，没有业务含义。而很多情况下为了便于诊断问题，都需要给线程赋予一个有意义的名字，那你知道有哪些办法可以给线程池里的线程指定名字吗？

>1.利用guava的ThreadFactoryBuilder
>2.自己实现ThreadFactory

