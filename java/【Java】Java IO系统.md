# 【Java】Java I/O系统

#Lang/Java

本章就介绍Java标准类库中各种各样的类以及它们的用法。

### 一、I/O

#### 1　File类

它既能代表`一个特定文件的名称，又能代表一个目录下的一组文件的名称`。如果它指的是一个文件集，我们就可以对此集合调用list()方法，这个方法会返回一个字符数组。我们很容易就可以理解返回的是一个数组而不是某个更具灵活性的类容器，因为元素的个数是固定的，所以如果我们想取得不同的目录列表，只需要再创建一个不同的File对象就可以了。实际上，FilePath（文件路径）对这个类来说是个更好的名字。本节举例示范了这个类的用法，包括了与它相关的FilenameFilter接口。

#### 2　输入和输出

编程语言的I/O类库中常使用流这个抽象概念，它代表任何有能力产出数据的数据源对象或者是有能力接收数据的接收端对象。“流”屏蔽了实际的I/O设备中处理数据的细节。

Java类库中的I/O类分成输入和输出两部分，可以在JDK文档里的类层次结构中查看到。通过继承，任何自Inputstream或Reader派生而来的类都含有名为read（）的基本方法，用于读取单个字节或者字节数组。同样，任何自OutputStream或Writer派生而来的类都含有名为write（）的基本方法，用于写单个字节或者字节数组。但是，我们通常不会用到这些方法，它们之所以存在是因为别的类可以使用它们，以便提供更有用的接口。因此，我们很少使用单一的类来创建流对象，而是通过叠合多个对象来提供所期望的功能（这是装饰器设计模式，你将在本节中看到它）。实际上，Java中“流”类库让人迷惑的主要原因就在于：创建单一的结果流，却需要创建多个对象。

##### 2.1　InputStream类型

InputStream的作用是用来表示那些从不同数据源产生输入的类。如表18-1所示，这些数据源包括：

1. 字节数组。
2. String对象。
3. 文件。
4. “管道”，工作方式与实际管道相似，即，从一端输入，从另一端输出。
5. 一个由其他种类的流组成的序列，以便我们可以将它们收集合并到一个流内。
6. 其他数据源，如Internet连接等。

每一种数据源都有相应的InputStream子类。另外，FilterInputStream也属于一种InputStream，为“装饰器”（decorator）类提供基类，其中，“装饰器”类可以把属性或有用的接口与输入流连接在一起。我们稍后再讨论它。

| 类                      | 功能                                                         | 构造参数                                       |
| ----------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| ByteArrayInputStream    | 允许将内存缓冲区当做 InputStream 使用                        | 缓冲区, 字节将从中取出.                        |
| StringBufferInputStream | 将 String转换为 InputStream                                  | 字符串, 底层实现使用的 StringBuffer            |
| FileInputStream         | 用于从文件中读取信息                                         | 字符串, 表示文件名, 文件或 FileDescriptor 对象 |
| PipedInputStream        | 产生用于写入相关 PipedOutStream的数据, 用于实现"管道化"的概念 |                                                |
| SequenceInputStream     | 将两个或多个 InputStream对象转换成一个 InputStream           |                                                |
| FilterInputStream       | 抽象类, 用作装饰器的接口. 其中, 装饰器为其他InputStream 类提供提供有用的功能 |                                                |

##### 2.2　OutputStream类型

如表18-2所示，该类别的类决定了输出所要去往的目标：字节数组（但不是String，不过你当然可以用字节数组自己创建）、文件或管道。

另外，FilterOutputStream为“装饰器”类提供了一个基类，“装饰器”类把属性或者有用的接口与输出流连接了起来，这些稍后会讨论。

| 类                    | 功能                                                         | 构造参数                                       |
| --------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| ByteArrayOutputStream | 在内存中创建缓冲区, 所有送往"流"的数据都要放置在此缓冲区     | 缓冲区初始化尺寸(可选的)                       |
| FileOutputStream      | 用于将信息写入文件                                           | 字符串, 表示文件名, 文件或 FileDescriptor 对象 |
| PipedOutputStream     | 任何写入其中数据都会自动作为相关 PipedInputstream的输出, 用于实现"管道化"概念 |                                                |
| FilterOutputStream    | 抽象类, 用作装饰器的接口. 其中, 装饰器为其他InputStream 类提供提供有用的功能 |                                                |

