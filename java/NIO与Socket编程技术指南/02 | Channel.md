# 02 | Channel

Java NIO的通道类似流，但又有些不同：

- 既可以从通道中读取数据，又可以写数据到通道。但流的读写通常是单向的。
- 通道可以异步地读写。
- 通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入。

正如上面所说，从通道读取数据到缓冲区，从缓冲区写入数据到通道。如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/5408072-723a35d697c0c1e8.png?imageMogr2/auto-orient/strip|imageView2/2/w/338/format/webp)

## Channel的实现

这些是Java NIO中最重要的通道的实现：

```sh
FileChannel
DatagramChannel
SocketChannel
ServerSocketChannel
```

- `FileChannel` 从文件中读写数据。
- `DatagramChannel` 能通过UDP读写网络中的数据。
- `SocketChannel` 能通过TCP读写网络中的数据。
- `ServerSocketChannel`可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个`SocketChannel`。

## 基本的 Channel 示例

下面是一个使用`FileChannel`读取数据到`Buffer`中的示例：

```csharp
private static void useNio(){
  RandomAccessFile aFile = null;
  try {
    aFile = new RandomAccessFile(
      "/Users/sschen/Documents/SerialVersion.txt","rw");
    FileChannel inChannel = aFile.getChannel();

    ByteBuffer byteBuffer = ByteBuffer.allocate(48);
    int byteReader = inChannel.read(byteBuffer);

    while (byteReader != -1) {
      System.out.println("Read:" + byteReader);
      byteBuffer.flip();
      while (byteBuffer.hasRemaining()) {
        System.out.println((char)byteBuffer.get());
      }
      byteBuffer.clear();
      byteReader = inChannel.read(byteBuffer);
    }
  } catch (FileNotFoundException | IOException e) {
    e.printStackTrace();
  } finally {
    try {
      aFile.close();
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}
```

注意 `buf.flip()` 的调用，首先读取数据到`Buffer`，然后反转`Buffer`,接着再从`Buffer`中读取数据。下一节会深入讲解Buffer的更多细节。

