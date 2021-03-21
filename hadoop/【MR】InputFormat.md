# 【MR】InputFormat

我们在编写 MapReduce 程序的时候，在设置输入格式的时候，会调用如下代码：`job.setInputFormatClass(KeyVakueTextInputFormat.class)` 来保证输入文件按照我们想要的格式被读取。所有的输入格式都继承于 InputFormat，这是一个抽象类，其子类有专门用于读取普通文件的 FileInputFormat，用来读取数据库的 DBInputFormat等等。

![](https://upload-images.jianshu.io/upload_images/7126254-348de605cbcf0bd8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)



InputFormat 抽象类仅有两个抽象方法, 分别解决两个主要问题:

- 如何将数据分割成分片[比如多少行为一个分片], 由 getSplits(JobContext) 完成
- 如何读取每个分片中的数据[比如按行读取], 由 createRecordReader(InputSplit, TaskAttemptContext) 完成。

```java
public abstract List<InputSplit> getSplits(JobContext context)
public abstract RecordReader<K,V> createRecordReader(InputSplit split,TaskAttemptContext context)
```

不同的 InputFormat 实现类都会按自己的实现来读取输入数据并产生输入分片，一个输入分片会被单独的 map task 作为数据源。下面我们先看看这些输入分片(inputSplit)是什么样的。

### 1. InputSplit

Mappers 的输入是一个一个的输入分片，称 InputSplit。InputSplit 是一个抽象类，它在逻辑上包含了提供给处理这个 InputSplit 的 Mapper 的所有 K-V 对。

```java
public abstract class InputSplit {
  
  public abstract long getLength() 
    throws IOException, InterruptedException;

  public abstract String[] getLocations() 
    throws IOException, InterruptedException;
}
```

### 1.1 getLength()

用来获取 InputSplit 的大小，以支持对 InputSplits 进行排序，而 getLocations() 则用来获取存储分片的位置列表。 

我们来看一个简单 InputSplit 子类：***FileSplit***。

```java
public class FileSplit extends InputSplit implements Writable {
  
  private Path file;
  private long start;
  private long length;
  private String[] hosts;

  FileSplit() {}

  public FileSplit(Path file, 
                   long start, 
                   long length, 
                   String[] hosts) {
    this.file = file;
    this.start = start;
    this.length = length;
    this.hosts = hosts;
  }
  
  //序列化、反序列化方法，获得hosts等等……
}
```

从上面的源码我们可以看到，一个 FileSplit 是由文件路径，分片开始位置，分片大小和存储分片数据的 hosts 列表组成，由这些信息我们就可以从输入文件中切分出提供给单个 Mapper 的输入数据。这些属性会在Constructor设置，我们在后面会看到这会在InputFormat的getSplits()中构造这些分片。
  我们再看CombineFileSplit：

public class CombineFileSplit extends InputSplit implements Writable {

  private Path[] paths;
  private long[] startoffset;
  private long[] lengths;
  private String[] locations;
  private long totLength;

  public CombineFileSplit() {}
  public CombineFileSplit(Path[] files, long[] start, 
                          long[] lengths, String[] locations) {
    initSplit(files, start, lengths, locations);
  }

  public CombineFileSplit(Path[] files, long[] lengths) {
    long[] startoffset = new long[files.length];
    for (int i = 0; i < startoffset.length; i++) {
      startoffset[i] = 0;
    }
    String[] locations = new String[files.length];
    for (int i = 0; i < locations.length; i++) {
      locations[i] = "";
    }
    initSplit(files, startoffset, lengths, locations);
  }

  private void initSplit(Path[] files, long[] start, 
                         long[] lengths, String[] locations) {
    this.startoffset = start;
    this.lengths = lengths;
    this.paths = files;
    this.totLength = 0;
    this.locations = locations;
    for(long length : lengths) {
      totLength += length;
    }
  }
  //一些getter和setter方法，和序列化方法
}

  与FileSplit类似，CombineFileSplit同样包含文件路径，分片起始位置，分片大小和存储分片数据的host列表，由于CombineFileSplit是针对小文件的，它把很多小文件包在一个InputSplit内，这样一个Mapper就可以处理很多小文件。要知道我们上面的FileSplit是对应一个输入文件的，也就是说如果用FileSplit对应的FileInputFormat来作为输入格式，那么即使文件特别小，也是单独计算成一个输入分片来处理的。当我们的输入是由大量小文件组成的，就会导致有同样大量的InputSplit，从而需要同样大量的Mapper来处理，这将很慢，想想有一堆map task要运行！！这是不符合Hadoop的设计理念的，Hadoop是为处理大文件优化的。
  最后介绍TagInputSplit，这个类就是封装了一个InputSplit，然后加了一些tags在里面满足我们需要这些tags数据的情况，我们从下面就可以一目了然。