#### 3　添加属性和有用的接口

FilterInputStream和FilterOutputStream是用来提供装饰器类接口以控制特定输入流（InputStream）和输出流（OutputStream）的两个类，它们的名字并不是很直观。FilterInput-Stream和FilterOutputStream分别自I/O类库中的基类InputStream和OutputStream派生而来，这两个类是装饰器的必要条件（以便能为所有正在被修饰的对象提供通用接口）。

##### 3.1　通过FilterInputStream从InputStream读取数据

FilterInputStream类能够完成两件完全不同的事情。其中，DataInputStream允许我们读取不同的基本类型数据以及String对象（所有方法都以“read”开头，例如readByte（）、readFloat（）等等）。搭配相应的DataOutputStream，我们就可以通过数据“流”将基本类型的数据从一个地方迁移到另一个地方。具体是哪些“地方”是由表18-1中的那些类决定的。

其他FilterInputstream类则在内部修改InputStream的行为方式：是否缓冲，是否保留它所读过的行（允许我们查询行数或设置“行数），以及是否把单一字符推回输入流等等。最后两个类看起来更像是为了创建一个编译器（它们被添加进来可能是为了对“用Java构建编译器”实验提供支持），因此我们在一般编程中不会用到它们。

我们几乎每次都要对输入进行缓冲——不管我们正在连接的是什么I/O设备，所以，I/O类库把无缓冲输入（而不是缓冲输入）作为特殊情况（或只是方法调用）就显得更加合理了。FilterInputStream的类型及功能如表18-3所示。

| 类                    | 功能                                                         | 构造参数                              |
| --------------------- | ------------------------------------------------------------ | ------------------------------------- |
| DataInputStream       | 与 DataOutputStream 搭配使用, 因此我们可以按照可移植的方式从流读取基本的数据类型(int, char, short, long, double, ......) | InputStream                           |
| BufferInputStream     | 使用它可以防止每次读取都进行实际写操作. 代表"使用缓冲区"     | InputStream, 可以指定缓冲区的大小     |
| LineNumberInputStream | 跟踪输入流中的行号, 可调用getLineNumber()或setLineNumber(int) | InputStream                           |
| PushbackInputStream   | 具有"能弹出一个字节的缓存区", 因此可以将最后读到的一个字符串退回 | InputStream, 它通常作为编译器的扫描器 |

##### 3.2　通过FilterOutPutStream向OutputStream写入

与DataInputStream对应的是DataOutputStream，它可以将各种基本数据类型以及String对象格式化输出到“流”中；这样以来，任何机器上的任何DataInputStream都能够读取它们。所有方法都以“wirte”开头，例如writeByte（）、writeFloat（）等等。

PrintStream最初的目的便是为了以可视化格式打印所有的基本数据类型以及String对象。这和DataOutputStream不同，后者的目的是将数据元素置入“流”中，使DataInputStream能够可移植地重构它们。

PrintStream内有两个重要的方法：print（）和println（）。对它们进行了重载，以便可打印出各种数据类型。print（）和println（）之间的差异是，后者在操作完毕后会添加一个换行符。

PrintStream也未完全国际化，不能以平台无关的方式处理换行动作（这些问题在printWriter中得到了解决，这在后面讲述）。

BufferedOutputStream是一个修改过的OutputStream，它对数据流使用缓冲技术；因此当每次向流写入时，不必每次都进行实际的物理写动作。所以在进行输出时，我们可能更经常的是使用它。FilterOutputStream的类型及功能如表18-4所示。

