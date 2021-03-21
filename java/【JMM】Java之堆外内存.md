# 【JMM】堆外内存

#Lang/Java

### OverView

在JAVA中，JVM内存指的是堆内存。

机器内存中，不属于堆内存的部分即为堆外内存。

堆外内存也称为直接内存。在C语言中，分配的就是机器内存 ( 因为C没有引入GC机制来自动回收垃圾, 需要开发人员自己回收垃圾, 释放内存 )，和本文中堆外内存是相似的概念。

在JAVA中，可以通过 Unsafe 和 NIO 包下的ByteBuffer来操作堆外内存。

在讲解DirectByteBuffer之前，需要先简单了解两个知识点。

#### Java引用类型

java引用类型，因为 DirectByteBuffe r是通过虚引用 (Phantom Reference) 来实现堆外内存的释放的。

PhantomReference 是所有“弱引用”中最弱的引用类型。不同于软引用和弱引用，虚引用无法通过 get() 方法来取得目标对象的强引用从而使用目标对象，观察源码可以发现 get() 被重写为永远返回 null。

那虚引用到底有什么作用？其实虚引用主要被用来 跟踪对象被垃圾回收的状态，通过查看引用队列中是否包含对象所对应的虚引用来判断它是否 即将被垃圾回收，从而采取行动。它并不被期待用来取得目标对象的引用，而目标对象被回收前，它的引用会被放入一个 ReferenceQueue 对象中，从而达到跟踪对象垃圾回收的作用。

