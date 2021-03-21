# 14 | NIO Path

Java Path接口是Java NIO 2更新的一部分，同Java NIO一起已经包括在Java6和Java7中。Java Path接口是在Java7中添加到Java NIO的。Path接口位于`java.nio.file` 包中，所以Path接口的完全限定名称为 `java.nio.file.Path`。

Java Path 实例表示文件系统中的路径。一个路径可以指向一个文件或一个目录。***路径可以是绝对路径，也可以是相对路径。***

- ***绝对路径***包含从文件系统的根目录到它指向的文件或目录的完整路径。
- ***相对路径***包含相对于其他路径的文件或目录的路径。

不要将文件系统路径与某些操作系统中的path环境变量混淆。`java.nio.file.Path`接口与path环境变量没有任何关系。

在许多方面，`java.nio.file.Path`接口类似于`java.io.File`类，但是有一些细微的差别。不过，在许多情况下，您可以使用`Path`接口来替换`File`类的使用。

## 创建一个Path实例

为了使用`java.nio.file.Path`实例必须创建一个`Path`实例。您可以使用`Paths`类(`java.nio.file.Paths`)中的静态方法来创建路径实例，名为`Paths.get()`。下面是一个Java `Paths.get()`示例:



```swift
import java.nio.file.Path;
import java.nio.file.Paths;

public class PathExample {

    public static void main(String[] args) {
        Path path = Paths.get("c:\\data\\myfile.txt");
    }
}
```

请注意示例顶部的两个导入语句。要使用`Path`接口和`Paths`类，我们必须首先导入它们。

其次，注意`Paths.get("c:\\data\\myfile.txt")`方法调用。它是调用`Path`实例的`Paths.get()`方法。换句话说，`Paths.get()`方法是`Path`实例的工厂方法。

### 创建一个绝对路径

创建绝对路径是通过调用`Paths.get()`工厂方法，给定绝对路径文件作为参数来完成的。下面是创建一个表示绝对路径的路径实例的例子:

```csharp
Path path = Paths.get("c:\\data\\myfile.txt");
```

绝对路径是`c:\data\myfile.txt`。在Java字符串中，重复`\`字符是必需的，因为`\`是一个转义字符，这意味着下面的字符告诉我们在字符串中的这个位置要定位什么字符。通过编写`\\`，您可以告诉Java编译器在字符串中写入一个`\`字符。

上面的路径是一个Windows文件系统路径。在Unix系统(Linux、MacOS、FreeBSD等)上，上面的绝对路径可能如下:

```csharp
Path path = Paths.get("/home/jakobjenkov/myfile.txt");
```

绝对路径现在为`/home/jakobjenkov/myfile.txt`.

如果您在Windows机器上使用了这种路径(从`/`开始的路径)，那么路径将被解释为相对于当前驱动器。例如,路径

```undefined
/home/jakobjenkov/myfile.txt
```

可以将其解释为位于C盘驱动器上。那么这条路径就会对应这条完整的路径:

```jsx
C:/home/jakobjenkov/myfile.txt
```



### 创建一个相对路径

相对路径是指从一条路径(基本路径)指向一个目录或文件的路径。一条相对路径的完整路径(绝对路径)是通过将基本路径与相对路径相结合而得到的。

Java NIO `Path`类也可以用于处理相对路径。您可以使用`Paths.get(basePath, relativePath)`方法创建一个相对路径。下面是Java中的两个相对路径示例:



```csharp
Path projects = Paths.get("d:\\data", "projects");

Path file     = Paths.get("d:\\data", "projects\\a-project\\myfile.txt");
```

第一个例子创建了一个Java `Path`的实例，指向路径(目录):`d:\data\projects`，第二个例子创建了一个`Path`的实例，指向路径(文件):`d:\data\projects\a-project\myfile.txt`

当在工作中使用相对路径时，你可以在你的路径字符串中使用两个特殊代码，它们是：

- `.`
- `..`

代码`.`表示“当前目录”，例如，如果你创建了这样一个相对路径：

```csharp
Path currentDir = Paths.get(".");
System.out.println(currentDir.toAbsolutePath());
```

然后，Java `Path`实例对应的绝对路径将是执行上述代码的应用程序的目录。

如果。在路径字符串的中间使用`.`，表示同样的目录作为路径指向那个点。这里有一个例子说明了这一点:

```csharp
Path currentDir = Paths.get("d:\\data\\projects\.\a-project");
```

这条路径将对应于路径:

```kotlin
d:\data\projects\a-project
```

`..`表示“父目录”或者“上一级目录”，这里有一个Path的Java例子表明这一点：

```csharp
Path parentDir = Paths.get("..");
```

这个例子创建的`Path`实例对应于运行该代码的应用程序的父目录。

如果你在路径字符串代码中间使用`..`，它将对应于在路径字符串的那个点上改变一个目录。例如:

```dart
String path = "d:\\data\\projects\\a-project\\..\\another-project";
Path parentDir2 = Paths.get(path);
```

这个示例创建的Java `Path`实例将对应于这个绝对路径:

```kotlin
d:\data\projects\another-project
```

`a-project`目录之后的`..`代码，会修改目录到项目的父目录中，所以这个目录是指向`another-project`目录。

`.`和`..`代码也可以用于两个子富川的合并方法`Paths.get()`中。下面是两个简单演示Java `Paths.get()`的例子：

```csharp
Path path1 = Paths.get("d:\\data\\projects", ".\\a-project");

Path path2 = Paths.get("d:\\data\\projects\\a-project",
                       "..\\another-project");
```

有更多的方法可以使用Java NIO `Path`类来处理相对路径。在本教程中，您将了解到更多相关知识。

## Path.normalize()

`Path`接口的`normalize()`方法可以使路径标准化。标准化意味着它将移除所有在路径字符串的中间的`.`和`..`代码，并解析路径字符串所引用的路径。下面是一个Java `Path.normalize()`示例:



```csharp
String originalPath =
        "d:\\data\\projects\\a-project\\..\\another-project";

Path path1 = Paths.get(originalPath);
System.out.println("path1 = " + path1);

Path path2 = path1.normalize();
System.out.println("path2 = " + path2);
```

这个`Path`示例首先创建一个中间带有`..`代码的路径字符串。然后，这个示例从这个路径字符串创建一个`Path`实例，并将该`Path`实例打印出来(实际上它会打印`Path.toString()`)。

然后，该示例在创建的`Path`实例上调用`normalize()`方法，该实例返回一个新的`Path`实例。这个新的、标准化的路径实例也会被打印出来。

下面是上述示例的输出:

```kotlin
path1 = d:\data\projects\a-project\..\another-project
path2 = d:\data\projects\another-project
```

正如您所看到的，标准化的路径不包含`a-project\..`部分，因为这是多余的。移除的部分不会增加最终的绝对路径。

