# 【JAVA NIO】ByteBuffer

#Lang/Java

### 缓冲区(Buffer)

缓冲区(Buffer)就是在内存中预留指定大小的存储空间用来对输入/输出(I/O)的数据作临时存储，这部分预留的内存空间就叫做缓冲区：

使用缓冲区有这么两个好处：

1. 减少实际的物理读写次数

2. 缓冲区在创建时就被分配内存，这块内存区域一直被重用，可以减少动态分配和回收内存的次数

>举个简单的例子，比如A地有1w块砖要搬到B地
>
>由于没有工具（缓冲区），我们一次只能搬一块，那么就要搬1w次（实际读写次数）
>
>如果A，B两地距离很远的话（IO性能消耗），那么性能消耗将会很大但是要是此时我们有辆大卡车（缓冲区），一次可运5000本，那么2次就够了
>
>相比之前，性能肯定是大大提高了。
>
>而且一般在实际过程中，我们一般是先将文件读入内存，再从内存写出到别的地方
>
>这样在输入输出过程中我们都可以用缓存来提升IO性能。

所以，buffer在IO中很重要。在旧I/O类库中（相对java.nio包）中的***BufferedInputStream、BufferedOutputStream、BufferedReader和BufferedWriter*** 在其实现中都运用了缓冲区。java.nio包公开了Buffer API，使得Java程序可以直接控制和运用缓冲区。

在Java NIO中，缓冲区的作用也是用来临时存储数据，可以理解为是I/O操作中数据的中转站。缓冲区直接为通道(Channel)服务，写入数据到通道或从通道读取数据，这样的操利用缓冲区数据来传递就可以达到对数据高效处理的目的。在NIO中主要有八种缓冲区类 ( 其中MappedByteBuffer是专门用于内存映射的一种ByteBuffer )：
![buffer](/Users/sherlock/Desktop/notes/allPics/Java/buffer.png)

#### Fields

所有缓冲区都有4个属性：

1. capacity
2. limit
3. position
4. mark

并遵循：mark <= position <= limit <= capacity，下表格是对着4个属性的解释：

| 属性     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Capacity | 容量，即可以容纳的最大数据量；在缓冲区创建时被设定并且不能改变 |
| Limit    | 表示缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写操作。且极限是可以修改的 |
| Position | 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变改值，为下次读写作准备 |
| Mark     | 标记，调用mark()来设置mark=position，再调用reset()可以让position恢复到标记的位置 |

#### Methods

##### 1、实例化

java.nio.Buffer类是一个抽象类，不能被实例化。Buffer类的直接子类，如ByteBuffer等也是抽象类，所以也不能被实例化。

但是ByteBuffer类提供了4个静态工厂方法来获得ByteBuffer的实例：	

| 方法                                       | 描述                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| allocate(int capacity)                     | 从堆空间中分配一个容量大小为capacity的byte数组作为缓冲区的byte数据存储器 |
| allocateDirect(int capacity)               | 是不使用JVM堆栈而是通过操作系统来创建内存块用作缓冲区，它与当前操作系统能够更好的耦合，因此能进一步提高I/O操作速度。但是分配直接缓冲区的系统开销很大，因此只有在缓冲区较大并长期存在，或者需要经常重用时，才使用这种缓冲区 |
| wrap(byte[] array)                         | 这个缓冲区的数据会存放在byte数组中，bytes数组或buff缓冲区任何一方中数据的改动都会影响另一方。其实ByteBuffer底层本来就有一个bytes数组负责来保存buffer缓冲区中的数据，通过allocate方法系统会帮你构造一个byte数组 |
| wrap(byte[] array, int offset, int length) | 在上一个方法的基础上可以指定偏移量和长度，这个offset也就是包装后byteBuffer的position，而length呢就是limit-position的大小，从而我们可以得到limit的位置为length+position(offset) |

我写了这几个方法的测试方法，大家可以运行起来更容易理解

2、另外一些常用的方法