| 类                   | 功能                                                         | 构造参数                                                     |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| DataOutputStream     | 与 DataInputStream 搭配使用, 因此我们可以按照可移植的方式从流读取基本的数据类型(int, char, short, long, double, ......) | OutputStream                                                 |
| PrintStream          | 用于产生格式化的输出, 其中 DataOutputStream处理数据的存储, PrintStream 处理显示 | OutputStream, 可以使用 boolean 指定是否在每次换行时清空缓冲区(可选的), 应该是对 OutputStream 对象的"final"封装, 可能会经常使用到它 |
| BufferedOutputStream | 使用它可以防止每次读取都进行实际写操作. 代表"使用缓冲区". 可以调用  flush() 清空缓冲区 | OutputStream, 可以指定缓冲区大小(可选的)                     |

#### 4　Reader和Writer

InputStream和OutputStreamt在以面向字节形式的I/O中仍可以提供极有价值的功能，Reader和Writer则提供兼容Unicode与面向字符的I/O功能。另外：

1. Java 1. 1向InputStream和OutputStreamt继承层次结构中添加了一些新类，所以显然这两个类是不会被取代的。

2. 有时我们必须把来自于“字节”层次结构中的类和“字符”层次结构中的类结合起来使用。为了实现这个目的，要用到“适配器”（adapter）类：`InputStreamReader`可以把InputStream转换为Reader，而`OutputStreamWriter`可以把OutputStream转换为Writer。

   >`InputStreamReader和OutputStreamWriter` 是两个适配器, 用于将 字节流转换为字符流 

设计Reader和Writer继承层次结构主要是为了国际化。老的I/O流继承层次结构仅支持8位字节流，并且不能很好地处理16位的Unicode字符。由于Unicode用于字符国际化（Java本身的char也是16位的Unicode），所以添加Reader和Writer继承层次结构就是为了在所有的I/O操作中都支持Unicode。另外，新类库的设计使得它的操作比旧类库更快。

##### 4.1　数据的来源和去处

几乎所有原始的Java I/O流类都有相应的Reader和Writer类来提供天然的Unicode操作。然而在某些场合，面向字节的InputStream和OutputStream才是正确的解决方案；特别是，java.util.zip类库就是面向字节的而不是面向字符的。***因此，最明智的做法是尽量尝试使用Reader和Writer，一旦程序代码无法成功编译，我们就会发现自己不得不使用面向字节的类库。***

> 这里大佬书中也建议, I/O 编程尽量面向字符流, 除非出现无法编译成功, 不得不面向字节流

下面的表展示了在两个继承层次结构中，信息的来源和去处（即数据物理上来自哪里及去向哪里）之间的对应关系：

| 来源与去处: Java1.0 类          | 相应的 Java1.1 类                  |
| ------------------------------- | ---------------------------------- |
| InputStream                     | Reader, 适配器: InputStreamReader  |
| OutputStream                    | Writer, 适配器: OutputStreamWriter |
| FileInputStream                 | FileReader                         |
| FileOutputStream                | FileWriter                         |
| StringBufferInputStream(已弃用) | StringReader                       |
| (无相应的类)                    | StringWriter                       |
| ByteArrayInputStream            | CharArrayReader                    |
| ByteArrayOutputStream           | CharArrayWriter                    |
| PipedInputStream                | PipedReader                        |
| PipedOutputStream               | PipedWriter                        |

##### 4.2　更改流的行为

对于InputStream和OutputStream来说，我们会使用FilterInputStream和FilterOutputStream的装饰器子类来修改“流”以满足特殊需要。Reader和Writer的类继承层次结构继续沿用相同的思想——但是并不完全相同。

在下表中，相对于前一表格来说，左右之间的对应关系的近似程度更加粗略一些。造成这种差别的原因是因为类的组织形式不同；尽管BufferedOutputStream是FilterOutputStream的子类，但是BufferedWriter并不是FilterWriter的子类（尽管FilterWriter是抽象类，没有任何子类，把它放在那里也只是把它作为一个占位符，或仅仅让我们不会对它所在的地方产生疑惑）。然而，这些类的接口却十分相似。

