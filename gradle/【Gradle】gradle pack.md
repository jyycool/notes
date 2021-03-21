# 【Gradle】gradle pack

打包一直是一个很头疼的问题,  从 maven开始一直是如此, 当开始使用 gradle 来管理项目之后, 这个头疼的问题又继续来了.

现如今项目的打包大概有如下几种需求:

- uber jar, 又称为 fat jar, 它是将项目中所有依赖全部打入 jar 包中, 通常这个包会比你实际 code 的体积更大
- thin jar, 这种 jar 通常只包含, 你 code 的代码, 它会在一个自定义目录下来管理第三方依赖

## 一、Uber jar

通常项目开发完成, publish, 这是最简单粗暴的一种方式. 它只需要在 `build.gradle` 中如下配置

```groovy
jar {
  manifest {
    attributes "Main-Class": "cgs.main.Main"
  }
  from (configurations.runtime.collect { it.isDirectory() ? it : zipTree(it) }) {
    exclude "META-INF/*.SF"
    exclude "META-INF/*.DSA"
    exclude "META-INF/*.RSA"
  }
}
```

或者自定义一个 UberJar 的 task

```groovy
task uberJar(type: Jar, dependsOn: [':compileJava', ':processResources']) {
  manifest {
    attributes "Main-Class": "cgs.main.Main"
  }
  classifier = "uber"
  from {
    configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) }
  } {
    exclude "META-INF/*.SF"
    exclude "META-INF/*.DSA"
    exclude "META-INF/*.RSA"
  }
  with jar

}

artifacts {
  archives uberJar
}

```

第一种配置方式执行 `./gradlew clean assemble` 便会在 `build/libs`

目录下生成一个 uber jar

第二种配置方式执行 `./gradle clean uberJar` 会在 `build/libs` 目录下生成两个 jar, 分别为 uber jar 和 thin jar( 这个 thin jar 是无法运行的, 因为它没有配置 classpath, 会提示找不到依赖)



## 二、executable thin jar

thin jar 的优点是体积够小, 并且它会在外部集中管理第三方依赖, 这里一定要注意是外部. 

首先来看个错误配置, 将依赖目录 lib 打在了 jar 内

```groovy
// 这种打包方式是错误的,他会把所有依赖放在项目.jar 下面的 libs 文件夹内
jar {
  manifest {
    attributes "Class-Path": configurations.runtime.collect { "libs/" + it.getName() }.join(' ')
    attributes "Main-Class": "cgs.main.Main"

    into ("libs/") {
      from configurations.runtime
    }
  }
}
```

当用这个配置生成的 jar, 查看它的目录结构:

```sh
➜  single_module jar tf build/libs/single_module-1.0-SNAPSHOT.jar
META-INF/
META-INF/MANIFEST.MF
cgs/
cgs/main/
cgs/main/Main.class
libs/
libs/commons-lang3-3.8.1.jar
libs/log4j-slf4j-impl-2.4.jar
libs/slf4j-api-1.7.12.jar
libs/log4j-core-2.4.jar
libs/log4j-api-2.4.jar
libs/commons-cli-1.3.1.jar
libs/commons-io-2.4.jar
```

我们可以看到它将所有依赖放在了 jar包内的 libs 目录下, 看似很成功.而当我们运行这个 jar 时, 会发现找不到第三方依赖, 原因就是我们把依赖放在了 jar 内, 而 manifest.mf 中配置的 class-path 路径是项目 jar 包所在目录下的相对路径.

```sh
➜  single_module java -jar build/libs/single_module-1.0-SNAPSHOT.jar     
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/commons/lang3/StringUtils
        at cgs.main.Main.main(Main.java:15)
        ... 1 more

```

所以需要将依赖配置在外部.

正确的配置应该是:

```groovy
// 这是正确的方法, 他创建了一个 thin jar, 并将所有依赖存放在外部的 build/libs/lib 目录下,
// thin jar 存放在 build/libs 目录下, 并且将 classpath 添加到了 manifest.mf 中
jar {
  manifest {
    attributes "Class-Path": configurations.runtime.collect { "lib/" + it.getName() }.join(' ')
    attributes "Main-Class": "cgs.main.Main"
  }
}
task copyJar(type: Copy) {
  from configurations.runtime
  into "$buildDir/libs/lib"
}
task release(type: Copy, dependsOn: [assemble, copyJar])
```

使用这种方式执行打包命令 `./gradlew clean release` 会将所有依赖配置在 `build/libs/lib`  目录下, jar 包则放在了 `build/libs` 目录下

```sh
➜  single_module tree build/libs 
build/libs
├── lib
│   ├── commons-cli-1.3.1.jar
│   ├── commons-io-2.4.jar
│   ├── commons-lang3-3.8.1.jar
│   ├── log4j-api-2.4.jar
│   ├── log4j-core-2.4.jar
│   ├── log4j-slf4j-impl-2.4.jar
│   └── slf4j-api-1.7.12.jar
└── single_module-1.0-SNAPSHOT.jar
```

但是这还有更优的方式, 我们可以生成一个 zip 包, 这个包内包含项目的 thin jar 以及一个包含所有依赖的 lib 目录.

它的配置如下

```go
jar {
  manifest {
    attributes "Class-Path": configurations.runtime.collect { "lib/" + it.getName() }.join(' ')
    attributes "Main-Class": "cgs.main.Main"
  }
}
task copyJar(type: Copy) {
  from configurations.runtime
  into "$buildDir/libs/lib"
}

task distZip(type: Zip, dependsOn: [assemble, copyJar]) {
  from "$buildDir/libs"
  into("lib") {
    from "$buildDir/libs/lib"
  }
}
```

当我们在执行打包目录 `./gradlew clean distZip`  后, 它会在 `build/distributions` 目录下生成一个 zip 包, 这个 zip 包的目录结构其实和上面 `build/libs` 的目录结构几乎相同

```sh
➜  Public tree single_module-1-1.0-SNAPSHOT
single_module-1-1.0-SNAPSHOT
├── lib
│   ├── commons-cli-1.3.1.jar
│   ├── commons-io-2.4.jar
│   ├── commons-lang3-3.8.1.jar
│   ├── log4j-api-2.4.jar
│   ├── log4j-core-2.4.jar
│   ├── log4j-slf4j-impl-2.4.jar
│   └── slf4j-api-1.7.12.jar
└── single_module-1.0-SNAPSHOT.jar
```

解压这个 zip 后我们可以直接运行 jar

```sh
➜  single_module-1-1.0-SNAPSHOT java -jar single_module-1.0-SNAPSHOT.jar
```

另外我们还可以根据需求在这个 zip 包内配置 bin 一些可执行脚本文件.