Java引用可详见 [Java引用引出的ThreadLocal安全问题](file:///Users/sherlock/Desktop/notes/java/Java引用引出的ThreadLocal安全问题.md)

#### linux的内核态和用户态

- 内核态：控制计算机的硬件资源，并提供上层应用程序运行的环境。比如socket I/0操作或者文件的读写操作等
- 用户态：上层应用程序的活动空间，应用程序的执行必须依托于内核提供的资源。
- 系统调用：为了使上层应用能够访问到这些资源，内核为上层应用提供访问的接口。

因此我们可以得知当我们通过JNI调用的native方法实际上就是从用户态切换到了内核态的一种方式。并且通过该系统调用使用操作系统所提供的功能。

> Q：为什么需要用户进程(位于用户态中)要通过系统调用(Java中即使JNI)来调用内核态中的资源，或者说调用操作系统的服务了？
>
> A：intel cpu提供Ring0-Ring3四种级别的运行模式，Ring0级别最高，Ring3最低。Linux使用了Ring3级别运行用户态，Ring0作为内核态。Ring3状态不能访问Ring0的地址空间，包括代码和数据。因此用户态是没有权限去操作内核态的资源的，它只能通过系统调用外完成用户态到内核态的切换，然后在完成相关操作后再有内核态切换回用户态。

### 堆外内存的创建

DirectByteBuffer是Java用于实现堆外内存的一个重要类，我们可以通过该类实现堆外内存的创建、使用和销毁。

DirectByteBuffer该类本身还是位于Java内存模型的堆中。堆内内存是JVM可以直接管控、操纵。

而 DirectByteBuffer中的 unsafe.allocateMemory(size); 是个一个native方法，这个方法分配的是堆外内存，通过C的malloc来进行分配的。分配的内存是系统本地的内存，并不在Java的内存中，也不属于JVM管控范围，所以在DirectByteBuffer一定会存在某种方式来操纵堆外内存。

unsafe.allocateMemory(size);分配完堆外内存后就会返回分配的堆外内存基地址，并将这个地址赋值给了address属性。这样我们后面通过JNI对这个堆外内存操作时都是通过这个address来实现的了。

在前面我们说过，在linux中内核态的权限是最高的，那么在内核态的场景下，操作系统是可以访问任何一个内存区域的，所以操作系统是可以访问到Java堆的这个内存区域的。

> Q：***那为什么操作系统不直接访问Java堆内的内存区域了？***
>
> A：这是因为JNI方法访问的内存区域是一个已经确定了的内存区域地质，那么该内存地址指向的是Java堆内内存的话，那么如果在操作系统正在访问这个内存地址的时候，Java在这个时候进行了GC操作，而GC操作会涉及到数据的移动操作[GC经常会进行先标志在压缩的操作。即，将可回收的空间做标志，然后清空标志位置的内存，然后会进行一个压缩，压缩就会涉及到对象的移动，移动的目的是为了腾出一块更加完整、连续的内存空间，以容纳更大的新对象]，数据的移动会使JNI调用的数据错乱。所以JNI调用的内存是不能进行GC操作的。
>
> Q：***如上面所说，JNI调用的内存是不能进行GC操作的，那该如何解决了？***
>
> A：
>
> 1. 堆内内存与堆外内存之间数据拷贝的方式(并且在将堆内内存拷贝到堆外内存的过程JVM会保证不会进行GC操作)：比如我们要完成一个从文件中读数据到堆内内存的操作，即***FileChannelImpl.read(HeapByteBuffer)***。这里实际上File I/O会将数据读到堆外内存中，然后堆外内存再讲数据拷贝到堆内内存，这样我们就读到了文件中的内存。而写操作则反之，我们会将堆内内存的数据线写到对堆外内存中，然后操作系统会将堆外内存的数据写入到文件中。
>
> 2. 直接使用堆外内存，如DirectByteBuffer：这种方式是直接在堆外分配一个内存(即，native memory)来存储数据，程序通过JNI直接将数据读/写到堆外内存中。因为数据直接写入到了堆外内存中，所以这种方式就不会再在JVM管控的堆内再分配内存来存储数据了，也就不存在堆内内存和堆外内存数据拷贝的操作了。这样在进行I/O操作时，只需要将这个堆外内存地址传给JNI的I/O的函数就好了。

```java
DirectByteBuffer(int cap) {  // package-private
  super(-1, 0, cap, cap);
	boolean
	pa = VM.isDirectMemoryPageAligned();
	int ps = Bits.pageSize();
	long size = Math.max(1L, (long)cap + (pa ? ps : 0));
	
	// 保留总分配内存(按页分配)的大小和实际内存的大小
  Bits.reserveMemory(size, cap);

	long base = 0;

  try{
		// 通过unsafe.allocateMemory分配堆外内存，并返回堆外内存的基地址
		base = unsafe.allocateMemory(size);
	} catch(OutOfMemoryError x) {
	
		Bits.unreserveMemory(size, cap);
		throw x;
  }
	unsafe.setMemory(base, size, (byte) 0);
	
  if(pa && (base % ps != 0)) {
	
		// Round up to page boundary
		address = base + ps - (base & (ps - 1));
	} else {
		address = base;
	}

	// 构建Cleaner对象用于跟踪DirectByteBuffer对象的垃圾回收，以实现当DirectByteBuffer被垃圾回收时，堆外内存也会被释放
	cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
	att = null;

}
```

### 堆外内存垃圾回收

Cleaner是PhantomReference的子类，并通过自身的next和prev字段维护的一个双向链表。PhantomReference的作用在于跟踪垃圾回收过程，并不会对对象的垃圾回收过程造成任何的影响。
所以cleaner = Cleaner.create(this, new Deallocator(base, size, cap)); 用于对当前构造的DirectByteBuffer对象的垃圾回收过程进行跟踪。
当DirectByteBuffer对象从pending状态 ——> enqueue状态时，会触发Cleaner的clean()，而Cleaner的clean()的方法会实现通过unsafe对堆外内存的释放。

[![4235178-792afac32aefd061](http://incdn1.b0.upaiyun.com/2017/08/71fb39c74b5e95097d5ce669a911fcb2.png)](http://www.importnew.com/?attachment_id=26342)[![4235178-07eaab88f1d02927](http://incdn1.b0.upaiyun.com/2017/08/55becba84de6902babbb03b6bfe17e18.png)](http://www.importnew.com/?attachment_id=26343) 

虽然Cleaner不会调用到Reference.clear()，但Cleaner的clean()方法调用了remove(this)，即将当前Cleaner从Cleaner链表中移除，这样当clean()执行完后，Cleaner就是一个无引用指向的对象了，也就是可被GC回收的对象。

1. 堆外内存会溢出么？
2. 什么时候会触发堆外内存回收？

通过修改JVM参数：-XX:MaxDirectMemorySize=40M，将最大堆外内存设置为40M。

既然堆外内存有限，则必然会发生内存溢出。

设置JVM参数：-XX:+DisableExplicitGC，禁止代码中显式调用System.gc()。可以看到出现OOM。

得到的结论是，堆外内存会溢出，并且其垃圾回收依赖于代码显式调用System.gc()。

