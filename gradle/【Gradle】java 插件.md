# 【Gradle】java 插件

本篇文章主要来学习 Java Gradle 插件相关的知识，因为 Java Gradle 插件相关的内容也是 Android Gradle 插件的基础。使用 Gradle 来构建项目无非就是帮助开发者做一些重复性的工作，如配置第三方依赖、编译源文件、单元测试、打包发布等，使用相应的 Grade 插件可方便项目的构建以及一定程度上提高开发效率，下面是今天学习的主要内容：

1. Java Gradle 插件的使用
2. Java 插件约定的项目结构
3. 配置第三方依赖
4. 如何构建 Java 项目
5. SourceSet 源集概念
6. Java 插件可添加的任务
7. Java 插件可添加的属性
8. 多项目构建
9. 发布构件

## 1. Java Gradle 插件的使用

使用一个 Gradle 插件使用的是 Project # apply() 方法：

```java
//java是Java Gradle插件的plugin id
apply plugin:'java'
12
```

使用 Java 插件之后会为当前工程添加默认设置和约定，如源代码的位置、单元测试代码的位置、资源文件的位置等，一般使用默认设置即可。

## 2. Java 插件约定的项目结构

Java 插件设置一些默认的设置和约定，下面来看一看 Java 项目的默认工程目录，目录结构基本如下：

```java
JavaGradle
└─src
    ├─main
    │  ├─java
    │  └─resources
    └─test
        ├─java
        └─resources
```

上面目录结构中:

- src/main/java 默认是源码存放目录
- src/main/resources 是资源文件、配置文件等目录
- src/test/java 单元测试源码存放目录
- src/test/resources 是单元测试资源文件、配置文件等目录

main 和 test 是 Java Gradle 插件内置的两个源代码集合，当然除此之外可以自己定义其他源代码集合，定义方式如下：

```java
apply plugin : 'java'

sourceSets{
    //指定新的源代码集合
  	code {
      
    }
}
```

然后在 src 下面创建相应的 java 和 resources 目录，具体目录下存放的文件内容与默认类似，默认目录结构如下：

```java
// 源代码目录
src/code/java
// 资源文件目录
src/code/resource
```

上述目录结构都是 Java Gradle 插件默认实现的，当然还可以修改具体目录的位置，配置方式如下：

```java
sourceSets{
    //修改默认目录，下面还是和默认位置一样，如需配置其他目录修改即可
    main{
        java{
            srcDir 'src/java'
        }
        
        resources{
            srcDir 'src/resources'
        }
        
    }
}
```

## 3. 配置第三方依赖

开发过程中总会使用第三方 jar 包，此时就要对 jar 包进行依赖配置，依赖的仓库类型以及仓库的位置，具体依赖时 Gradle 就会从配置的仓库中去寻找相关依赖，配置仓库和配置具体依赖参考如下：

```java
//配置仓库位置
repositories{
    //仓库类型如jcenter库、ivy库、maven中心库、maven本地库、maven私服库等
    mavenCentral()
    mavenLocal()
    maven {
        uri "http"//xxxx
    }
    jcenter()
    google()
    //...
}

//配置具体的依赖
dependencies{
    //依赖三要素：group(组别)、name(名称)、version(版本)
    //分别对应Maven中的GAV(groupid、artifactid、version)
    
    //完整写法
    compile group: 'com.squareup.okhttp3', name: 'okhttp', version:'3.0.1'
    //简写(不能有空格)
    compile 'com.squareup.okhttp3:okhttp:3.0.1'
}
```

上述代码中 compile 是一个编译时依赖，Gradle 还提供了其他依赖，具体参考如下：

```java
compile：编译时依赖
runtime：运行时依赖
testCompile：测试时编译时依赖
testRuntime：仅在测试用例运行时依赖
archives：发布构件时依赖，如jar包等
default：默认依赖配置
123456
```

Gradle 3.0 之后有 implementation、api 来替代 compile, 具体参考[【Gradle】3.0后新的依赖关系](../[Gradle]3.0后新的依赖关系.md)

Java Gradle 插件还支持为不同的源代码集合指定不同的依赖，具体参考如下：

```java
//为不同的源代码集合配置不同的依赖
dependencies{
    //配置格式：sourceSetCompile、sourceSetRuntime
    mainCompile 'com.squareup.okhttp3:okhttp:3.0.1'
    codeCompile 'com.squareup.okhttp3:okhttp:2.0.1'
}
```

上面介绍的是某个外部库的依赖，除此之外在开发中还会遇到 Module 的依赖、文件的依赖，实际上 Module 的依赖实际上就是某个子项目的依赖，文件以来一般就是 jar 包的依赖。

