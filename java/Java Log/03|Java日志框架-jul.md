# 03|Java日志框架-jul

JUL 全称 Java util Logging 是 java原生的日志框架，使用时不需要另外引用第三方类库，相对其他日志框架使用方便，学习简单，能够在小型应用中灵活使用。

## 一、核心组件

- Loggers

  被称为记录器，应用程序通过获取 Logger 对象，调用其 API 来发布日志信息。Logger 通常时应用程序访问日志系统的入口程序。 

- Appenders

  也被称为 Handlers，每个 Logger 都会关联一组 Handlers，Logger会将日志交给关联 Handlers 处理，由 Handlers 负责将日志做记录。Handlers 在此是一个抽象，其具体的实现决定了日志记录的位置可以是控制台、文件、网络上的其他日志服务或操作系统日志等。

- Layouts

  也被称为 Formatters，它负责对日志事件中的数据进行转换和格式化。Layouts 决定了数据在一条日志记录中的最终形式。 

- Level

  每条日志消息都有一个关联的日志级别。该级别粗略指导了日志消息的重要性和紧迫，我可以将 Level 和 Loggers，Appenders 做关联以便于我们过滤消息。 

- Filters

  过滤器，根据需要定制哪些信息会被记录，哪些信息会被放过。

总结一下就是：

用户使用 Logger 来进行日志记录，Logger 持有若干个 Handler，日志的输出操作是由Handler 完成的。 在 Handler 在输出日志前，会经过 Filter 的过滤，判断哪些日志级别过滤放行哪些拦截，Handler 会将日志内容输出到指定位置（日志文件、控制台等）。Handler在输出日志时会使用 Layout，将输出内容进行排版。

## 二、示例

```java
public class JULTest {

  @Test 
  public void testQuick() throws Exception {

    // 1.创建日志记录器对象 
    Logger logger = Logger.getLogger("cgs.log.JULTest"); 			// 2.日志记录输出 
    logger.info("hello jul");
    logger.log(Level.INFO, "info msg");
    String name = "jack"; Integer age = 18; 
    logger.log(
      Level.INFO, 
      "用户信息：{0},{1}", 
      new Object[]{name, age});
  }
}
```



## 三、日志的级别

jul中定义的日志级别

>- java.util.logging.Level 中定义了日志的级别： 
>SEVERE（最高值） 
>WARNING 
>INFO （默认级别） 
>CONFIG 
>FINE 
>FINER 
>FINEST（最低值）
>- 还有两个特殊的级别：
>OFF，可用来关闭日志记录。 
>ALL，启用所有消息的日志记录。

虽然我们测试了7个日志级别但是默认只实现info以上的级别

```java
@Test 
public void testLogLevel() throws Exception {

  // 1.获取日志对象 
  Logger logger = Logger.getLogger("cgs.log.QuickTest"); 	  
  // 2.日志记录输出 
  logger.severe("severe"); logger.warning("warning");  	
  logger.info("info"); logger.config("cofnig"); 
  logger.fine("fine"); logger.finer("finer"); 
  logger.finest("finest");
}
```

自定义日志级别配置

```java
@Test 
public void testLogConfig() throws Exception {

  // 1.创建日志记录器对象 
  Logger logger = Logger.getLogger("cgs.log.JULTest");

  // 一、自定义日志级别 
  // a.关闭系统默认配置 
  logger.setUseParentHandlers(false); 
  // b.创建handler对象 
  ConsoleHandler consoleHandler = new ConsoleHandler(); 
  // c.创建formatter对象 
  SimpleFormatter simpleFormatter = new SimpleFormatter(); 
  // d.进行关联 
  consoleHandler.setFormatter(simpleFormatter); 	
  logger.addHandler(consoleHandler); 
  // e.设置日志级别 
  logger.setLevel(Level.ALL); consoleHandler.setLevel(Level.ALL);

  // 二、输出到日志文件 
  FileHandler fileHandler = new FileHandler("d:/logs/jul.log"); 
  fileHandler.setFormatter(simpleFormatter); 
  logger.addHandler(fileHandler);

  // 2.日志记录输出 
  logger.severe("severe"); logger.warning("warning"); 
  logger.info("info"); logger.config("config"); 
  logger.fine("fine"); logger.finer("finer"); 
  logger.finest("finest");
}
```



