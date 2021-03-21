# 【Giter8】项目模板工具 Giter8



在学习时经常需要写一些 Demo 或者 开发一些小型项目, 重复使用 Idea, 来创建工程之后需要再配置配置文件 (例如 maven 的 pom.xml 或 sbt 的 build.sbt); 为此, 经过调研后选择使用`giter8`来创建项目骨架模板, 它相比 maven 更细致更加丰富了项目的模板.



## 安装giter8

仅介绍OSX的安装方法

```sh
brew install giter8
```

> 如果安装过程中出现如下错误
>
> ```sh
> Error: giter8 has been disabled because it fetches unversioned dependencies at runtime!
> ```
>
> 解决方法:
>
> ```sh
> brew eidt giter8
> ```
>
> 然后删除这一行内容
>
> ```
> disable! because: "fetches unversioned dependencies at runtime"
> ```
>
> 之后保存退出, 再执行安装命令
>
> ```sh
> brew install giter8
> ```
>
> 之后成功安装 giter8

## 用法

模板存储库必须位于GitHub上，并使用 后缀 `.g8`。 这是 [Wiki上的模板列表 ](https://github.com/foundweekends/giter8/wiki/giter8-templates)。 

如果要使用一个模板, 例如 [unfiltered/unfiltered.g8](https://github.com/unfiltered/unfiltered.g8):

```sh
$ g8 unfiltered/unfiltered.g8:
```

Giter8 将其翻译为 GitHub上 `unfiltered/unfiltered.g8` 仓库并向 GitHub查询项目模板参数。另外，也可以使用git仓库全名  

```sh
$ g8 https://github.com/unfiltered/unfiltered.g8.git
```

之后系统会提示您输入该项目的名称即方括号中的值：  

```
name [my_project_name]: 
```

输入您自己的值或直接按Enter键接受默认值。 之后提供了值，giter8获取模板，应用参数，并将其写入您的文件系统。   

如果命令行具有 `name`参数，giter8 会在当前目录下创建一个名为 name (你自己指定的项目名) 的目录, 该目录就是你项目的根目录. 若不指定 `name` 参数, 则当前所在目录即为项目根目录, 目录名即为项目名.

一旦你熟悉了giter8 模板参数, 就可以在命令行直接使用 `name` 参数指定项目名称以及目录, 而不需要使用傻傻的交互式界面：  

```sh
$ g8 unfiltered/unfiltered.g8 --name=my_project_name
```

## 常用模板

Here are some templates maintained by the developers of the projects that are up to date.

#### foundweekends

- [foundweekends/giter8.g8](https://github.com/foundweekends/giter8.g8) (A template for Giter8 templates)

#### Scala

- [scala/scala-seed.g8](https://github.com/scala/scala-seed.g8) (Seed template for Scala)
- [scala/hello-world.g8](https://github.com/scala/hello-world.g8) (A minimal Scala application)
- [scala/scalatest-example.g8](https://github.com/scala/scalatest-example.g8) (A template for trying out ScalaTest)

#### Akka

- [akka/akka-quickstart-scala.g8](https://github.com/akka/akka-quickstart-scala.g8) (Akka Quickstart with Scala)
- [akka/akka-quickstart-java.g8](https://github.com/akka/akka-quickstart-java.g8) (Akka Quickstart with Java)
- [akka/akka-http-quickstart-scala.g8](https://github.com/akka/akka-http-quickstart-scala.g8) (Akka HTTP Quickstart in Scala)
- [akka/akka-http-quickstart-java.g8](https://github.com/akka/akka-http-quickstart-java.g8) (Akka HTTP Quickstart in Java)

#### Play

- [playframework/play-scala-seed.g8](https://github.com/playframework/play-scala-seed.g8) (Play Scala Seed Template)
- [playframework/play-java-seed.g8](https://github.com/playframework/play-java-seed.g8) (Play Java Seed template)

#### Scala Native

- [scala-native/scala-native.g8](https://github.com/scala-native/scala-native.g8) (Scala Native)
- [portable-scala/sbt-crossproject.g8](https://github.com/portable-scala/sbt-crossproject.g8) (sbt-crosspoject)

#### http4s

- [http4s/http4s.g8](https://github.com/http4s/http4s.g8) (http4s services)

#### Scalatra

- [scalatra/scalatra.g8](https://github.com/scalatra/scalatra.g8) (Basic Scalatra project template)

#### Spark

- [holdenk/sparkProjectTemplate.g8](https://github.com/holdenk/sparkProjectTemplate.g8) (Template for Scala [Apache Spark](http://www.spark-project.org) project).

#### Spark Job Server

- [spark-jobserver/spark-jobserver.g8](https://github.com/spark-jobserver/spark-jobserver.g8) (Template for Spark Jobserver)

#### Fast data / Spark / Flink

- [imarios/frameless.g8](https://github.com/imarios/frameless.g8) (A simple [frameless](https://github.com/adelbertc/frameless) template to start with more expressive types for [Spark](https://github.com/apache/spark))
- [tillrohrmann/flink-project.g8](https://github.com/tillrohrmann/flink-project.g8) ([Flink](http://flink.apache.org) project.)
- [nttdata-oss/basic-spark-project.g8](https://github.com/nttdata-oss/basic-spark-project.g8) ([Spark](https://spark.incubator.apache.org/) basic project.)

#### ScalaFX

- [scalafx/scalafx.g8](https://github.com/scalafx/scalafx.g8) (Creates a ScalaFX project with build support of sbt and dependencies.)
- [sfxcode/sapphire-sbt.g8](https://github.com/sfxcode/sapphire-sbt.g8) (Creates MVC ScalaFX App based on [sapphire-core](https://sfxcode.github.io/sapphire-core)

#### sbt-typescript

- [joost-de-vries/play-angular-typescript.g8](https://github.com/joost-de-vries/play-angular-typescript.g8) (Play Typescript Angular2 application)
- [joost-de-vries/play-reactjs-typescript.g8](https://github.com/joost-de-vries/play-reactjs-typescript.g8) (Play Typescript ReactJs Typescript)

#### React

- [ddanielbee/typescript-react.g8](https://github.com/ddanielbee/typescript-react.g8)(React + Typescript application)



#### 参考文件:

- [官方文档](http://www.foundweekends.org/giter8/testing.html)

