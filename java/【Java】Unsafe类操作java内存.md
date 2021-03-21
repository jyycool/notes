# 【Java】Unsafe类操作java内存

#Lang/Java

本文主要介绍Java中几种分配内存的方法。我们会看到如何使用sun.misc.Unsafe来统一操作任意类型的内存。以前用C语言开发的同学通常都希望能在Java中通过较底层的接口来操作内存，他们一定会对本文中要讲的内容感兴趣

## Overview

查看Unsafe的源码我们会发现它提供了一个`getUnsafe()`的静态方法。

```java
@CallerSensitive
public static Unsafe getUnsafe() {
  Class var0 = Reflection.getCallerClass();
  if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
   	throw new SecurityException("Unsafe");
  } else {
    return theUnsafe;
  }
}
```

但是，如果直接调用这个方法会抛出一个 ***SecurityException*** 异常，这是因为Unsafe仅供java内部类使用，外部类不应该使用它。

那么，我们就没有方法了吗？

当然不是，我们有反射啊！查看源码，我们发现它有一个属性叫 theUnsafe，我们直接通过反射拿到它即可。

```java
package com.github.sherlock.java.nio.unsafe;

import sun.misc.Unsafe;
import java.lang.reflect.Field;
/**
 * Descriptor:
 * Author: sherlock
 *
 *  Unsafe.getUnsafe() 这个方法获取不到, 会抛出SecurityException异常,
 *  Unsafe实例仅供Java内部类使用, 但可以通过反射来获取Unsafe实例.
 */
public class ManageMemory {

  // 获取不到实例, 且抛出 java.lang.SecurityException: Unsafe 异常
  //    private static final Unsafe unsafe = Unsafe.getUnsafe();

  private static final Unsafe unsafe;

  static {
    try {
      Field field = Unsafe.class.getDeclaredField("theUnsafe");
      field.setAccessible(true);
      unsafe = (Unsafe)field.get(null);
    }
    catch (Exception e)
    {
      throw new RuntimeException(e);
    }
  }

  public static void main(String[] args) {

    // 获取到数组的首元素偏移地址
    long longArrayOffset = unsafe.arrayBaseOffset(long[].class);
    System.out.printf("longArrayOffset: [%d]\n", longArrayOffset);

    final long[] arr = new long[ 1000 ];
    int index = arr.length - 1;
    // FFFF FFFF FFFF FFFF
    arr[index] = -1;

    System.out.printf( "Before change, arr[%d] = %s \n", index, Long.toHexString(arr[ index ]));

    for (int i = 0; i < 8; i++) {
      unsafe.putByte(arr, longArrayOffset + 8L * index + i, (byte)0);
    }
  }
}
```

### 一、数组元素的定位

Unsafe类中有很多以 ***BASE_OFFSET*** 结尾的常量，比如 ***ARRAY_INT_BASE_OFFSET***，***ARRAY_BYTE_BASE_OFFSET*** 等，这些常量值是通过***arrayBaseOffset*** 方法得到的。***arrayBaseOffset*** 方法是一个本地方法，可以获取数组第一个元素的偏移地址。

Unsafe类中还有很多以 ***INDEX_SCALE*** 结尾的常量，比如 ***ARRAY_INT_INDEX_SCALE*** ， ***ARRAY_BYTE_INDEX_SCALE*** 等，这些常量值是通过 ***arrayIndexScale*** 方法得到的。***arrayIndexScale*** 方法也是一个本地方法，可以获取数组的转换因子，也就是数组中元素的增量地址。

将 ***arrayBaseOffset*** 与 ***arrayIndexScale*** 配合使用，可以定位数组中每个元素在内存中的位置。

```java
public static void main(String[] args) {
  /*
  	基本数据类型数组:
      boolean[]:  [Z, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [1]
      byte[]:			[B, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [1]
      short[]: 		[S, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [2]
      char[]: 		[C, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [2]
      int[]:			[I, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [4]
      float[]:		[F, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [4]
      long[]:			[J, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [8]
      double[]:		[D, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [8]
               
    引用类型数组:
    	String[], Integer[], Double[],......,它们的数组中元素的增量地址: [4]
    
    这也从侧面反映出, 一个对象的引用的长度就是4byte
  */

  Class[] clazzs = new Class[8];
	clazzs[0] = boolean[].class;
	clazzs[1] = byte[].class;
  clazzs[2] = short[].class;
  clazzs[3] = char[].class;
  clazzs[4] = int[].class;
  clazzs[5] = float[].class;
  clazzs[6] = long[].class;
  clazzs[7] = double[].class;

  Stream.of(clazzs).forEach(clazz -> {
		int longBaseOffset = unsafe.arrayBaseOffset(clazz);
 		int longIndexScale = unsafe.arrayIndexScale(clazz);
    System.out.printf("%s, 数组首部元素偏移地址: [%d], 数组中元素的增量地址: [%d]\n",
                clazz.getName(),
                longBaseOffset,
                longIndexScale);
  });

  // long[],数组的首元素偏移地址
  int longBaseOffset = Unsafe.ARRAY_LONG_BASE_OFFSET;
  // long[],数组中元素的增量地址
  int longIndexScale = Unsafe.ARRAY_LONG_INDEX_SCALE;
}
```