## 四、Logger间的父子关系

JUL 中 Logger 之间存在父子关系，这种父子关系通过树状结构存储，JUL 在初始化时会创建一个顶层 RootLogger 作为所有 Logger 父 Logger，存储上作为树状结构的根节点。并父子关系通过路径来关联。

Logger 类的顶层父类是 LogManager 内部类 RootLogger `java.util.logging.LogManager$RootLogger` 

```java
@Test 
public void testLogParent() throws Exception {

// 日志记录器对象父子关系 
  Logger logger1 = Logger.getLogger("cgs.log"); 
  Logger logger2 = Logger.getLogger("cgs");

  System.out.println(logger1.getParent() == logger2); 
  // 所有日志记录器对象的顶级父元素 class 为java.util.logging.LogManager$RootLogger name为"" 
  System.out.println("logger2 parent:" + logger2.getParent() + "，name：" + logger2.getParent().getName());

	// 一、自定义日志级别 
  // a.关闭系统默认配置 
  logger2.setUseParentHandlers(false); 
  // b.创建handler对象 
  ConsoleHandler consoleHandler = new ConsoleHandler(); 
  // c.创建formatter对象 
  SimpleFormatter simpleFormatter = new SimpleFormatter(); 
  // d.进行关联 
  consoleHandler.setFormatter(simpleFormatter); 
  logger2.addHandler(consoleHandler); 
  // e.设置日志级别 
  logger2.setLevel(Level.ALL); 
  consoleHandler.setLevel(Level.ALL);

	// 测试日志记录器对象父子关系 
  logger1.severe("severe"); 
  logger1.warning("warning"); 
  logger1.info("info"); 
  logger1.config("config"); 
  logger1.fine("fine"); 
  logger1.finer("finer"); 
  logger1.finest("finest");
}
```

## 五、日志的配置文件

默认配置文件路径 `$JAVA_HOME\jre\lib\logging.properties`

```java
@Test 
public void testProperties() throws Exception { 
  // 读取自定义配置文件 
  InputStream in = JULTest.class
    .getClassLoader()
    .getResourceAsStream("logging.properties"); 
  
  // 获取日志管理器对象 
  LogManager logManager = LogManager.getLogManager(); 
  // 通过日志管理器加载配置文件 
  logManager.readConfiguration(in);
  Logger logger = Logger.getLogger("com.itheima.log.JULTest"); 
  logger.severe("severe"); logger.warning("warning"); 
  logger.info("info"); logger.config("config"); 
  logger.fine("fine"); logger.finer("finer"); 
  logger.finest("finest");
}
```

配置文件：

```properties
## RootLogger使用的处理器(默认就是ConsoleHandler） 
handlers = java.util.logging.ConsoleHandler 
# RootLogger日志等级
.level= INFO
##自定义Logger
com.itheima.handlers = java.util.logging.FileHandler 
# 自定义Logger日志等级 
com.itheima.level = INFO 
# 忽略父日志设置 
com.itheima.useParentHandlers = false

## 控制台处理器 ConsoleHandler
# 输出日志级别 
java.util.logging.ConsoleHandler.level = INFO 
# 输出日志格式 
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter

## 文件处理器 FileHandler
# 输出日志级别 
java.util.logging.FileHandler.level = INFO 
# 输出日志格式 
java.util.logging.FileHandler.formatter = java.util.logging.SimpleFormatter

## 输出日志文件路径 
java.util.logging.FileHandler.pattern = /java%u.log 
# 输出日志文件限制大小（50000字节） 
java.util.logging.FileHandler.limit = 50000 
# 输出日志文件限制个数 
java.util.logging.FileHandler.count = 10 
# 输出日志文件 是否是追加 
java.util.logging.FileHandler.append = true
```

## 六、日志原理解析

![](https://img-blog.csdnimg.cn/20200505165057804.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xlb25fSmluaGFpX1N1bg==,size_16,color_FFFFFF,t_70)

1. 初始化 LogManager

   - LogManager 加载 logging.properties 配置

   - 添加 Logger 到 LogManager

2. 从单例 LogManager 获取 Logger

3. 设置级别 Level，并指定日志记录 LogRecord

4. Filter 提供了日志级别之外更细粒度的控制

5. Handler 是用来处理日志输出位置

6. Formatter 是用来格式化LogRecord的

   


