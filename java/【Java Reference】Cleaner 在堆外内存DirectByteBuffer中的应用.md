# 【Java Reference】Cleaner 在堆外内存DirectByteBuffer中的应用

#Lang/Java

### 一、DirectByteBuffer 架构

#### 1.1 代码UML

![directbytebuffer](/Users/sherlock/Desktop/notes/allPics/Java/directbytebuffer.png)

#### 1.2 申请内存Flow图

![allocateDirect](/Users/sherlock/Desktop/notes/allPics/Java/allocateDirect.png)

### 二、DirectByteBuffer 实战 Demo

#### 2.1 使用 ByteBuffer.allocateDirect 申请堆外内存

```java
package cgs.java.nio.unsafe;

import org.junit.Test;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.nio.ByteBuffer;

/**
 * Descriptor:
 * Author: sherlock
 */
public class DirectByteBufferTest {

    private static final int _1_MB =  1 * 1024 * 2014;

    @Vm("-verbose:gc -XX:+PrintGCDetails -XX:MaxDirectMemorySize=20m")
    @Test
    public void allocate_direct_memory() {
        while (true) {
            ByteBuffer.allocateDirect(_1_MB);
        }
    }

    @Vm("-verbose:gc -XX:+PrintGCDetails -XX:MaxDirectMemorySize=20m -XX:+DisableExplicitGC")
    @Test
    public void allocate_direct_memory_no_gc() {
        while (true) {
            ByteBuffer.allocateDirect(_1_MB);
        }
    }

    
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.CLASS)
    private @interface Vm {

        public String value();
    }

}
```

> 结果： 设置堆内存最大20m，但是代码完全不停歇，没有 OOM 发生。
>
> 为什么不会 OOM 呢 ？
> 把 GC 信息打印出来 ： @VM -verbose:gc -XX:+PrintGCDetails

```scala
[GC (System.gc()) [PSYoungGen: 665K->64K(38400K)] 1686K->1085K(125952K), 0.0005744 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 64K->0K(38400K)] [ParOldGen: 1021K->1021K(87552K)] 1085K->1021K(125952K), [Metaspace: 5242K->5242K(1056768K)], 0.0079472 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 38400K, used 1996K [0x0000000795580000, 0x0000000798000000, 0x00000007c0000000)
  eden space 33280K, 6% used [0x0000000795580000,0x0000000795773370,0x0000000797600000)
  from space 5120K, 0% used [0x0000000797b00000,0x0000000797b00000,0x0000000798000000)
  to   space 5120K, 0% used [0x0000000797600000,0x0000000797600000,0x0000000797b00000)
 ParOldGen       total 87552K, used 1021K [0x0000000740000000, 0x0000000745580000, 0x0000000795580000)
  object space 87552K, 1% used [0x0000000740000000,0x00000007400ff4c0,0x0000000745580000)
 Metaspace       used 5250K, capacity 5408K, committed 5504K, reserved 1056768K
  class space    used 602K, capacity 627K, committed 640K, reserved 1048576K
```



#### 2.2 加上 -XX:+DisableExplicitGC 后

这个参数效果是禁止 System.gc()，也就是禁止主动调用垃圾回收

测试上述代码中的 ***allocate_direct_memory_no_gc ()*** 方法

```scala
java.lang.OutOfMemoryError: Direct buffer memory

	at java.nio.Bits.reserveMemory(Bits.java:694)
	at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:123)
	at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:311)
	......

Heap
 PSYoungGen      total 38400K, used 9995K [0x0000000795580000, 0x0000000798000000, 0x00000007c0000000)
  eden space 33280K, 30% used [0x0000000795580000,0x0000000795f42c48,0x0000000797600000)
  from space 5120K, 0% used [0x0000000797b00000,0x0000000797b00000,0x0000000798000000)
  to   space 5120K, 0% used [0x0000000797600000,0x0000000797600000,0x0000000797b00000)
 ParOldGen       total 87552K, used 0K [0x0000000740000000, 0x0000000745580000, 0x0000000795580000)
  object space 87552K, 0% used [0x0000000740000000,0x0000000740000000,0x0000000745580000)
 Metaspace       used 5304K, capacity 5440K, committed 5632K, reserved 1056768K
  class space    used 610K, capacity 659K, committed 768K, reserved 1048576K
```

