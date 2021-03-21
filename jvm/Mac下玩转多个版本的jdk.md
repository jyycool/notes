## Mac下玩转多个版本的jdk

因为 JDK8 的 jmap -heap 在Mac下使用有BUG 问题, 官方建议使用 JDK10, 于是直接下载了 JDK11.0.7, 但是因为很多 框架根据, 比如 Hadoop, Zookeeper等启动还是要依赖 JDK8, 所以想在Mac系统下使用多JDK环境

### java_home工具

macOS/OS X从很早之前就自带了检查JDK安装路径的工具，即`/usr/libexec/java_home`。如果直接执行，就返回当前的 ***$JAVA_HOME***:

```sh
➜  ~ /usr/libexec/java_home
/Library/Java/JavaVirtualMachines/jdk-11.0.7.jdk/Contents/Home
```

加上-V参数，就可以列出所有的JDK安装路径：

```sh
➜  ~ /usr/libexec/java_home -V
Matching Java Virtual Machines (3):
    11.0.7, x86_64:	"Java SE 11.0.7"	/Library/Java/JavaVirtualMachines/jdk-11.0.7.jdk/Contents/Home
    1.8.0_251, x86_64:	"Java SE 8"	/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
    1.8.0_161, x86_64:	"Java SE 8"	/Library/Java/JavaVirtualMachines/jdk1.8.0_161.jdk/Contents/Home
```

加上-v参数并指定版本，就返回特定版本的JDK安装路径：

```sh
➜  ~ /usr/libexec/java_home -v 1.8
/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
➜  ~ /usr/libexec/java_home -v 11
/Library/Java/JavaVirtualMachines/jdk-11.0.7.jdk/Contents/Home
```

利用它就可以方便地在各个版本之间切换了。

### 修改环境变量

执行`vim ~/.bash_profile`，然后加入如下内容：

```sh
export JAVA_8_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home
export JAVA_11_HOME=/Library/Java/JavaVirtualMachines/jdk-11.0.9.jdk/Contents/Home
export JAVA_HOME=$JAVA_8_HOME
alias jdk8='export JAVA_HOME=$JAVA_8_HOME'
alias jdk11='export JAVA_HOME=$JAVA_11_HOME'
```

之后在命令行中执行 `jdk8/11` 别名命令，就可以在JDK之间切换。如果是使用ZSH的话，就在.zshrc中加入上面的内容，或者直接写上`source ~/.bash_profile`即可。然后重启iTerm2即可

### 验证

从 JDK8 切换到 JDK11 再切回JDK8

```sh
➜  ~ java -version
java version "1.8.0_251"
Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
➜  ~ jdk11
➜  ~ java -version
java version "11.0.7" 2020-04-14 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.7+8-LTS)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.7+8-LTS, mixed mode)
➜  ~ echo $JAVA_HOME
/Library/Java/JavaVirtualMachines/jdk-11.0.7.jdk/Contents/Home
➜  ~ jdk8
➜  ~ java -version
java version "1.8.0_251"
Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
➜  ~ echo $JAVA_HOME
/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
```