在Gradle系列之构建脚本基础这篇文章中已经知道构建某个子项目必须在 setting.gradle 文件中使用 include 关键字将子项目引入进来，这里也是一样，现在 setting.gradle 文件中要依赖的项目引入进来，然后按照如下方式依赖某个子项目，具体如下：

```java
//依赖一个子项目
dependencies{
    //setting.gradle文件中 include ':childProject'  
    //依赖子项目
    compile project('childProject')
}
```

文件依赖主要就是 jar 包的依赖，一般都是将 jar 包放在项目的 libs 目录下，然后在配置依赖 jar 包，具体参考如下：

```java
//依赖一个jar包
dependencies{
    //配置单个jar
    compile file('libs/java1.jar')
    //配置多个jar
    compile files('libs/java1.jar','libs/java2.jar')
    //批量配置jar,配置好jar所在路径，会将后缀为jar的所有文件依赖到项目中
    compile fileTree(dir:'libs',include:'*.jar')
}
```

## 4. 如何构建 Java 项目

Gradle 中所有的操作都是基于任务的，Java Gradle 插件同样内置了一系列的任务帮助我们构建项目，执行 build 任务 Gradle 就开始构建当前项目了，可以使用 gradlew build 开始执行构建任务，Gradle 会编译源代码文件、处理资源文件、生成 jar 包、编译测试代码、运行单元测试等。

如在 Android 开发中有个 clean 任务，执行 clean 操作就会删除 build 文件夹以及其他构建项目生成的文件，如果编译出错可以尝试向 clean 然后 build。此外还有 check 任务，该任务会在单元测试的时候使用到，javadoc 任务可以方便生成 Java 格式的 doc api 文档，学习 Java 项目的构建目的还是为了学习 Android 项目构建做准备，所以如何使用 Gradle 构建一个 Java 项目就到此为止。

任务具体参考: [【Gradle】deep in gradle|01](../[Gradle]deep in gradle|01)

## 5. SourceSet 源集概念

这一小节来认识一下 SourceSet ，这也就是前问中提到的源代码集合，它是 Java Gradle 插件用来描述和管理源代码及其资源的一个抽象概念，是一个 Java 源代码文件和资源文件的集合，故可以通过源代码集合配置源代码文件的位置、设置源代码集合的属性等，源集可以针对不同的业务将源代码分组管理，如 Java Gradle 插件默认提供的 main 和 test 源代码目录，一个用于业务代码，另一个用于单元测试，非常方便。

Java Gradle 插件在 Project 下提供一个 sourceSet 属性以及 sourceSet{} 闭包来访问和配置源集，sourceSet 是一个 SourceSetContainer, 源集的常用属性如下：

```groovy
//比如main、test等表示源集的名称
name(String)

//表示编译后源集所需的classpath
compileClasspath(FileCollection)
//表示源码编译后的class文件目录
output.classDir(File)
//表示编译后生成的资源文件目录
output.resourcesDir(File)

//表示该源集的Java源码所在目录
java.srcDirs(Set)
//表示该源集的资源文件所在目录
resources.srcDirs(Set)

//表示该源集的Java源文件对象
java(SourceDirectorySet)
//表示源集的资源文件
resources(SourceDirectorySet)
```

下面是设置 main 这个源集的输出目录，参考如下：

```java
//设置某个源集的属性
sourceSets{
    main{
        //源集名称只读
        println name
        //其他属性设置
        //从4.0开始已经被过时。替代的是dir
        output.classesDir = file("target/classes")
//        output.dir("a/b")
        output.resourcesDir = file("target/classes")
        //....
    }
}
```

## 6. Java 插件可添加的任务

项目的构建还是通过一系列 Gradle 插件提供的任务，下面是 Java 项目中常用的任务，具体如下：

| 任务名称                 | 类型        | 描述                                                         |
| ------------------------ | ----------- | ------------------------------------------------------------ |
| 默认源集通用任务         |             |                                                              |
| compileJava              | JavaCompile | 表示使用javac编译java源文件                                  |
| processResources         | Copy        | 表示把资源文件复制到生成的资源文件目录中                     |
| classes                  | Task        | 表示组装产生的类和资源文件目录                               |
| compileTestJava          | JavaCompile | 表示使用javac编译测试java源文件                              |
| processTestResources     | Copy        | 表示把资源文件复制到生成的资源文件目录中                     |
| testClasses              | Task        | 表示组装产生的测试类和相关资源文件                           |
| jar                      | Jar         | 表示组装jar文件                                              |
| javadoc                  | Javadoc     | 表示使用javadoc生成Java API文档                              |
| uploadArchives           | Upload      | 表示上传包含Jar的构建，使用archives{}闭包进行配置            |
| clean                    | Delete      | 表示清理构建生成的目录文件                                   |
| cleanTaskName            | Delete      | 表示删除指定任务生成的文件，如cleanJar是删除jar任务生成的文件 |
| 自定义源集任务           |             | （SourceSet是具体的源集名称）                                |
| compileSourceSetJava     | JavaCompile | 表示使用javac编译指定源集的源代码                            |
| processSouceSetResources | Copy        | 表示把指定源集的资源文件复制到生成文件中的资源目录中         |
| sourcesSetClasses        | Task        | 表示组装给定源集的类和资源文件目录                           |