| 方法                                    | 描述                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| limit()                                 | 其中读取和设置这4个属性的方法的命名和jQuery中的val(),val(10)类似，一个负责get，一个负责set |
| reset()                                 | 把position设置成mark的值，相当于之前做过一个标记，现在要退回到之前标记的地方 |
| clear()                                 | position = 0;limit = capacity;mark = -1;  有点初始化的味道，但是并不影响底层byte数组的内容 |
| flip()                                  | limit = position;position = 0;mark = -1;翻转，也就是让flip之后的position到limit这块区域变成之前的0到position这块，翻转就是将一个处于存数据状态的缓冲区变为一个处于准备取数据的状态 |
| rewind()                                | 把position设为0，mark设为-1，不改变limit的值                 |
| remaining()                             | return limit - position;返回limit和position之间相对位置差    |
| hasRemaining()                          | return position < limit返回是否还有未读内容                  |
| compact()                               | 把从position到limit中的内容移到0到limit-position的区域内，position和limit的取值也分别变成limit-position、capacity。如果先将positon设置到limit，再compact，那么相当于clear() |
| get()                                   | 相对读，从position位置读取一个byte，并将position+1，为下次读写作准备 |
| get(int index)                          | 绝对读，读取byteBuffer底层的bytes中下标为index的byte，不改变position |
| get(byte[] dst, int offset, int length) | 从position位置开始相对读，读length个byte，并写入dst下标从offset到offset+length的区域 |
| put(byte b)                             | 相对写，向position的位置写入一个byte，并将postion+1，为下次读写作准备 |
| put(int index, byte b)                  | 绝对写，向byteBuffer底层的bytes中下标为index的位置插入byte b，不改变position |
| put(ByteBuffer src)                     | 用相对写，把src中可读的部分（也就是position到limit）写入此byteBuffer |
| put(byte[] src, int offset, int length) | 从src数组中的offset到offset+length区域读取数据并使用相对写写入此byteBuffer |

#### buffer order

在一个64位的CPU中“字长”为64个bit，也就是8个byte。***在这样的CPU中，总是以8字节对齐的方式来读取或写入内存***，那么同样这8个字节的数据是以什么顺序保存在内存中的呢？例如，

现在我们要向内存地址为a的地方写入数据0x0A0B0C0D00000000，那么这8个字节分别落在哪个地址的内存上呢？这就涉及到字节序的问题了。

***每个数据都有所谓的“有效位（significant byte）”***，它的意思是“表示这个数据所用的字节”。例如一个64位整数，它的有效位就是8个字节。而对于0x0A0B0C0D00000000来说，它的有效位从高到低便是0A、0B、0C、0D、00、00、00、00——这里您可以把它作为一个256进制的数来看（相对于我们平时所用的10进制数）。

#### 初始化

首先无论读写，均需要初始化一个ByteBuffer容器。如上所述，ByteBuffer其实就是对byte数组的一种封装，所以可以使用静态方法wrap(byte[] data)手动封装数组，也可以通过另一个静态的allocate(int size)方法初始化指定长度的ByteBuffer。初始化后，ByteBuffer的position就是0；其中的数据就是初始化为0的字节数组；limit = capacity = 字节数组的长度；用户还未自定义标记位置，所以mark = -1，即undefined状态。下图就表示初始化了一个容量为16个字节的ByteBuffer，其中每个字节用两位16进制数表示(1 byte = 8 bit, 4bit 一般用一个 16 进制数表示, 因为 4 bit 的 2 进制最大就是 1111 = 16 进制的 F, 所以 1 byte 就用 2 个 16 进制数表示)：

![allocateBuffer](/Users/sherlock/Desktop/notes/allPics/Java/allocateBuffer.png)
	
#### ByteBuffer写数据

##### 1. 手动put写入数据

可以手动通过put(byte b)或put(byte[] b)方法向ByteBuffer中添加一个字节或一个字节数组。ByteBuffer也方便地提供了几种写入基本类型的put方法：putChar(char val)、putShort(short val)、putInt(int val)、putFloat(float val)、putLong(long val)、putDouble(double val)。执行这些写入方法之后，就会以当前的position位置作为起始位置，写入对应长度的数据，并在写入完毕之后将position向后移动对应的长度。下图就表示了分别向ByteBuffer中写入1个字节的byte数据和4个字节的Integer数据的结果：

![buffer_put](/Users/sherlock/Desktop/notes/allPics/Java/buffer_put.png)	
	

但是当想要写入的数据长度大于ByteBuffer当前剩余的长度时，则会抛出BufferOverflowException异常，剩余长度的定义即为limit与position之间的差值（即 limit - position）。如上述例子中，若再执行buffer.put(new byte[12]);就会抛出BufferOverflowException异常，因为剩余长度为11。可以通过调用ByteBuffer.remaining();查看该ByteBuffer当前的剩余可用长度。

