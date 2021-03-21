# 15 | NIO Files

Java NIO `Files`类(`java.nio.file.Files`)提供了几种操作文件系统中的文件的方法。这个Java NIO `Files`教程将介绍最常用的这些方法。`Files`类包含许多方法，所以如果您需要一个在这里没有描述的方法，那么请检查JavaDoc。`Files`类可能还会有一个方法来实现它。

`java.nio.file.Files`类与`java.nio.file.Path`实例一起工作，因此在处理`Files`类之前，您需要了解`Path`类。

## Files.exists()

`Files.exists()`方法检查给定的`Path`在文件系统中是否存在。

可以创建在文件系统中不存在的`Path`实例。例如，如果您计划创建一个新目录，您首先要创建相应的`Path`实例，然后创建目录。

由于`Path`实例可能指向，也可能没有指向文件系统中存在的路径，你可以使用`Files.exists()`方法来确定它们是否存在(如果需要检查的话)。

这里是一个Java `Files.exists()`的例子：



```csharp
Path path = Paths.get("data/logging.properties");

boolean pathExists = Files.exists(
  path,
  new LinkOption[]{LinkOption.NOFOLLOW_LINKS}
);
```

这个例子首先创建一个`Path`实例指向一个路径，我们想要检查这个路径是否存在。然后，这个例子调用`Files.exists()`方法，然后将`Path`实例作为第一个参数。

注意`Files.exists()`方法的第二个参数。这个参数是一个选项数组，它影响`Files.exists()`如何确定路径是否存在。在上面的例子中的数组包含`LinkOption.NOFOLLOW_LINKS`，这意味着`Files.exists()`方法不应该在文件系统中跟踪符号链接，以确定文件是否存在。

## Files.createDirectory()

`Files.createDirectory()`方法，用于根据`Path`实例创建一个新目录，下面是一个`Files.createDirectory()`例子：



```csharp
Path path = Paths.get("data/subdir");

try {
    Path newDir = Files.createDirectory(path);
} catch(FileAlreadyExistsException e){
    // 目录已经存在
} catch (IOException e) {
    // 其他发生的异常
    e.printStackTrace();
}
```

第一行创建表示要创建的目录的`Path`实例。在`try-catch`块中，用路径作为参数调用`Files.createDirectory()`方法。如果创建目录成功，将返回一个`Path`实例，该实例指向新创建的路径。

如果该目录已经存在，则是抛出一个`java.nio.file.FileAlreadyExistsException`。如果出现其他错误，可能会抛出`IOException`。例如，如果想要的新目录的父目录不存在，则可能会抛出`IOException`。父目录是您想要创建新目录的目录。因此，它表示新目录的父目录。

## Files.copy()

`Files.copy()`方法从一个路径拷贝一个文件到另外一个目录，这里是一个Java `Files.copy()`例子：



```csharp
Path sourcePath      = Paths.get("data/logging.properties");
Path destinationPath = Paths.get("data/logging-copy.properties");

try {
    Files.copy(sourcePath, destinationPath);
} catch(FileAlreadyExistsException e) {
    // 目录已经存在
} catch (IOException e) {
    // 其他发生的异常
    e.printStackTrace();
}
```

首先，该示例创建一个源和目标`Path`实例。然后，这个例子调用`Files.copy()`，将两个`Path`实例作为参数传递。这可以让源路径引用的文件被复制到目标路径引用的文件中。

如果目标文件已经存在，则抛出一个`java.nio.file.FileAlreadyExistsException`异常。如果有其他错误，则会抛出一个`IOException`。例如，如果将该文件复制到不存在的目录，则会抛出`IOException`。

## 重写已存在的文件

可以强制`Files.copy()`覆盖现有的文件。这里有一个示例，演示如何使用`Files.copy()`覆盖现有文件。



```csharp
Path sourcePath      = Paths.get("data/logging.properties");
Path destinationPath = Paths.get("data/logging-copy.properties");

try {
    Files.copy(sourcePath, destinationPath,
            StandardCopyOption.REPLACE_EXISTING);
} catch(FileAlreadyExistsException e) {
    // 目标文件已存在
} catch (IOException e) {
    // 其他发生的异常
    e.printStackTrace();
}
```

请注意`Files.copy()`方法的第三个参数。如果目标文件已经存在，这个参数指示`copy()`方法覆盖现有的文件。

## Files.move()

Java NIO `Files`还包含一个函数，用于将文件从一个路径移动到另一个路径。移动文件与重命名相同，但是移动文件既可以移动到不同的目录，也可以在相同的操作中更改它的名称。是的,`java.io.File`类也可以使用它的`renameTo()`方法来完成这个操作，但是现在已经在`java.nio.file.Files`中有了文件移动功能。

这里有一个Java `Files.move()`例子：



```csharp
Path sourcePath      = Paths.get("data/logging-copy.properties");
Path destinationPath = Paths.get("data/subdir/logging-moved.properties");

try {
    Files.move(sourcePath, destinationPath,
            StandardCopyOption.REPLACE_EXISTING);
} catch (IOException e) {
    //移动文件失败
    e.printStackTrace();
}
```

首先创建源路径和目标路径。源路径指向要移动的文件，而目标路径指向文件应该移动到的位置。然后调用`Files.move()`方法。这会导致文件被移动。

请注意传递给`Files.move()`的第三个参数。这个参数告诉`Files.move()`方法来覆盖目标路径上的任何现有文件。这个参数实际上是可选的。

如果移动文件失败，`Files.move()`方法可能抛出一个`IOException`。例如，如果一个文件已经存在于目标路径中，并且您已经排除了`StandardCopyOption.REPLACE_EXISTING`选项，或者被移动的文件不存在等等。