> 结论：禁止 System.gc()，JVM 内存不足了，为什么会这样呢？？？
> 那么肯定是 allocateDirect 有调用 System.gc()。



### 三、DirectByteBuffer 源码剖析

#### 3.1 allocateDirect 方法

- ByteBuffer#allocateDirect 方法

  ```java
  public static ByteBuffer allocateDirect(int capacity) {
  	return new DirectByteBuffer(capacity);
  }
  ```

#### 3.2 构造方法 DirectByteBuffer(int)

- DirectByteBuffer#DirectByteBuffer(int) 方法

  ```java
  DirectByteBuffer(int cap) {                   // package-private
          super(-1, 0, cap, cap);
           // 计算需要分配的内存大小
          boolean pa = VM.isDirectMemoryPageAligned();
          int ps = Bits.pageSize();
          long size = Math.max(1L, (long)cap + (pa ? ps : 0));
          //=== 3.3 告诉内存管理器要分配内存
          Bits.reserveMemory(size, cap);
  
          long base = 0;
          try {
          	// 3.4 分配直接内存
              base = unsafe.allocateMemory(size);
          } catch (OutOfMemoryError x) {
  	        // 3.5 通知 bits 释放内存
              Bits.unreserveMemory(size, cap);
              throw x;
          }
          unsafe.setMemory(base, size, (byte) 0);
          //  计算内存的地址
          if (pa && (base % ps != 0)) {
              // Round up to page boundary
              address = base + ps - (base & (ps - 1));
          } else {
              address = base;
          }
          // 3.6 创建Cleaner!!!! 重点讲这个
          cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
          att = null;
      }
  ```

#### 3.3 reserveMemory 方法

- Bits#reserveMemory 方法

  ```java
  static void reserveMemory(long size, int cap) {
          if (!memoryLimitSet && VM.isBooted()) {
          	// 初始化maxMemory，就是使用-XX:MaxDirectMemorySize指定的最大直接内存大小
              maxMemory = VM.maxDirectMemory();
              memoryLimitSet = true;
          }
  
          //=== 3.3.1 第一次先采取最乐观的方式直接尝试告诉Bits要分配内存
          if (tryReserveMemory(size, cap)) {
              return;
          }
          // 内存获取失败 往下走。。。
          
          // === 3.3.2 获取 JavaLangRefAccess
          final JavaLangRefAccess jlra = SharedSecrets.getJavaLangRefAccess();
  
          // retry while helping enqueue pending Reference objects
          // which includes executing pending Cleaner(s) which includes
          // Cleaner(s) that free direct buffer memory
          /*
          	tryHandlePendingReference方法：消耗 pending 队列，1. 丢到 Enqueue队列，2. 调用 cleaner.clean() 方法释放内存。
          	=== 3.3.3 尝试多次获取
          	失败：tryHandlePendingReference方法的 pending 队列完尽
          	成功：释放了空间，tryReserveMemory 成功
          */
          while (jlra.tryHandlePendingReference()) { // 这个地方返回 false ，也就是 pending 队列完尽，就返回 false
              if (tryReserveMemory(size, cap)) {
                  return;
              }
          }
  
          // trigger VM's Reference processing
          //=== 3.3.4 Full GC 看到没有！！！！ System.gc(）在这
          System.gc();
  
          // a retry loop with exponential back-off delays
          // (this gives VM some time to do it's job)
          // === 3.3.5 9次循环，不断延迟要求分配内存
          boolean interrupted = false;
          try {
              long sleepTime = 1;
              int sleeps = 0;
              // 不断*2，按照1ms,2ms,4ms,...,256ms的等待间隔尝试9次分配内存
              while (true) {
                  if (tryReserveMemory(size, cap)) {
                      return;
                  }
                  if (sleeps >= MAX_SLEEPS) { //最多循环9次， final int MAX_SLEEPS = 9;
                      break;
                  }
                  if (!jlra.tryHandlePendingReference()) {
                      try {
                          Thread.sleep(sleepTime);
                          sleepTime <<= 1; // 左边移动一位，也就 * 2
                          sleeps++; 
                      } catch (InterruptedException e) {
                          interrupted = true;
                      }
                  }
              }
  
              // no luck
              throw new OutOfMemoryError("Direct buffer memory");
  
          } finally {
              if (interrupted) {
                  // don't swallow interrupts
                  Thread.currentThread().interrupt();
              }
          }
      }
  ```

  