##### 2. 从SocketChannel中读数据至ByteBuffer

在实际应用中，往往是调用SocketChannel.read(ByteBuffer dst)，从SocketChannel中读入数据至指定的ByteBuffer中。由于ByteBuffer常常是非阻塞的，所以该方法的返回值即为实际读取到的字节长度。假设实际读取到的字节长度为 n，ByteBuffer剩余可用长度为 r，则二者的关系一定满足：0 <= n <= r。继续接上述的例子，假设调用read方法，从SocketChannel中读入了4个字节的数据，则buffer的情况如下：

![channel_buffer_read](/Users/sherlock/Desktop/notes/allPics/Java/channel_buffer_read.png)	
	

#### 从ByteBuffer中读数据

##### 1. 复位position

现在ByteBuffer容器中已经存有数据，那么现在就要从ByteBuffer中将这些数据取出来解析。由于position就是下一个读写操作的起始位置，故在读取数据后直接写出数据肯定是不正确的，要先把position复位到想要读取的位置。

首先看一个rewind()方法，该方法仅仅是简单粗暴地将position直接复原到0，limit不变。这样进行读取操作的话，就是从第一个字节开始读取了。如下图：

![buffer_position](/Users/sherlock/Desktop/notes/allPics/Java/buffer_position.png)

	该方法虽然复位了position，可以从头开始读取数据，但是并未标记处有效数据的结束位置。如本例所述，ByteBuffer总容量为16字节，但实际上只读取了9个字节的数据，因此最后的7个字节是无效的数据。故rewind()方法常常用于字节数组的完整拷贝。

实际应用中更常用的是flip()方法，该方法不仅将position复位为0，同时也将limit的位置放置在了position之前所在的位置上，这样position和limit之间即为新读取到的有效数据。如下图：

![bytebuffer_flip](/Users/sherlock/Desktop/notes/allPics/Java/bytebuffer_flip.png)

##### 2. 读取数据

在将position复位之后，我们便可以从ByteBuffer中读取有效数据了。类似put()方法，ByteBuffer同样提供了一系列get方法，从position开始读取数据。get()方法读取1个字节，getChar()、getShort()、getInt()、getFloat()、getLong()、getDouble()则读取相应字节数的数据，并转换成对应的数据类型。如getInt()即为读取4个字节，返回一个Int。在调用这些方法读取数据之后，ByteBuffer还会将position向后移动读取的长度，以便继续调用get类方法读取之后的数据。

这一系列get方法也都有对应的接收一个int参数的重载方法，参数值表示从指定的位置读取对应长度的数据。如getDouble(2)则表示从下标为2的位置开始读取8个字节的数据，转换为double返回。不过实际应用中往往对指定位置的数据并不那么确定，所以带int参数的方法也不是很常用。get()方法则有两个重载方法：

- get(byte[] dst, int offset, int length)

  表示尝试从 position 开始读取 length 长度的数据拷贝到 dst 目标数组 offset 到 offset + length 位置，相当于执行了

  ```java
  for (int i = off; i < off + len; i++)
  	dst[i] = buffer.get();
  ```

  

- get(byte[] dst)

  尝试读取 dst 目标数组长度的数据，拷贝至目标数组，相当于执行了

  ```java
  buffer.get(dst, 0, dst.length);
  ```

此处应注意读取数据后，已读取的数据也不会被清零。下图即为从例子中连续读取1个字节的byte和4个字节的int数据：

![buffer_get](/Users/sherlock/Desktop/notes/allPics/Java/buffer_get.png)

此处同样要注意，当想要读取的数据长度大于ByteBuffer剩余的长度时，则会抛出 ***BufferUnderflowException*** 异常。如上例中，若再调用buffer.getLong()就会抛出 ***BufferUnderflowException*** 异常，因为 remaining 仅为4。

##### 3. 确保数据长度

为了防止出现上述的 BufferUnderflowException 异常，最好要在读取数据之前确保 ByteBuffer 中的有效数据长度足够。

