# 【Java】classpath

Java虚拟机规范并没有规定虚拟机应该从哪里寻找类，因此不同的虚拟机实现可以采用不同的方法。Oracle的Java虚拟机实现根 据类路径（class path）来搜索类。按照搜索的先后顺序，类路径可以 分为以下3个部分：

- 启动类路径（bootstrap classpath）
- 扩展类路径（extension classpath） 
- 用户类路径（user classpath）

启动类路径默认对应`jre\lib`目录，Java标准库（大部分在`rt.jar`里） 位于该路径。

扩展类路径默认对应`jre\lib\ext`目录，使用Java扩展机制的类位于这个路径。

我们自己实现的类，以及第三方类库则位于用户类路径。可以通过`-Xbootclasspath`选项修改启动类路径，不过通常并不建议这样做，所以这里就不详细介绍了。

用户类路径的默认值是当前目录，也就是“`.`”。可以设置 `CLASSPATH`环境变量来修改用户类路径，但是这样做不够灵活，所以不推荐使用。更好的办法是给java命令传递`--classpath`（或简写为-cp）选项。`--classpath/-cp`选项的优先级更高，可以覆盖`CLASSPATH`环境变量设置。

`--classpath/-cp` 选项既可以指定 class 文件目录，也可以指定 JAR文件或者 ZIP文件，如下：

```sh
java -cp path/to/classes ...
java -cp path/to/lib1.jar ...
java -cp path/to/lib2.zip ...
```

还可以同时指定多个目录或文件，用分隔符分开即可。分隔符 因操作系统而异。在Windows系统下是分号，在类UNIX（包括 Linux、Mac OS X等）系统下是冒号。例如在Linux下：

```sh
java -cp .:path/toclasses:lib/a.jar:lib/b.jar:lib/c.zip ...
```

从Java 6开始，还可以使用通配符（*）指定某个目录下的所有JAR文件，格式如下：

```sh
java -cp .:lib/* ...
```

