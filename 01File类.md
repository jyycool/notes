# 01|File类

自 Java 1.0 版本以来, Java 的 I/O 库发生了明显的变化, 在原来面向字节的类中添加了面向字符和基于 Unicode 的类。在 JDK1.4 中, 添加了 NIO 类就是为了改进性能及功能。因此, 在充分理解 Java I/O 系统以便正确的运用之前, 我们需要学习相当数量的类。另外, 很有必要理解 I/O 类库的演化过程。



File 类它既能表示特定的文件的名称, 又能代表一个目录下的一组文件的名称。

如果它指的是一个文件集, 我们就可以对它使用 list() 方法, 该方法会返回一个字符数组, 之所以返回值是数组而不是其他更灵活的容器类, 因为元素数量是固定的。

## 1 构造方法

```java
File(File parent, String child);
// 从父抽象路径名和子路径名字符串创建新的 File实例。

File(String pathname);
// 通过将给定的路径名字符串转换为抽象路径名来创建新的 File实例。

File(String parent, String child);
// 从父路径名字符串和子路径名字符串创建新的 File实例。

File(URI uri);
// 通过将给定的 file: URI转换为抽象路径名来创建新的 File实例。
```



## 2 静态方法

1. `File # createTempFile(String prefix, String suffix)`

   该方法会在 `${java.io.tmpdir}` 目录下生成 `$prefix$hashcode$suffix` 文件。

   ```scala
   test("createTempFile(String prefix, String suffix)") {
     println(s"java.io.tmpdir = ${System.getProperty("java.io.tmpdir")}")
     val file: File = File.createTempFile("tmp", ".txt")
     println(file.getAbsolutePath)
     file.deleteOnExit()
   
     val writer: BufferedWriter = new BufferedWriter(new FileWriter(file))
     writer.write("miao\tmiao\tmiao")
     writer.close()
   }
   /*
   /var/folders/6p/dn3tmpz1043_3gfrg8v3hdtr0000gn/T/		     /var/folders/6p/dn3tmpz1043_3gfrg8v3hdtr0000gn/T/tmp2602175943029084198.txt
    */
   ```

2. `File # createTempFile(String prefix, String suffix, File directory)`

   该方法可以指定生成临时文件的目录 `directory`。

   ```scala
   test("createTempFile(String prefix, String suffix, File directory)") {
     val tmpDir: String = "tmp"
     val dir: File = new File(tmpDir)
     val tmpFile: File = File.createTempFile("tmp", ".txt", dir)
     tmpFile.deleteOnExit()
     println(tmpFile.getAbsolutePath)
   
     val writer: BufferedWriter = new BufferedWriter(new FileWriter(tmpFile))
     writer.write("miao\tmiao\tmiao")
     writer.close()
   
     TimeUnit.SECONDS.sleep(30)
   }
   /*
   /Users/sherlock/IdeaProjects/Lang/thing-in-java/tmp/tmp2907321349423116267.txt
   */
   ```

3. `File # listRoots()`

   列出可用的文件系统根。

   ```scala
   test("listRoots()") {
     val filePaths: Array[String] = File.listRoots()
     						.map(_.getAbsolutePath)
     println(filePaths.mkString("\n"))
   }
   /*
   	/
   */
   ```

   

## 3 普通方法

1. `boolean canExecute()/canRead()/canWrite()`

   测试应用程序是否[可执行/可读/可写]此抽象路径名表示的文件。主要和文件的执行权限有关。下面仅测试执行权限:

   ```Scala
   # 注意下面两个文件的执行权限不同
   ➜  tmp ll
   total 8
   -rwxr-xr-x  1 sherlock  staff     0B Feb 10 19:12 file.txt
   -rw-r--r--  1 sherlock  staff    30B Feb 10 19:10 miao.sh
   
   test("canExecute()") {
     val miao: String = "tmp/miao.sh"
     val miaoFile: File = new File(miao)
     println(miaoFile.canExecute)
   
     val file: String = "tmp/file.txt"
     val file1: File = new File(file)
     println(file1.canExecute)
   }
   /*
   	false
   	true
   */
   ```