## Files.delete()

`Files.delete()`方法可以删除一个文件或者目录。下面是一个Java `Files.delete()`例子：



```cpp
Path path = Paths.get("data/subdir/logging-moved.properties");

try {
    Files.delete(path);
} catch (IOException e) {
    // 删除文件失败
    e.printStackTrace();
}
```

首先，创建指向要删除的文件的`Path`。然后调用`Files.delete()`方法。如果`Files.delete()`由于某种原因不能删除文件(例如，文件或目录不存在)，会抛出一个`IOException`。

## Files.walkFileTree()

`Files.walkFileTree()`方法包含递归遍历目录树的功能。`walkFileTree()`方法将`Path`实例和`FileVisitor`作为参数。`Path`实例指向您想要遍历的目录。`FileVisitor`在遍历期间被调用。

在我解释遍历是如何工作之前，这里我们先了解`FileVisitor`接口:

```java
public interface FileVisitor {
    public FileVisitResult preVisitDirectory(
        Path dir, BasicFileAttributes attrs) throws IOException;

    public FileVisitResult visitFile(
        Path file, BasicFileAttributes attrs) throws IOException;

    public FileVisitResult visitFileFailed(
        Path file, IOException exc) throws IOException;

    public FileVisitResult postVisitDirectory(
        Path dir, IOException exc) throws IOException {
}
```

您必须自己实现`FileVisitor`接口，并将实现的实例传递给`walkFileTree()`方法。在目录遍历过程中，您的`FileVisitor`实现的每个方法都将被调用。如果不需要实现所有这些方法，那么可以扩展`SimpleFileVisitor`类，它包含`FileVisitor`接口中所有方法的默认实现。

这里是一个`walkFileTree()`的例子：



```java
Files.walkFileTree(path, new FileVisitor<Path>() {
  @Override
  public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
    System.out.println("pre visit dir:" + dir);
    return FileVisitResult.CONTINUE;
  }

  @Override
  public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
    System.out.println("visit file: " + file);
    return FileVisitResult.CONTINUE;
  }

  @Override
  public FileVisitResult visitFileFailed(Path file, IOException exc) throws IOException {
    System.out.println("visit file failed: " + file);
    return FileVisitResult.CONTINUE;
  }

  @Override
  public FileVisitResult postVisitDirectory(Path dir, IOException exc) throws IOException {
    System.out.println("post visit directory: " + dir);
    return FileVisitResult.CONTINUE;
  }
});
```

`FileVisitor`实现中的每个方法在遍历过程中的不同时间都被调用:

在访问任何目录之前调用`preVisitDirectory()`方法。在访问一个目录之后调用`postVisitDirectory()`方法。

调用`visitFile()`在文件遍历过程中访问的每一个文件。它不会访问目录-只会访问文件。在访问文件失败时调用`visitFileFailed()`方法。例如，如果您没有正确的权限，或者其他什么地方出错了。

这四个方法中的每个都返回一个`FileVisitResult`枚举实例。`FileVisitResult`枚举包含以下四个选项:

- CONTINUE 继续
- TERMINATE 终止
- SKIP_SIBLING 跳过同级
- SKIP_SUBTREE 跳过子级

通过返回其中一个值，调用方法可以决定如何继续执行文件。

`CONTINUE`**继续**意味着文件的执行应该像正常一样继续。

`TERMINATE`**终止**意味着文件遍历现在应该终止。

`SKIP_SIBLINGS`**跳过同级**意味着文件遍历应该继续，但不需要访问该文件或目录的任何同级。

`SKIP_SUBTREE`**跳过子级**意味着文件遍历应该继续，但是不需要访问这个目录中的子目录。这个值只有从`preVisitDirectory()`返回时才是一个函数。如果从任何其他方法返回，它将被解释为一个`CONTINUE`继续。

## 文件搜索

这里是一个用于扩展`SimpleFileVisitor`的`walkFileTree()`，以查找一个名为`README.txt`的文件:

```kotlin
Path rootPath = Paths.get("data");
String fileToFind = File.separator + "README.txt";

try {
  Files.walkFileTree(rootPath, new SimpleFileVisitor<Path>() {
    
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
      String fileString = file.toAbsolutePath().toString();
      //System.out.println("pathString = " + fileString);

      if(fileString.endsWith(fileToFind)){
        System.out.println("file found at path: " + file.toAbsolutePath());
        return FileVisitResult.TERMINATE;
      }
      return FileVisitResult.CONTINUE;
    }
  });
} catch(IOException e){
    e.printStackTrace();
}
```

## 递归删除目录

`Files.walkFileTree()`也可以用来删除包含所有文件和子目录的目录。`Files.delete()`方法只会删除一个目录，如果它是空的。通过遍历所有目录并删除每个目录中的所有文件(在`visitFile()`)中，然后删除目录本身(在`postVisitDirectory()`中)，您可以删除包含所有子目录和文件的目录。下面是一个递归目录删除示例:



```java
Path rootPath = Paths.get("data/to-delete");

try {
  Files.walkFileTree(rootPath, new SimpleFileVisitor<Path>() {
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
      System.out.println("delete file: " + file.toString());
      Files.delete(file);
      return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult postVisitDirectory(Path dir, IOException exc) throws IOException {
      Files.delete(dir);
      System.out.println("delete dir: " + dir.toString());
      return FileVisitResult.CONTINUE;
    }
  });
} catch(IOException e){
  e.printStackTrace();
}
```

## 文件类中的其他方法

`java.nio.file.Files`类包含许多其他有用的函数，比如用于创建符号链接的函数、确定文件大小、设置文件权限等等。有关这些方法的更多信息，请查看`java.nio.file.Files`类的JavaDoc。