| 来源与去处: Java1.0 类        | 相应的 Java1.1 类                                            |
| ----------------------------- | ------------------------------------------------------------ |
| FilterInputStream             | FilterReader                                                 |
| FilterOutputStream            | FilterWriter(抽象类, 没有子类)                               |
| BufferedInputStream           | BufferedReader(也有 readLine())                              |
| BufferedOutputStream          | BufferedWriter                                               |
| DataInputStream               | 使用DataInputStream(除了要使用 readLine()时以外, 这是应该使用BufferedReader) |
| LineNumberInputStream(已弃用) | LineNumberReader                                             |
| PrintStream                   | PrintWriter                                                  |
| StreamTokenizer               | StreamTokenizer                                              |
| PushbackInputStream           | PushbackReader                                               |

有一点很清楚：***无论我们何时使用readLine（），都不应该使用DataInputStream（这会遭到编译器的强烈反对），而应该使用BufferedReader。除了这一点，DataInputStream仍是I/O类库的首选成员。***

为了更容易地过渡到使用PrintWriter，它提供了一个既能接受Writer对象又能接受任何OutputStream对象的构造器。PrintWriter的格式化接口实际上与PrintStream相同。

在Java SE5中添加了PrintWriter构造器，以简化在将输出写入时的文件创建过程，你马上就会看到它。

有一种PrintWriter构造器还有一个选项，就是“自动执行清空”选项。如果构造器设置此选项，则在每个Println（）执行之后，便会自动清空。

#### 5　自我独立的类：RandomAccessFile

RandomAccessFile适用于由大小已知的记录组成的文件，所以我们可以使用seek（）将记录从一处转移到另一处，然后读取或者修改记录。文件中记录的大小不一定都相同，只要我们能够确定那些记录有多大以及它们在文件中的位置即可。

最初，我们可能难以相信RandomAccessFile不是InputStream或者OutputStream继承层次结构中的一部分。除了***实现了DataInput和DataOutput接口***（DataInputStream和DataOutputStream也实现了这两个接口）之外，它和这两个继承层次结构没有任何关联。它甚至不使用InputStream和OutputStream类中已有的任何功能。***它是一个完全独立的类，从头开始编写其所有的方法（大多数都是本地的）***。这么做是因为RandomAccessFile拥有和别的I/O类型本质不同的行为，因为我们可以在一个文件内向前和向后移动。在任何情况下，它都是自我独立的，直接从Object派生而来。

从本质上来说，RandomAccessFile的工作方式类似于把DataInputStream和DataOutStream组合起来使用，还添加了一些方法。其中方法getFilePointer（）用于查找当前所处的文件位置，seek（）用于在文件内移至新的位置，length（）用于判断文件的最大尺寸。另外，其构造器还需要第二个参数（和C中的fopen（）相同）用来指示我们只是“随机读”（r）还是“既读又写”（rw）。它并不支持只写文件，这表明RandomAccessFile若是从DataInputStream继承而来也可能会运行得很好。

只有RandonAccessFile支持搜寻方法，并且只适用于文件。BufferedInputStream却能允许标注（mark（））位置（其值存储于内部某个简单变量内）和重新设定位置（reset（）），但这些功能很有限，不是非常有用。

在JDK 1.4中，RandomAccessFile的大多数功能（但不是全部）由nio存储映射文件所取代，本章稍后会讲述。





### 二、新I/O

JDK 1.4的java.nio.*包中引入了新的JavaI/O类库，其目的在于提高速度。实际上，旧的I/O包已经使用nio重新实现过，以便充分利用这种速度提高，因此，即使我们不显式地用nio编写代码，也能从中受益。速度的提高在文件I/O和网络I/O中都有可能发生，我们在这里只研究前者 [1] ；对于后者，在《Thinking in Enterprise Java》中有论述。

速度的提高来自于所使用的结构更接近于操作系统执行I/O的方式：通道和缓冲器。我们可以把它想像成一个煤矿，通道是一个包含煤层（数据）的矿藏，而缓冲器则是派送到矿藏的卡车。卡车载满煤炭而归，我们再从卡车上获得煤炭。也就是说，我们并没有直接和通道交互；我们只是和缓冲器交互，并把缓冲器派送到通道。通道要么从缓冲器获得数据，要么向缓冲器发送数据。

