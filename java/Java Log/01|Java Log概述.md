# 01|Java Log概述

作为Java程序员，幸运的是，Java 拥有功能和性能都非常强大的日志库；不幸的是，这样的日志库有不止一个——相信每个人都曾经迷失在 JUL(Java Util Log), JCL(Commons Logging), Log4j, SLF4J, Logback，Log4j2 等等的迷宫中。在我见过的绝大多数项目中，都没有能够良好的配置和使用日志库。

在阿里开发手册中有做如下强制要求：

> 【强制】应用中不可以直接使用日志系统(Log4j, Logback)中的 API, 而应该依赖使用日志门面框架(Slf4J, JCL)的 API, 有利于维护和各个类的日志处理方式的统一。

## 一、什么是简单日志门面?

这么多的日志框架为什么不可以直接使用呢？为什么要用 slf4j/jcl 呢？这里提到的简单日志门面(Simple Logging Facade)又是什么呢？

### 1.1 门面模式

门面模式(Facade Pattern), 也称之为外观模式，其核心为：外部与一个子系统的通信必须通过一个统一的外观对象进行，使得子系统更易于使用。(官方解释很绕啊......)

![](https://img-blog.csdnimg.cn/20181128151611259.png)

就像前面介绍的几种日志框架一样，每一种日志框架都有自己单独的API，要使用对应的框架就要使用其对应的API，这就大大的增加应用程序代码对于日志框架的耦合性。

为了解决这个问题，就是在日志框架和应用程序之间架设一个沟通的桥梁，对于应用程序来说，无论底层的日志框架如何变，都不需要有任何感知。只要门面服务做的足够好，随意换另外一个日志框架，应用程序不需要修改任意一行代码，就可以直接上线。

**在软件开发领域有这样一句话：计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决。而门面模式就是对于这句话的典型实践。**

### 1.2 日志门面框架

当前工业中流行的简单日志门面框架就两种:

1. SLF4J - "Simple Logging Facade for Java", 为java 提供的简单日志Facade。
2. JCL - "Jakarta Commons Logging", 是 Apache 提供的一个通用日志Facade。

#### 1.2.1 jcl

JCL采用了设计模式中的“适配器模式”，它是为“所有的Java日志实现”提供的一个统一的接口，然后在适配类中将对日志的操作委托给具体的日志框架，它**自身也提供一个日志的实现**，但是功能非常弱（**SimpleLog**）。所以一般不会单独使用它。它允许开发人员使用不同的具体日志实现工具：Log4j,jdk自带的日志（JUL）

JCL有两个基本的抽象类：**Log(基本记录器)**和**LogFactory(负责创建Log实例)**

![](https://img-blog.csdnimg.cn/2020081308203837.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1pob3V6aV9oZW5n,size_16,color_FFFFFF,t_70)

##### 1.2.1.1 JCL原理

1. 通过LogFactory动态加载Log实现类

![img](https://img-blog.csdnimg.cn/20200813082800486.png)

2. 日志门面支持的日志实现数组

   ```java
   private static final String[] classesToDiscover = {
       "org.apache.commons.logging.impl.Log4JLogger",
       "org.apache.commons.logging.impl.Jdk14Logger",
       "org.apache.commons.logging.impl.Jdk13LumberjackLogger",
       "org.apache.commons.logging.impl.SimpleLog"
   };
   ```

3. 获取具体的日志实现

   ```java
   for(int i = 0; i < classesToDiscover.length && result == null; ++i){
   	result = this.createLogFromClass(
       								classesToDiscover[i], 
       								logCategory, 
       								true);
   }
   ```

以之前代码为例。这里首先会在 classesToDiscover 属性中寻找，按数组下标顺序进行寻找，开始我们没有加入log4j依赖和配置文件，这里 "org.apache.commons.logging.impl.Log4JLogger",就找不到，会继续在数组中寻找，第二个"org.apache.commons.logging.impl.Jdk14Logger",是JDK自带的，此时就使用它。当我们加入log4j依赖和配置文件后，就直接找到org.apache.commons.logging.impl.Log4JLogger，并使用它。如果需要再加入其它的日志，需要修改原码，并且在log中实现它，扩展性很不好。JCL已经被 apache 淘汰了。

#### 1.2.2 slf4j

当下最为流行的 Java 日志门面框架当仁不让的就是 slf4j, 它也成为日志门面事实上的标准。slf4j 入口就是众多接口的集合，但它不负责具体的日志实现，只在编译时负责寻找合适的日志系统进行绑定。具体有哪些接口，全部都定义在slf4j-api中。查看slf4j-api源码就可以发现，里面除了public final class LoggerFactory类之外，都是接口定义。因此，slf4j-api本质就是一个接口定义。

下图比较清晰的描述了他们之间的关系：

![](https://gss0.baidu.com/-vo3dSag_xI4khGko9WTAnF6hhy/zhidao/wh%3D600%2C800/sign=d55d72b16e380cd7e64baaeb9174810c/63d9f2d3572c11dfcccf3222682762d0f703c291.jpg)

##### 1.2.2.1 SLF4J原理

下面主要针对大家所熟悉的 slf4j 来聊一下门面做了哪些事儿以及如何做到的。

SLF4J共有两大特性:

1. 静态绑定
2. 桥接

###### 1. 静态绑定

我们在 coding 时使用 slf4j 的 api，而不需要关心底层使用的日志框架是什么。在部署时选择具体的日志框架进行绑定。

如下图，我们以 log4j 为例。首先我们的 application 中会使用 slf4j 的 api进行日志记录。我们引入适配层的 jar 包 slf4j-log4j12.jar及底层日志框架实现log4j.jar。简单的说适配层做的事情就是把 slf4j 的 api 转化成 log4j 的 api。通过这样的方式来屏蔽底层框架实现细节。

![click to enlarge](https://imgconvert.csdnimg.cn/aHR0cHM6Ly93d3cuc2xmNGoub3JnL2ltYWdlcy9jb25jcmV0ZS1iaW5kaW5ncy5wbmc)

 

###### 2. 桥接

比如你的 application 中使用了 slf4j，并绑定了 logback。但是项目中引入了一个A.jar，A.jar使用的日志框架是log4j.那么有没有方法让 slf4j 来接管这个 A.jar 包中使用 log4j 输出的日志呢？这就用到了桥接包。你只需要引入 log4j-over-slf4j.jar 并删除 log4j.jar 就可以实现slf4j 对 A.jar 中 log4j 的接管。

听起来有些不可思议。你可能会想如果删除 log4j.jar 那 A.jar 不会报编译错误嘛？答案是不会。因为log4j-over-slf4j.jar实现了 log4j 几乎所有 public 的 api。但关键方法都被改写了。不再是简单的输出日志，而是将日志输出指令委托给slf4j。

![click to enlarge](https://imgconvert.csdnimg.cn/aHR0cHM6Ly93d3cuc2xmNGoub3JnL2ltYWdlcy9sZWdhY3kucG5n)

应用中不应该直接使用具体日志框架中的 api，而应该依赖 slf4j 的 api。使用日志门面编程。具体的日志框架是在部署时确定的，表象上就是通过引用不同的 jar 包来实现 slf4j 到具体日志框架的绑定。

常用的绑定有：

1. log4j

   引入 slf4j-log4j12.jar 和 log4j.jar

2. logback

   引入 logback-classic.jar 和 logback-core.jar

如果你的应用使用的 slf4j+logback 组合，但是引用的其他jar使用的是log4j等其他日志框架如何处理？

log4j用log4j-over-slf4j.jar 替换 log4j.jar

jcl: 用jcl-over-slf4j.jar 替换 commons-logging

