# 【SBT】sbt 入门指南

#sbt

[*SBT*](https://www.scala-sbt.org/) 是 Scala 的构建工具，全称是 Simple Build Tool， 类似 Maven 或 Gradle。 SBT 的野心很大，采用Scala编程语言本身编写配置文件，这使得它稍显另类，虽然增强了灵活性，但是对于初学者来说同时也增加了上手难度。另外由于SBT默认从国外下载依赖，导致第一次构建非常缓慢，使用体验非常糟糕！ 如果你是一名Scala初学者，本文希望帮你减轻一些第一次使用的痛苦。

本文的主要内容是帮助初学者从头到尾构建并运行一个Scala项目，重点在于讲解国内镜像仓库的配置。对于每一个操作步骤，会分别针对Windows、Mac和Linux三个主流操作系统进行讲解， 最终帮助你快速构建一个可运行的Scala开发环境。

## 第一步：安装SBT

单击这里下载 [*SBT 1.3.13*](https://piccolo.link/sbt-1.3.13.zip)，下载完成后解压到指定目录，例如 `~/env/sbt-1.3.13`，然后将 `${sbt_home}/bin` 添加至环境变量 PATH。SBT 1.3.13 采用 [Coursier ](https://github.com/coursier/coursier)以无锁的方式并行下载依赖，极大地提升了使用体验！

> 请确认本机已安装 Java 运行环境。

 

## 第二步：设置国内仓库，加快构建过程

### 2.1 设置全局仓库配置

首先在 Home 目录下创建 *.sbt* 目录。

如果是Windows系统，则进入CMD执行如下命令：

```
cd  C:\Users\USER_NAME
mkdir  .sbt
cd  .sbt
```

如果是Mac或Linux系统，则进入Bash执行如下命令：

```
cd  ~
mkdir  .sbt
cd  .sbt
```

然后创建 *repositories* 文件内容如下，并将文件拷贝到 .sbt 目录下，

```
[repositories]
  local
  osc: http://maven.oschina.net/content/groups/public/ ,allowInsecureProtocol
  aliyun: http://maven.aliyun.com/nexus/content/groups/public/ ,allowInsecureProtocol
  aliyun2: https://maven.aliyun.com/repository/public/
  huaweicloud-maven: https://repo.huaweicloud.com/repository/maven/
  jboss: http://repository.jboss.org/nexus/content/groups/public/ ,allowInsecureProtocol
  maven-central: https://repo1.maven.org/maven2/
  typesafe: http://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly ,allowInsecureProtocol
  sbt-plugin-repo: https://repo.scala-sbt.org/scalasbt/sbt-plugin-releases, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
```

### 2.2 设置所有项目均使用全局仓库配置，忽略项目自身仓库配置

该参数可以通过 *Java System Property* 进行设置。在 SBT 中，有三种方法可以设置 *Java System Property*，可以根据需要自行选择。

#### 方法一：**修改SBT配置文件**（推荐）

> 提醒一下， *sbt-1.3.13/conf/* 目录下有两个配置文件， *sbtconfig.txt* 仅适用于 *Windows* 平台，而 *sbtopts* 仅适用于 *Mac/Linux* 平台。

针对 *Windows* 平台，打开 *sbt-1.3.0/conf/sbtconfig.txt 文件，在*末尾新增一行，内容如下：

```
-Dsbt.override.build.repos=true
```

针对 *Mac/Linux* 平台，打开 *sbt-1.3.0/conf/sbtopts 文件，在*末尾新增一行，内容如下：

```
-Dsbt.override.build.repos=true
```

#### 方法二： 设置环境变量

在 *Windows* 上通过 *set* 命令进行设置，

```
set SBT_OPTS="-Dsbt.override.build.repos=true"
```

在 *Mac/Linux* 上使用 *export* 命令进行设置，

```
export SBT_OPTS="-Dsbt.override.build.repos=true"
```

#### 方法三： 传入命令行参数

执行 sbt 命令时， 直接在命令后面加上配置参数，

```
sbt -Dsbt.override.build.repos=true
```

> 注意，如果由于某些原因，你的 *repositories* 文件并不在默认的 *.sbt* 目录下，则需要通过 *-Dsbt.repository.config* 指定 *repositories* 文件的具体位置，该参数的三种设置方法同 *-Dsbt.override.build.repos* 。例如采用修改SBT配置文件方式 **（推荐）**，则打开 *${sbt_home}\conf\sbtconfig.txt 文件，在*末尾新增如下内容：
>
> ```sh
> # 设置所有项目均使用全局仓库配置，忽略项目自身仓库配置
> -Dsbt.override.build.repos=true
> 
> # 指定仓库全局配置文件位置(默认是: ~/.sbt/repositories)
> -Dsbt.repository.config=/Users/sherlock/environment/sbt-1.3.13/repo.repositories
> 
> # 设置仓库位置(默认位置${user.home}/.ivy2)
> -Dsbt.ivy.home=/Users/sherlock/environment/repositories/ivy
> 
> ```
>
> 在使用 2019 及更新版 IDEA 创建 SBT 工程的时候, 建议把上述配置添内容加到 SBT 配置选项 VM parameters 中.



## 第三步：构建并运行第一个Scala项目

### 3.1 修改项目SBT构建版本

单击 [*hello-scala*](https://www.playscala.cn/resource/5d7a45e2eeab565f1f3f9614) 下载一个最简单的Scala项目，并解压到指定目录，如 *D:\idea-projects* 。由于SBT 1.3.13包含了多项性能提升，如果是已有的本地项目，请手动将项目的SBT构建版本改成1.3.13 。具体方法为：打开 *project/build.properties* 文件，将内容修改如下：

```
sbt.version = 1.3.13
```

在命令行中切换至 *hello-scala* 目录，执行sbt命令进入 *sbt shell* ，

> 第一次进入 *sbt shell* 时，由于需要下载相关依赖，大概需要几十秒时间，第二次及以后进入 *sbt shell* 会很快。



```sh
➜  hello-scala sbt
Java HotSpot(TM) 64-Bit Server VM warning: Ignoring option MaxPermSize; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: Ignoring option MaxPermSize; support was removed in 8.0
[info] Loading project definition from /Users/sherlock/environment/sbt-1.3.0/examples/hello-scala/project
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[info] Loading settings for project hello-scala from build.sbt ...
[info] Set current project to hello-scala (in build file:/Users/sherlock/environment/sbt-1.3.0/examples/hello-scala/)
[info] Welcome to sbt 1.3.0.
[info] Here are some highlights of this release:
[info]   - Coursier: new default library management using https://get-coursier.io
[info]   - Super shell: displays actively running tasks
[info]   - Turbo mode: makes `test` and `run` faster in interactive sessions. Try it by running `set ThisBuild / turbo := true`.
[info] See https://www.lightbend.com/blog/sbt-1.3.0-release for full release notes.
[info] Hide the banner for this release by running `skipBanner`.
[info] sbt server started at local:///Users/sherlock/.sbt/1.0/server/5b50db49133eda481ee4/sock
sbt:hello-scala>
```

检查当前项目的SBT构建版本是否为1.3.0，

```sh
sbt:hello-scala> sbtVersion
[info] 1.3.0
```

### 3.2 确认全局仓库是否已经覆盖项目自身仓库

```sh
sbt:hello-scala> show overrideBuildResolvers
[info] true
```

确认仓库列表是否与 *~/.sbt/repositories* 文件一致：

```sh
sbt:hello-scala> show fullResolvers
```

```sh
sbt:hello-scala> show fullResolvers
[info] * Raw(ProjectResolver(inter-project, mapped: ))
[info] * maven-local: file:/Users/sherlock/environment/localRepo
[info] * FileRepository(local, Patterns(ivyPatterns=Vector(${ivy.home}/local/[organisation]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)([branch]/)[revision]/[type]s/[artifact](-[classifier]).[ext]), artifactPatterns=Vector(${ivy.home}/local/[organisation]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)([branch]/)[revision]/[type]s/[artifact](-[classifier]).[ext]), isMavenCompatible=false, descriptorOptional=false, skipConsistencyCheck=false), FileConfiguration(true, None))
[info] * aliyun: http://maven.aliyun.com/nexus/content/groups/public
[info] * huaweicloud-maven: https://repo.huaweicloud.com/repository/maven/
[info] * maven-central: https://repo1.maven.org/maven2/
[info] * URLRepository(sbt-plugin-repo, Patterns(ivyPatterns=Vector(https://repo.scala-sbt.org/scalasbt/sbt-plugin-releases/[organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]), artifactPatterns=Vector(https://repo.scala-sbt.org/scalasbt/sbt-plugin-releases/[organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]), isMavenCompatible=false, descriptorOptional=false, skipConsistencyCheck=false), false)
[success] Total time: 0 s, completed 2020年9月30日 上午6:29:59
sbt:hello-scala>
```

### 3.3 编译并运行

确认无误后执行编译命令，

```sh
➜  hello-scala sbt
Java HotSpot(TM) 64-Bit Server VM warning: Ignoring option MaxPermSize; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: Ignoring option MaxPermSize; support was removed in 8.0
[info] Loading project definition from /Users/sherlock/environment/sbt-1.3.0/examples/hello-scala/project
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[info] Loading settings for project hello-scala from build.sbt ...
[info] Set current project to hello-scala (in build file:/Users/sherlock/environment/sbt-1.3.0/examples/hello-scala/)
[info] Welcome to sbt 1.3.0.
[info] Here are some highlights of this release:
[info]   - Coursier: new default library management using https://get-coursier.io
[info]   - Super shell: displays actively running tasks
[info]   - Turbo mode: makes `test` and `run` faster in interactive sessions. Try it by running `set ThisBuild / turbo := true`.
[info] See https://www.lightbend.com/blog/sbt-1.3.0-release for full release notes.
[info] Hide the banner for this release by running `skipBanner`.
[info] sbt server started at local:///Users/sherlock/.sbt/1.0/server/5b50db49133eda481ee4/sock
sbt:hello-scala> compile
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[info] Compiling 1 Scala source to /Users/sherlock/environment/sbt-1.3.0/examples/hello-scala/target/scala-2.12/classes ...
[info] Non-compiled module 'compiler-bridge_2.12' for Scala 2.12.8. Compiling...
[info]   Compilation completed in 11.97s.
[success] Total time: 16 s, completed 2020年9月30日 上午6:34:45
sbt:hello-scala>
```

查看SBT本地缓存，确认一下是否从国内仓库下载依赖。针对不同的操作系统，对应的缓存路径如下：

- Windows缓存路径是 *%LOCALAPPDATA%\Coursier\Cache\v1* ，即如果用户名是joymufeng，则完整路径是 *C:\Users\joymufeng\AppData\Local\Coursier\Cache\v1* 。
- Linux缓存路径为 *~/.cache/coursier/v1* 。
- Mac缓存路径为 *~/Library/Caches/Coursier/v1 。*

```sh
MacOS的缓存检查
➜  maven2 pwd
/Users/sherlock/Library/Caches/Coursier/v1/https/repo1.maven.org/maven2
➜  maven2 ll
total 0
drwxr-xr-x  12 sherlock  staff  384  9 30 06:09 com
drwxr-xr-x   3 sherlock  staff   96  9 30 06:09 io
drwxr-xr-x   3 sherlock  staff   96  9 30 06:09 jline
drwxr-xr-x   3 sherlock  staff   96  9 30 06:09 net
drwxr-xr-x   9 sherlock  staff  288  9 30 06:09 org
➜  maven2
```

最后执行项目

```sh
sbt:hello-scala> run
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[warn] insecure HTTP request is deprecated 'http://maven.aliyun.com/nexus/content/groups/public'; switch to HTTPS or opt-in as ("aliyun" at "http://maven.aliyun.com/nexus/content/groups/public").withAllowInsecureProtocol(true)
[info] running hello.Hello
Hello, Scala
[success] Total time: 0 s, completed 2020年9月30日 上午6:37:53
sbt:hello-scala>
```