##### 3.3.1 tryReserveMemory 方法

- Bits#tryReserveMemory 方法

  ```java
  // -XX:MaxDirectMemorySize限制的是总cap，而不是真实的内存使用量，(在页对齐的情况下，真实内存使用量和总cap是不同的)
  private static boolean tryReserveMemory(long size, int cap) {
          // -XX:MaxDirectMemorySize limits the total capacity rather than the
          // actual memory usage, which will differ when buffers are page
          // aligned.
          long totalCap;
          while (cap <= maxMemory - (totalCap = totalCapacity.get())) {
              if (totalCapacity.compareAndSet(totalCap, totalCap + cap)) {
                  reservedMemory.addAndGet(size);
                  count.incrementAndGet();
                  return true;
              }
          }
  
          return false;
      }
  ```

##### 3.3.2 getJavaLangRefAccess 方法

- SharedSecrets.getJavaLangRefAccess()
  这个是个啥呀？

  ```java
  public static JavaLangRefAccess getJavaLangRefAccess() {
          return javaLangRefAccess;
      }
  ```

  ![setIn](/Users/sherlock/Desktop/notes/allPics/Java/setIn.png)

  原来是存在与 Reference 类的 static 块里面



##### 3.3.3 tryHandlePendingReference 方法

- JavaLangRefAccess#tryHandlePendingReference 方法

  ```java
  package sun.misc;
  
  public interface JavaLangRefAccess {
  
      /**
       * Help ReferenceHandler thread process next pending
       * {@link java.lang.ref.Reference}
       *
       * @return {@code true} if there was a pending reference and it
       *         was enqueue-ed or {@code false} if there was no
       *         pending reference
       */
      boolean tryHandlePendingReference();
  }
  ```

  看注释，该方法协助 ReferenceHandler内部线程进行下一个 pending 的处理，内部主要是希望遇到 Cleaner，然后调用 c.clean(); 进行堆外内存的释放。
  从上一个方法== 3.3.2 getJavaLangRefAccess 方法 ==可以知道，该方法已经被 @Override，具体实现是 === java.lang.ref.Reference#tryHandlePending 方法 ===
  ![handlePending](/Users/sherlock/Desktop/notes/allPics/Java/handlePending.png)

  ```java
  while (jlra.tryHandlePendingReference()) { // 这个地方返回 false ，也就是 pending 队列没有了，就返回 false
  	if (tryReserveMemory(size, cap)) {
  		return;
  	}
  }
  
  ```

  while 能够把当前全部 pending 队列中的 reference 都消化掉，要么Enqueue，要么Cleaner去进行 clean() 操作。

  === 3.3.3 while 死循环尝试申请内存
  tryHandlePendingReference方法：消耗 pending 队列，1. 丢到 Enqueue队列，2. 调用 cleaner.clean() 方法释放内存。
  失败：tryHandlePendingReference方法的 pending 队列完尽
  成功：释放了空间，tryReserveMemory 成功
  只有这样消耗光了 pending，才会往下走 === 3.3.4 System.gc() == ；

##### 3.3.4 System.gc()

假如上述的步骤还是没能释放内存的话，那么将会触发 Full GC。但我们知道，调用System.gc()并不能够保证full gc马上就能被执行。所以在后面代码中，会进行最多MAX_SLEEPS = 9次尝试，看是否有足够的可用堆外内存来分配堆外内存。并且每次尝试之前，都对延迟（*2）等待时间，已给JVM足够的时间去完成full gc操作。
这个地方要注意：如果设置了-XX:+DisableExplicitGC，将会禁用显示GC，这会使System.gc()调用无效。

##### 3.3.5 获取内存的9次尝试

- 如果9次尝试后依旧没有足够的可用堆外内存来分配本次堆外内存，则 throw new OutOfMemoryError(“Direct buffer memory”);

#### 3.4 allocateMemory 方法

![allocate](/Users/sherlock/Desktop/notes/allPics/Java/allocate.png)

- sun.misc.Unsafe#allocateMemory

  ```java
  public native long allocateMemory(long bytes);
  ```

- unsafe

  ```java
  class DirectByteBuffer extends MappedByteBuffer implements DirectBuffer{
      // Cached unsafe-access object
      protected static final Unsafe unsafe = Bits.unsafe();
  }
  ```