## 7. Java 插件可以添加的属性

Java Gradle 插件中的常用属性都被添加到 Project 中，这些属性可以直接使用，具体如下：

| 属性名称            | 类型               | 描述                                   |
| ------------------- | ------------------ | -------------------------------------- |
| sourceSets          | SourceSetContauner | Java项目的源集，可在闭包内进行相关配置 |
| sourceCompatibility | JavaVersion        | 编译Java源文件使用的Java版本           |
| targetCompatinility | JavaVersion        | 编译生成类的Java版本                   |
| archivesBaseName    | String             | 打包成jar或zip文件的名称               |
| manifest            | Manifest           | 用来访问和配置manifest清单文件         |
| libsDir             | File               | 存放生成的类库目录                     |
| distsDir            | File               | 存放生成的发布的文件目录               |

## 8. 多项目构建

使用 Gradle 进行多个项目的构建，一般都是一个主项目依赖其他的子模块项目，是否构建这些子项目主要在 Setting.gradle 文件中配置，是否使用这些子项目则必须要在主项目中配置项目依赖，上文中接受过依赖的三种方式：库依赖、项目依赖和文件依赖，这里使用到的就是项目依赖，多项目配置中经常使用到 subprojects 和 allprojects ，具体如下：

```groovy
//子项目统一配置
subprojects{
    //配置子项目都使用Java Gradle插件
    apply plugin: 'java'
    //配置子项目都是用Maven中心库
    repositories{
        mavenCentral()
    }
    //其他通用配置
    //...
}
//全部项目统一配置
allprojects{
    //配置所有项目都使用Java Gradle插件
    apply plugin: 'java'
    //配置所有项目都是用Maven中心库
    repositories{
        mavenCentral()
    }
    //其他通用配置
    //...
}
```

## 9. 发布构件

Gradle 构建的产物，一般称之为构件，一个构建可以是一个 jar 包、zip 包等，那么如何发布构件呢，下面介绍如何发布一个 jar 构件到项目本地文件夹或 mavenLocal() 中，具体如下：

```groovy
/**
 * 发布构件到项目文件夹/或mavenLocal()
 */
apply plugin : 'java'
//生成jar的任务，类似于自定义wrapper
task publishJar(type:Jar)
//构件版本
version '1.0.0'
//构件通过artifacts{}闭包配置
artifacts{
    archives publishJar
}
//构件发布上传
uploadArchives{
    repositories{
        flatDir{
            name 'libs'
            dirs "$projectDir/libs"
        }
        //将构建发布到mavenLocal()
        mavenLocal()
    }
}
```

执行 uploadArchives 任务就会在相应的位置生成相应的 jar 文件，执行命令如下：

```java
//执行uploadArchives
gradle uploadArchives
```

执行成功之后，就会在项目的 libs 目录下看到生成的 jar 文件，如下图所示：

![项目目录](https://upload-images.jianshu.io/upload_images/2494569-cc67cd68a6b721c5.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行成功之后，就会在用户的 .m2/repository 目录下看到生成的 jar 文件，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190513223811364.png)

下面是如何将 jar 发布到搭建的 maven 私服上，代码参考如下：

```groovy
/**
 * 发布构件到maven私服
 */
apply plugin : 'java'
apply plugin : 'maven'
//生成jar的任务，类似于自定义wrapper
task publishJar(type:Jar)
//构件版本
version '1.0.0'
//构件通过artifacts{}闭包配置
artifacts{
    archives publishJar
}
//构件发布上传
uploadArchives{
    repositories{
        mavenDeployer{
            repository(url:'http://xxx'){
                //仓库用户名和密码
                authentication(userName:'username',password:'pass')
            }
            snapshotRepository(url:'http://xxx'){
                authentication(userName:'username',password:'pass')
            }
        }
    }
}
```

上传过程也是执行 uploadArchives 任务，有 maven 私服的可以尝试一下，关于 Java Gradle 插件的学习就到此为止

