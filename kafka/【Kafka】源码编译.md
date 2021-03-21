# 【Kafka】源码编译

## 一、开篇

最近准备研究一下 kafka 的源码，最直接的就是把源码下载下来，通过 debug 进行研究。经过一段折腾，最终把环境给搭建好了，在这里和大家分享一下。

## 二、步骤

### 2.1 安装必备环境

- 安装 Java 和 Gradle
- 安装Idea的 Scala 插件

提示：

> 如果你电脑上没有安装过 Gradle 或者 Scala 可以先去官网上看一下 kafka 依赖的版本，然后下载成一样的，不然在编译过程中还会现在他们依赖的版本。或者修改配置文件改变版本号

最终全部安装完成后是这样的

```sh
# Java
➜  gradle java -version
java version "1.8.0_251"
Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
```

```sh
# Scala
➜  gradle scala -version
Scala code runner version 2.13.2 -- Copyright 2002-2020, LAMP/EPFL and Lightbend, Inc.
```

```sh
# Gradle
➜  gradle gradle -v

------------------------------------------------------------
Gradle 5.6.4
------------------------------------------------------------

Build time:   2019-11-01 20:42:00 UTC
Revision:     dd870424f9bd8e195d614dc14bb140f43c22da98

Kotlin:       1.3.41
Groovy:       2.5.4
Ant:          Apache Ant(TM) version 1.9.14 compiled on March 12 2019
JVM:          1.8.0_251 (Oracle Corporation 25.251-b08)
OS:           Mac OS X 10.15.6 x86_64
```

### 2.2 下载源码

下载 Kafka 的源码, 使用如下命令：

```cpp
$ cd {要存储Kafka源码的路径}
$ git clone https://github.com/apache/kafka.git
```

运行完这个代码后，会直接拉取最新的代码到本地。

> 如果 github 特别慢, 可以使用这个命令去 GitHub 的镜像网站拉取源码
>
> ```
> git clone https://git.sdut.me/apache/kafka.git
> ```
>
> 或者可以直接去 kafka 官网把源码包下载下来

### 2.3 下载 gradle 的 Wrapper 程序套件

切换到源码目录。执行 gradle 命令，主要目的是下载 Gradle 的 Wrapper 程序套件。

```java
cd kafka
gradle
```

> 这里有个坑, 我建议你最好在执行上面命令前, 先把 gradle-6.5-all.zip 这个包下载到本地.
>
> 然后修改 ${kafka源码目录}/gradle/wrapper/gradle-wrapper.properties
>
> 将其中的 distributionUrl 的值替换为你下载到本地的 gradle-6.5-all.zip 的路径.
>
> 例如我的就是:
>
> ```
> distributionUrl=file\:///Users/sherlock/environment/gradle_dists/gradle-6.5-all.zip
> ```

之后出现了如下报错

```sh
➜  kafka260 gradle

> Configure project :
Building project 'core' with Scala version 2.13.2

FAILURE: Build failed with an exception.

* Where:
Build file '/Users/sherlock/IdeaProjects/Sources/kafka260/build.gradle' line: 446

* What went wrong:
A problem occurred evaluating root project 'kafka260'.
> Failed to apply plugin [id 'org.gradle.scala']
   > Could not find method scala() for arguments [build_7pwwtig6kja2qy3v23zc3mgbn$_run_closure5$_closure73$_closure105@4e76abd6] on object of type org.gradle.api.plugins.scala.ScalaPlugin.

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 3s
```

这个错误,真的各种搜不到, 因为根据信息, 错误发生在 build.gradle 的 446 行, 最后实在没办法突发奇想般的把那段代码注释掉, 注释的代码如下:

```groovy
// plugins.withType(ScalaPlugin) {

  //   scala {
  //     zincVersion = versions.zinc
  //   }

  //   task scaladocJar(type:Jar) {
  //     classifier = 'scaladoc'
  //     from "$rootDir/LICENSE"
  //     from "$rootDir/NOTICE"
  //     from scaladoc.destinationDir
  //   }

  //   //documentation task should also trigger building scala doc jar
  //   docsJar.dependsOn scaladocJar

  //   artifacts {
  //     archives scaladocJar
  //   }
// }
```

抱着试一试的心态重新执行命令：

```sh
➜  kafka260 gradle

> Configure project :
Building project 'core' with Scala version 2.13.2
Building project 'streams-scala' with Scala version 2.13.2

> Task :help

Welcome to Gradle 5.6.4.

To run a build, run gradle <task> ...

To see a list of available tasks, run gradle tasks

To see a list of command-line options, run gradle --help

To see more detail about a task, run gradle help --task <task>

For troubleshooting, visit https://help.gradle.org

BUILD SUCCESSFUL in 12m 39s
1 actionable task: 1 executed
```

编译完成！ 有点小激动

### 2.4 将 Kafka 源码引入 IDEA

IDEA 导入项目选择 new --> project from existing source... --> 选择 kafka 源码目录下的 build.gradle

接着为项目配置好 java, scala 环境

最最重要的是在 build.gradle 中找到 repositories{}, 在其中添加 maven国内镜像源, 修改后如下

```java
repositories {
	mavenLocal()
	maven {url "http://maven.aliyun.com/nexus/content/groups/public/"}
	maven {url "http://maven.oschina.net/content/groups/public/"}
	mavenCentral()
	jcenter()
	maven {
		url "https://plugins.gradle.org/m2/"
	}
}
```

之后就等 IDEA 的 gradle 自己去下载编译吧,最后编译完成 IDEA会有如下提示

```sh
> Configure project :
Building project 'core' with Scala version 2.13.2
Building project 'streams-scala' with Scala version 2.13.2

Deprecated Gradle features were used in this build, making it incompatible with Gradle 7.0.
Use '--warning-mode all' to show the individual deprecation warnings.
See https://docs.gradle.org/6.5/userguide/command_line_interface.html#sec:command_line_warnings

CONFIGURE SUCCESSFUL in 6m 3s
```

最后的最后, 贴一张 IDEA 编译后的图

<img src="/Users/sherlock/Desktop/notes/allPics/Kafka/kafka 源码编译完.png" alt="kafka 源码编译完" style="zoom:50%;" />

> PS: gradle 对用惯 maven 的人来说真的不太友好...
>
> 被 gradle 折腾了一整天