### 二、数组分配的上限

Java里数组的大小是受限制的，因为它使用的是int类型作为数组下标。这意味着你无法申请超过Integer.MAX_VALUE（2^31-1）大小的数组。这并不是说你申请内存的上限就是2G。你可以申请一个大一点的类型的数组。

```java
byte[] bytes = new byte[Integer.MAX_VALUE]; // 2G
long[] longs = new long[Integer.MAX_VALUE]; // 16G
```

这个会分配大约 16G - 8 字节，如果你设置的 -Xmx 参数足够大的话（通常你的堆至少得保留50%以上的空间，也就是说分配16G的内存，你得设置成-Xmx24G。这只是一般的规则，具体分配多大要看实际情况）。

不幸的是，在Java里，由于数组元素的类型的限制，你操作起内存来会比较麻烦。在操作数组方面，***ByteBuffer*** 应该是最有用的一个类了，它提供了读写不同的Java类型的方法。它的缺点是，目标数组类型必须是byte[]，也就是说你分配的内存缓存最大只能是2G。

### 三、所有数组都当作byte[]操作

假设现在2G内存对我们来说远远不够，如果是16G的话还算可以。我们已经分配了一个long[]，不过我们希望把它当作byte数组来进行操作。在Java里我们得求助下C程序员的好帮手了——***sun.misc.Unsafe***。这个类有两组方法：`getN(object, offset)`,这个方法是要从object偏移量为offset的位置获取一个指定类型的值并返回它，N在这里就是代表着那个要返回值的类型，而 `putN(Object, offset, value）`方法就是要把一个值写到Object的offset的那个位置。

不幸的是，这些方法只能获取或者设置某个类型的值。如果你从数组里拷贝数据，你还需要unsafe的另一个方法，`copyMemory(srcObject, srcOffset, destObject,destOffet,count)`。这和 System.arraycopy 的工作方式类似，不过它拷贝的是字节而不是数组元素。

想通过sun.misc.Unsafe来访问数组的数据，你需要两个东西：

1. 数组对象里数据的偏移量
2. 拷贝的元素在数组数据里的偏移量

Arrays和Java别的对象一样，都有一个对象头，它是存储在实际的数据前面的。

- 数组对象头的长度可以通过unsafe.arrayBaseOffset(T[].class)方法来获取到，这里T是数组元素的类型。
- 数组元素的大小可以通过unsafe.arrayIndexScale(T[].class) 方法获取到。这也就是说要访问类型为T的第N个元素的话，你的偏移量offset应该是arrayOffset+N*arrayScale。

我们来写个简单的例子吧。我们分配一个long数组，然后更新它里面的几个字节。我们把最后一个元素更新成-1（16进制的话是0xFFFF FFFF FFFF FFFF)，然再逐个清除这个元素的所有字节。