```java
private void checkReadLen(
	long reqLen,
	ByteBuffer buffer,
	SocketChannel dataSrc
) throws IOException {
  int readLen;
  if (buffer.remaining() < reqLen) { // 剩余长度不够，重新读取
  	buffer.compact(); // 准备继续读取
    System.out.println("Buffer remaining is less than" + reqLen + ". Read Again...");
    while (true) {
      readLen = dataSrc.read(buffer);
      System.out.println("Read Again Length: " + readLen + "; Buffer Position: " + buffer.position());
      if (buffer.position() >= reqLen) { // 可读的字节数超过要求字节数
        break;
      }
    }
    buffer.flip();
    System.out.println("Read Enough Data. Remaining bytes in buffer: " + buffer.remaining());
  }
}
```

##### 4. 字节序处理

基本类型的值在内存中的存储形式还有字节序的问题，这种问题在不同CPU的机器之间进行网络通信时尤其应该注意。同时在调用ByteBuffer的各种get方法获取对应类型的数值时，ByteBuffer也会使用自己的字节序进行转换。因此若ByteBuffer的字节序与数据的字节序不一致，就会返回不正确的值。如对于int类型的数值8848，用16进制表示，大字节序为：0x 00 00 22 90；小字节序为：0x 90 22 00 00。若接收到的是小字节序的数据，但是却使用大字节序的方式进行解析，获取的就不是8848，而是-1876819968，也就是大字节序表示的有符号int类型的 0x 90 22 00 00。

JavaNIO提供了java.nio.ByteOrder枚举类来表示机器的字节序，同时提供了静态方法ByteOrder.nativeOrder()可以获取到当前机器使用的字节序，使用ByteBuffer中的order()方法即可获取该buffer所使用的字节序。同时也可以在该方法中传递一个ByteOrder枚举类型来为ByteBuffer指定相应的字节序。如调用buffer.order(ByteOrder.LITTLE_ENDIAN)则将buffer的字节序更改为小字节序。

一开始并不知道还可以这样操作，比较愚蠢地手动将读取到的数据进行字节序的转换。不过觉得还是可以记下来，也许在别的地方用得到。JDK中的 Integer 和 Long 都提供了一个静态方法reverseBytes()来将对应的 int 或 long 数值的字节序进行翻转。而若想读取 float 或 double，也可以先读取 int 或 long，然后调用 Float.intBitsToFloat(int val) 或 Double.longBitsToDouble(long val) 方法将对应的 int 值或 long 值进行转换。当ByteBuffer中的字节序与解析的字节序相反时，可以使用如下方法读取：

```java
int i = Integer.reverseBytes(buffer.getInt()); 
float f = Float.intBitsToFloat(Integer.reverseBytes(buffer.getInt()));
long l = Long.reverseBytes(buffer.getLong());
double d = Double.longBitsToDouble(buffer.getLong());
```

##### 5. 继续写入数据

由于ByteBuffer往往是非阻塞式的，故不能确定新的数据是否已经读完，但这时候依然可以调用ByteBuffer的compact()方法切换到写入模式。***该方法就是将 position 到 limit 之间还未读取的数据拷贝到 ByteBuffer 中数组的最前面，然后再将 position 移动至这些数据之后的一位，将 limit 移动至 capacity。这样 position 和 limit 之间就是已经读取过的老的数据或初始化的数据，就可以放心大胆地继续写入覆盖了***。仍然使用之前的例子，调用 compact() 方法后状态如下：

> 总结的说, compact(), 就是将当前 position 之后还未读取的数据移动到数组头, 并将 position 移动到 未读数据.length + 1的位置

![buffer_read](/Users/sherlock/Desktop/notes/allPics/Java/buffer_read.png)

### 总结

总之ByteBuffer的基本用法就是：
初始化（`allocate`）–> 写入数据（`read / put`）–> 转换为写出模式（`flip`）–> 写出数据（`get`）–> 转换为写入模式（`compact`）–> 写入数据（`read / put`）

>1. 每put一次数据, position 会增加写入数据的 byte长度
>2. 调用 flip(), 会将 limit = position, position = mark = 0, ByteBuffer 实际读取的 position --> limit 之间的数据
>3. 调用 clear() , 会将 position = mark = 0, limit = capacty, 相当于重新初始化.
>4. put() n条数据之后, 立即 clear(), 则clear()前未读取的 n 条数据, 复位后会丢失.
>5. flip() 之后, 调动 clear(), 则会从头开始重新读取数据
>6. 调用 compact(), 会将 当前 position = mark = 0



