# 【Jar】jar命令详解

JAR 包是 Java 中所特有一种压缩文档, 其实可以把它理解为 zip 包。当然也是有区别的, JAR 包中有一个 `META-INF\MANIFEST.MF` 文件, 当你生成 JAR 包时, 它会自动生成。

JAR 包是由 `JAVA_HOME/bin/jar`命令生成的，当我们安装好 JDK，设置好path 路径，就可以正常使用 `jar` 命令，它会用 `JAVA_HOME/lib/tool.jar` 工具包中的类。



## 一、jar命令参数

jar命令格式

```sh
jar {ctxuf}[vme0Mi] <jar包> 
```

其中 `{ctxu}` 这四个参数必须选选其一。`[vfme0Mi]` 是可选参数，文件名也是必须的。

- `-c` 创建一个jar包
- `-t` 显示jar中的内容列表
- `-x` 解压jar包
- `-u` 添加文件到jar包中
- `-f` 指定jar包的文件名
- `-v` 生成详细的报造，并输出至标准设备
- `-m` 指定 manifest.mf 文件.(manifest.mf文件中可以对jar包及其中的内容作一些一设置)
- `-0` 产生jar包时不对其中的内容进行压缩处理
- `-M` 不产生所有文件的清单文件(Manifest.mf)。这个参数与忽略掉-m参数的设置
- `-i`  为指定的jar文件创建索引文件
- `-C` 表示转到相应的目录下执行jar命令,相当于cd到那个目录，然后不带-C执行jar命令