2. `compareTo()/equals()/==`

   File 类的比较只看路径名, 就算是同一个文件使用相对路径和绝对路径分别表示的 File 比较也是不相等的。

   ```scala
   /*
   	true
   	0
   	true
    */
   test("compare files") {
     val miao: String = "tmp/miao.sh"
     val f1: File = new File(miao)
     val f2: File = new File(miao)
     println(f1 equals f2)
     println(f1 compareTo f2)
     println(f1 == f2)
   }
   ```

   

3. `boolean delete()`

   删除由此抽象路径名表示的文件或目录。

4. `void deleteOnExit()`

   请求在 JVM 终止时删除由此抽象路径名表示的文件或目录。 

5. `String getAbsolutePath()`

   返回此抽象路径名的绝对路径名字符串。 

6. `File getAbsoluteFile()/getAbsoluteFile()/getCanonicalFile()`

   返回此抽象路径名的[绝对路径/规范路径]形式。如果调用的 File 对象使用的是相对路径, 那么这两个方法会返回新的 File 对象。File 类的比较只看路径名是否相同。

   ```scala
   /*
   false
   true
   miaoFile: tmp/miao.sh
   miaoFile.getAbsoluteFile: /Users/sherlock/IdeaProjects/Lang/thing-in-java/tmp/miao.sh
   miaoFile.getCanonicalFile: /Users/sherlock/IdeaProjects/Lang/thing-in-java/tmp/miao.sh
   */
   test("file path") {
     // 这是相对路径
     val miao: String = "tmp/miao.sh"
     val miaoFile: File = new File(miao)
     val absFile: File = miaoFile.getAbsoluteFile
     val canoFile: File = miaoFile.getCanonicalFile
     println(absFile == miaoFile)
     println(absFile == canoFile)
   
     println(s"miaoFile: $miaoFile")
     println(s"miaoFile.getAbsoluteFile: $absFile")
     println(s"miaoFile.getCanonicalFile: $canoFile")
   }
   ```

7. `boolean exists()`

   测试此抽象路径名表示的文件或目录是否存在。  

8. `setExecutable(boolean)/setExecutable(boolean, boolean)`

   为此抽象路径名设置[所有者/(所有者/每个人)]的执行权限。

9. `setLastModified(long time)`

   设置由此抽象路径名命名的文件或目录的最后修改时间。   

10. `setReadable(boolean)/setReadable(boolean, boolean)`

    一种方便的方法来设置[所有者/(所有者/每个人)]对此抽象路径名的读取权限。   

11. `setReadOnly()`

    标记由此抽象路径名命名的文件或目录，以便只允许读取操作。

12. `setWritable(boolean)setWritable(boolean, boolean)`

    一种方便的方法来设置[所有者/(所有者/每个人)]对此抽象路径名的写入权限。

13. `getCanonicalPath()`

    返回此抽象路径名的规范路径名字符串。 例如 /usr/local/./bin, 这种路径会被规范为/usr/local/bin

14. `isAbsolute()/isDirectory()/isFile()/isHidden()`

    测试这个抽象路径名是否是绝对的。

    测试此抽象路径名表示的文件是否为目录。

    测试此抽象路径名表示的文件是否为普通文件。

    测试此抽象路径名命名的文件是否为隐藏文件。

15. `toURI()`

    构造一个表示此抽象路径名的 `file:` URI。 

16. `String[] list()/list(FilenameFilter)`

    返回一个字符串数组，命名由此抽象路径名表示的目录中的文件和目录。

    返回一个字符串数组，命名由此抽象路径名表示的目录中满足指定过滤器的文件和目录。   

17. `File[] listFiles()/listFiles(FileFilter)/listFiles(FilenameFilter)`

    返回一个抽象路径名数组，表示由该抽象路径名表示的目录中的文件。

    返回一个抽象路径名数组，表示由此抽象路径名表示的满足指定过滤器的目录中的文件和目录。

    返回一个抽象路径名数组，表示由此抽象路径名表示的满足指定过滤器的目录中的文件和目录。 

18. `getParent()/getParentFile()`

    返回此抽象路径名的父路径名的字符串，如果此路径名未命名为父目录，则返回null。

    返回此抽象路径名的父路径名的 File 对象, 如果此路径名没有指定父目录, 则返回null。 