```java
package com.github.sherlock.java.nio.unsafe;

import sun.misc.Unsafe;

import java.lang.reflect.Field;
import java.util.stream.Stream;

/**
 * Descriptor:
 * Author: sherlock
 *
 *  Unsafe.getUnsafe() 这个方法获取不到, 会抛出SecurityException异常,
 *  Unsafe实例仅供Java内部类使用, 但可以通过反射来获取Unsafe实例.
 */
public class ManageMemory {

    // 获取不到实例, 且抛出 java.lang.SecurityException: Unsafe 异常
//    private static final Unsafe unsafe = Unsafe.getUnsafe();

    private static final Unsafe unsafe;

    static {
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            unsafe = (Unsafe)field.get(null);
        }
        catch (Exception e)
        {
            throw new RuntimeException(e);
        }
    }

    public static void main(String[] args) {
    /*
  		基本数据类型数组:
        boolean[]:  [Z, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [1]
        byte[]:			[B, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [1]
        short[]: 		[S, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [2]
        char[]: 		[C, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [2]
        int[]:			[I, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [4]
        float[]:		[F, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [4]
        long[]:			[J, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [8]
        double[]:		[D, 数组首部元素偏移地址: [16], 数组中元素的增量地址: [8]
               
    	引用类型数组:
    		String[], Integer[], Double[],......,它们的数组中元素的增量地址: [4]
    
    	这也从侧面反映出, 一个对象的引用的长度就是4byte
  	*/
		Class[] clazzs = new Class[8];
    clazzs[0] = boolean[].class;
    clazzs[1] = byte[].class;
    clazzs[2] = short[].class;
    clazzs[3] = char[].class;
    clazzs[4] = int[].class;
    clazzs[5] = float[].class;
    clazzs[6] = long[].class;
    clazzs[7] = double[].class;

    Stream.of(clazzs).forEach(clazz -> {
    		int longBaseOffset = unsafe.arrayBaseOffset(clazz);
        int longIndexScale = unsafe.arrayIndexScale(clazz);
//            System.out.printf("%s, 数组首部元素偏移地址: [%d], 数组中元素的增量地址: [%d]\n", clazz.getName(), longBaseOffset, longIndexScale);
    });

    // 数组的首元素偏移地址, 元素的增量地址
    int longBaseOffset = Unsafe.ARRAY_LONG_BASE_OFFSET;
    int longIndexScale = Unsafe.ARRAY_LONG_INDEX_SCALE;
    // long[]
    final long[] arr = new long[1000];
    int last = arr.length - 1;
    arr[last] = -1;
      
		// Before change, arr[999] = ffffffffffffffff 
    System.out.printf( "Before change, arr[%d] = %s \n", last, Long.toHexString(arr[last]));

    for (int i = 0; i < 8; i++) {
      // 直接操作内存中数组
    	unsafe.putByte(arr, longBaseOffset + longIndexScale * last + i, (byte)0);
      System.out.printf( "第 [%d] 次change, HexVal = %s, arr[%d] = %d\n", 
                    i + 1, 
                    Long.toHexString(arr[last]), 
                    last, 
                    arr[last]);
  	}
	}
}

/*
	执行结果:
	Before change, arr[999] = ffffffffffffffff 
  第 [1] 次change, HexVal = ffffffffffffff00, arr[999] = -256
  第 [2] 次change, HexVal = ffffffffffff0000, arr[999] = -65536
  第 [3] 次change, HexVal = ffffffffff000000, arr[999] = -16777216
  第 [4] 次change, HexVal = ffffffff00000000, arr[999] = -4294967296
  第 [5] 次change, HexVal = ffffff0000000000, arr[999] = -1099511627776
  第 [6] 次change, HexVal = ffff000000000000, arr[999] = -281474976710656
  第 [7] 次change, HexVal = ff00000000000000, arr[999] = -72057594037927936
  第 [8] 次change, HexVal = 0, arr[999] = 0
 */
```

### 四、sun.misc.Unsafe的内存分配

Java里我们的能分配的内存大小是有限的。这个限制在Java的最初版本里就已经定下来了。如果现在我们需要更多的内存。在Java里两个方法：

1. 分配许多小块的内存，然后逻辑上把它们当作一块连续的大内存来使用。
2. 使用 ***sun.misc.Unsafe.allcateMemory(long)*** 来进行内存分配。

第一个方法只是从算法的角度来看，所以我们还是来看下第二个方法。

***sun.misc.Unsafe*** 提供了一组方法来进行内存的分配，重新分配，以及释放。它们和C的***malloc/free*** 方法很像：

1. ***long Unsafe.allocateMemory(long size)*** 

   分配一块内存空间。这块内存可能会包含垃圾数据（没有自动清零）。如果分配失败的话会抛一个java.lang.OutOfMemoryError的异常。它会返回一个非零的内存地址（看下面的描述）

2. ***Unsafe.reallocateMemory(long address, long size)***

   重新分配一块内存，把数据从旧的内存缓冲区（address指向的地方）中拷贝到的新分配的内存块中。如果 ***address = 0***，这个方法和 allocateMemory 的效果是一样的。它返回的是新的内存缓冲区的地址。

3. ***Unsafe.freeMemory(long address)***

   释放一个由前面那两方法生成的内存缓冲区。如果 ***address = 0*** 什么也不干 。

***这些方法分配的内存 ( 这些内存就是堆外内存 )*** 应该在一个被称为单寄存器地址的模式下使用：Unsafe提供了一组只接受一个地址参数的方法（不像双寄存器模式，它们需要一个Object还有一个偏移量offset）。通过这种方式分配的内存可以比你在 ***-Xmx*** 的Java参数里配置的还要大。

> ***Tips: Unsafe 分配出来的内存是无法进行垃圾回收的 ( 因为是堆外内存或者称直接内存 )。你得把它当成一种正常的资源，自己去进行管理。***

下面是使用 ***Unsafe.allocateMemory*** 分配内存的一个例子，同时它还检查了整个内存缓冲区是不是可读写的：