唯一直接与通道交互的缓冲器是ByteBuffer——也就是说，可以存储未加工字节的缓冲器。当我们查询JDK文档中的java.nio.ByteBuffer时，会发现它是相当基础的类：通过告知分配多少存储空间来创建一个ByteBuffer对象，并且还有一个方法选择集，用于以原始的字节形式或基本数据类型输出和读取数据。但是，没办法输出或读取对象，即使是字符串对象也不行。这种处理虽然很低级，但却正好，因为这是大多数操作系统中更有效的映射方式。

旧I/O类库中有三个类被修改了，用以产生File Channel。这三个被修改的类是 ***FileInputStream*** 、 ***FileOutputStream*** 以及用于既读又写的 ***RandomAccessFile*** 。注意这些是字节操纵流，与低层的nio性质一致。Reader和Writer这种字符模式类不能用于产生通道；但是java.nio.channels.Channels类提供了实用方法，用以在通道中产生Reader和Writer。

```java
package cgs.tij.nio;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.charset.Charset;

import static java.nio.charset.StandardCharsets.UTF_8;

/**
 * @Description TODO
 * @Author sherlock
 * @Date
 */
public class GetChannel {

    private static final int _1_MB = 1024 * 1024;
    private static final String OUT = "/Users/sherlock/IdeaProjects/Thinking_in_Java/Chaptor18-Java_IO/output.txt";

    public static void main(String[] args) throws IOException {

        String content = "Licensed to the Apache Software Foundation (ASF) under one or more contributor license agreements.";

        FileChannel fc = new FileOutputStream(OUT).getChannel();
        fc.write(ByteBuffer.wrap(content.getBytes(UTF_8)));
        fc.close();

        // add to the end of file
        fc = new RandomAccessFile(OUT, "rw").getChannel();
        fc.position(fc.size());
        fc.write(ByteBuffer.wrap("Some more".getBytes()));
        fc.close();

        // read the file
        fc = new FileInputStream(OUT).getChannel();
        ByteBuffer buffer = ByteBuffer.allocate(_1_MB);
        fc.read(buffer);
        buffer.flip();
        while (buffer.hasRemaining()) {
            System.out.print((char) buffer.get());
        }

    }

    /*
        Licensed to the Apache Software Foundation (ASF) under one or more contributor license agreements.Some more
     */
}
```

对于这里所展示的任何流类，getChannel（）将会产生一个FileChannel。通道是一种相当基础的东西：可以向它传送用于读写的ByteBuffer，并且可以锁定文件的某些区域用于独占式访问（稍后讲述）。

将字节存放于ByteBuffer的方法之一是：使用一种 ***put()*** 方法直接对它们进行填充，填入一个或多个字节，或基本数据类型的值。不过，正如所见，也可以使用 ***warp()*** 方法将已存在的字节数组“包装”到ByteBuffer中。一旦如此，就不再复制底层的数组，而是把它作为所产生的ByteBuffer的存储器，我们称之为数组支持的ByteBuffer。

文件用RandomAccessFile被再次打开。注意我们可以在文件内随处移动 FileChannel；在这里，我们把它移到最后，以便附加其他的写操作。

对于只读访问，我们必须显式地使用静态的 ***allocate()*** 方法来分配ByteBuffer。nio的目标就是快速移动大量数据，因此ByteBuffer的大小就显得尤为重要——实际上，这里使用的1K可能比我们通常要使用的小一点（必须通过实际运行应用程序来找到最佳尺寸）。

甚至达到更高的速度也有可能，方法就是使用 ***allocateDirect()*** 而不是 ***allocate()*** ，以产生一个与操作系统有更高耦合性的“直接”缓冲器。但是，这种分配的开支会更大，并且具体实现也随操作系统的不同而不同，因此必须再次实际运行应用程序来查看直接缓冲是否可以使我们获得速度上的优势。

一旦调用read（）来告知FileChannel向ByteBuffer存储字节，就必须调用缓冲器上的flip（），让它做好让别人读取字节的准备（是的，这似乎有一点拙劣，但是请记住，它是很拙劣的，但却适用于获取最大速度）。如果我们打算使用缓冲器执行进一步的read（）操作，我们也必须得调用clear（）来为每个read（）做好准备。
