# 【Gradle】安装部署及配置 IDEA 达到可生产



### 1. 本地 IDEA 配置 Gradle

Gradel 的本地安装不多解释, 下载 zip包解压, 然后在本地环境变量中配置好 GRADLE_HOME, 并将该变量添加到 PATH中. 到这里你会觉得本地安装很容易嘛, 所以你想到配置 Gradel + Idea 来达到生产可用, 于是坑来了.

在 Idea 中配置 Gradle 的时候, 会让你配置一个 Gradle user home 这个变量的值, 请注意, 这个值不是, 不是, 不是指向你本地安装的 gradle 的根路径, 这个值的默认路径是 ${user.home}/.gradle这个文件夹.

Gradle 的设计者考虑到 gradle 会频繁的升级, 因此每个 gradle 工程会带有一个 gradle 目录，这个目录主要作用就是描述当前工程需要哪个版本的 gradle，以及如何下载 gradle. 因此你拿到别人的 gradle 项目，即使本机没有安装gradle, 当你执行以 **gradlew** 开头的命令时会自行下载这个项目想要的gradle版本。那么自行下载的这个 gradle 保存在什么位置呢 ? 答案就是你配置的 gradle user home 的位置, 因为这是 IDEA 强制你配置的, 所以当你是用 IDEA 来管理 gradle 项目的时候, 默认下载 gradle 就是上面你配置的位置. 个人建议这里保留使用 ${user.home}/.gradle 这个值.

紧接着你会发现, 因为国内网络原因, IDEA 在下载 gradle-6.5-bin.zip 这个包根本下不下来, 这时你需要手动去网上下载这个 zip 包, 然就将其复制到 `/Users/sherlock/.gradle/wrapper/dists/gradle-6.5-bin/bdkk7y62x0xsc656yygs55sqc`这个路径下接着你的项目就会 build 成功.

项目 build 成功后会有一个 gradle/wrapper 文件夹, 里面有一个 gradle-wrapper.properties 文件内容如下:

```sh
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-6.5-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

它的意思就是指明了刚刚所下载的 gradle-6.5-bin.zip 保存在什么位置, 这里我建议你在本地环境变量中添加 GRADLE_USER_HOME=${user.home}.gradle 这个值

之后你就可以使用 gradlew 这个命令来管理你的 gradle项目的生命周期.



### 2. 配置国内仓库源

因为 gradle 并没有自己的仓库, 它默认使用的是 mavenCentral() 仓库, 这个仓库因为国内网络原因, 下载很差, 所以我们需要配置国内仓库镜像源.

`Maven`中有全局`settings.xml`，`Gradle`中也有与之对应的`init.gradle`。`init.gradle` 文件在 build 开始之前执行，所以你可以在这个文件配置一些你想预先加载的操作
例如配置 build 日志输出、配置你的机器信息，比如jdk安装目录，配置在build时必须个人信息，比如仓库或者数据库的认证信息，and so on.

#### 加载顺序：

1. 在命令行指定文件，例如：gradle --init-script yourdir/init.gradle -q taskName.你可以多次输入此命令来指定多个init文件
2. 把init.gradle文件放到${user.home}/.gradle/ 目录下.
3. 把以.gradle结尾的文件放到${user.home}/.gradle/init.d/ 目录下.
4. 把以.gradle结尾的文件放到${GRADLE_HOME}/init.d/ 目录下.

如果存在上面的4种方式的2种以上，gradle会按上面的1-4序号依次执行这些文件，如果给定目录下存在多个init脚本，会按拼音a-z顺序执行这些脚本. 类似于build.gradle脚本，init脚本有时groovy语言脚本。

这里我们建议采用第二种方法, 在 init.gradle 中预先配置好国内仓库源, 并将该文件放在${user.home}/.gradle/ 目录下.

```groovy
def gradle = getGradle()
println "***************************************************"
println "Dump gradle information"
println "Version:${gradle.getGradleVersion()}"
println "UserHomeDir:${gradle.getGradleUserHomeDir()}"
println "HomeDir:${gradle.getGradleHomeDir()}"
println "***************************************************"

// 阿里云 maven 服务，阿里云的maven服务应该算是比较靠谱的了吧
def MAVEN_ALIYUN = 'http://maven.aliyun.com/nexus/content/groups/public'

allprojects {
  	// 这里配置的是gradle plugin插件的仓库地址，比如build tools
    buildscript {
        repositories {
          	mavenLocal()
            maven {
                url MAVEN_ALIYUN
            }
          	mavenCentral()
        }
    }
		
  	// 这里配置的是具体项目依赖的仓库地址，比如三方库
    repositories { 
      	mavenLocal()
        maven {
            url MAVEN_ALIYUN
        }
      	mavenCentral()
    }
}
```