- java.nio.Bits#unsafe

  ```java
  class Bits {   
  	private static final Unsafe unsafe = Unsafe.getUnsafe();
  	static Unsafe unsafe() {
     		return unsafe;
  	}
  }
  ```

- sun.misc.Unsafe#getUnsafe

  ```java
  @CallerSensitive
  public static Unsafe getUnsafe() {
    Class<?> caller = Reflection.getCallerClass();
    if (!VM.isSystemDomainLoader(caller.getClassLoader()))
      throw new SecurityException("Unsafe");
    return theUnsafe;
  }
  ```

- 总结： ByteBuffer.allocateDirect(int capacity) 的底层是 unsafe.allocateMemory()

##### 3.4.1 Unsafe类

Java提供了Unsafe类用来进行直接内存的分配与释放

```java
public native long allocateMemory(long var1);
public native void freeMemory(long var1);
```

#### 3.5 unreserveMemory 方法

- java.nio.Bits#unreserveMemory

```java

static void unreserveMemory(long size, int cap) {
  long cnt = count.decrementAndGet();
  long reservedMem = reservedMemory.addAndGet(-size);
  long totalCap = totalCapacity.addAndGet(-cap);
  assert cnt >= 0 && reservedMem >= 0 && totalCap >= 0;
}
```

#### 3.6 构造 Cleaner对象

```java
cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
```

这个create静态方法提供给我们来实例化Cleaner对象，需要两个参数：

- 被引用的对象
- 实现了Runnable接口的对象，这个用来回调的时候执行内部的 run 方法

新创建的Cleaner对象被加入到了 dummyQueue 队列里。

##### 3.6.1 Deallocator 对象

- 内部类 java.nio.DirectByteBuffer.Deallocator

```java
private static class Deallocator
        implements Runnable{    
		private static Unsafe unsafe = Unsafe.getUnsafe();

    private long address;
    private long size;
    private int capacity;

    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }

    public void run() {
    	// 堆外内存已经被释放了
        if (address == 0) {
            // Paranoia
            return;
        }
        // 3.6.2 调用C++代码释放堆外内存
        unsafe.freeMemory(address);
        address = 0; // 设置为0，表示已经释放了
        // 刚刚的 3.5 释放后，标记资源
        Bits.unreserveMemory(size, capacity);
    }

}
```
##### 3.6.2 freeMemory 方法

```java
 /**
     * Disposes of a block of native memory, as obtained from {@link
     * #allocateMemory} or {@link #reallocateMemory}.  The address passed to
     * this method may be null, in which case no action is taken.
     *
     * @see #allocateMemory
     */
    public native void freeMemory(long address);
```

### 四、总结

#### 4.1 tryHandlePendingReference 的调用场景

1. 幽灵线程死循环调用 看下：【JAVA Reference】ReferenceQueue 与 Reference 源码剖析（二）# Reference的static代码块
2. 申请内存的时候，会调用（本文）

#### 4.2 堆外缓存的特点

1. 对垃圾回收停顿的改善可以明显感觉到
2. 对于大内存有良好的伸缩性
3. 在进程间可以共享，减少虚拟机间的复制
4. netty 就是使用堆外缓存，可以减少数据的复制操作，提高性能

#### 4.3 使用堆外内存的原因

1. 可以一定程度改善垃圾回收停顿的影响。full gc 意味着彻底回收，过大的堆会影响Java应用的性能。如果使用堆外内存的话，堆外内存是直接受操作系统管理( 而不是JVM )。
2. 在某些场景下可以提升程序I/O操纵的性能。减少了将数据从堆内内存拷贝到堆外内存的步骤。

#### 4.4 对外内存的使用场景

1. 直接的文件拷贝操作，或者I/O操作。直接使用堆外内存就能少去内存从用户内存拷贝到系统内存的操作，因为I/O操作是系统内核内存和设备间的通信，而不是通过程序直接和外设通信的。
2. 堆外内存适用于生命周期中等或较长的对象。( 如果是生命周期较短的对象，在YGC的时候就被回收了，就不存在大内存且生命周期较长的对象在FGC对应用造成的性能影响 )。

同时，还可以使用池+堆外内存 的组合方式，来对生命周期较短，但涉及到I/O操作的对象进行堆外内存的再使用。( Netty中就使用了该方式 )





















