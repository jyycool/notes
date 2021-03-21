# 01 | Commons CLI

## 一、overview

Apache Commons CLI 库提供了用于解析传递给程序的命令行选项的 APIs。它还能够打印详细说明命令行工具可用选项的帮助消息。

Commons CLI 支持不同类型的选项： 

- POSIX之类的选项 (即`tar -zxvf foo.tar.gz`)
- GNU喜欢长选项 (例如`du --human-可读--max-depth = 1`)
- 类似Java的属性 (即`java -Djava.awt.headless=true -Djava.net.systemProxies = true Foo`)
- 带值的简短选项 (即`gcc -O2 foo.c`)
- 单连字符的长选项 (即`ant -projecthelp`)

一个典型由 Commons CLI 显示的帮助信息是这样的:

```sh
usage: ls
 -A,--almost-all          do not list implied . and ..
 -a,--all                 do not hide entries starting with .
 -B,--ignore-backups      do not list implied entried ending with ~
 -b,--escape              print octal escapes for nongraphic 																		characters
		--block-size <SIZE>   use SIZE-byte blocks
 -c                       with -lt: sort by, and show, ctime (time of    
 													last modification of file status  	
 													information) with -l:show ctime and sort by 													name otherwise: sort by ctime
 -C                       list entries by columns
```

引入依赖:

```

```



## 二、简介

命令行处理分为三个阶段。它们是***定义，解析和询问***阶段。以下各节将依次讨论每个阶段，并讨论如何使用 CLI 来实现。

### 2.1 定义

每个命令行必须定义一组选项，这些选项将用于定义应用程序的接口。 

CLI使用 `Options` 类作为 Option 实例的容器。在 CLI中 有两种创建 `Option` 的方法

- 通过 Option 构造函数: `new Option(...)`
- 通过 Option 的 builder 构造器: `Option.buider().build()`

cli-1.4 中已经废弃了 `OptionBuilder`, 它推荐使用 `Option.builder().build()` 这种方式来构建 Argument Option

#### 2.1.1 Option 分类

- Boolean Option

  这类 Option 通常没有值, 例如常见的 java -version, java -h, 这些参数通常不带值。

  这类 Option 推荐使用构造函数来定义。

  ```java
  public Option(String opt, String description);
  public Option(String opt, boolean hasArg, String description);
  /**
  	@param opt: 短参数, 如 h
  	@param longOpt: 长参数, 如 help
  	@param hasArg: 是否有值
  	@param description: 参数描述
   */
  public Option(String opt, String longOpt, boolean hasArg, String description);
  ```

- Argument Option

  这类 Option 通常有值, 例如 flink run -m yarn-cluster, 其中 yarn-cluster 就是 参数 m 的值。

  这类 Option 推荐使用 `builder` 构造器来构造。

  ```java
  // 它表示的 Option: -f,--file <filePath1,filePath2,...> 文件的路径
  Option.builder("f")         // 短key
  		.longOpt("file")       	// 长key
      .hasArg(true)           // 是否含有参数
      .argName("filePath1,filePath2,...")    // 参数值的名称
      .required(true)         // 命令行中必须包含此 option
      .desc("文件的路径")       // 描述
      .optionalArg(false)     // 参数的值是否可选
      .numberOfArgs(3)        // 指明参数有多少个参数值
    //.hasArgs()              // 无限制参数值个数
      .valueSeparator(',')    // 参数值的分隔符
      .type(String.class)     //参数值的类型
      .build());
  ```

因为定义阶段的结果是一个`Options`实例。所以最后要将定义好的 Option 加入 Options 容器中。

```java
private static Options getClientOptions(Options options) {
  options.addOption(HELP);
  options.addOption(PROJECT_HELP);
  options.addOption(VERSION);
  options.addOption(QUIET);
  options.addOption(VERBOSE);
  options.addOption(DEBUG);
  options.addOption(EMACS);
  options.addOption(LOG_FILE);
  options.addOption(LOGGER);
  options.addOption(LISTENER);
  options.addOption(BUILD_FILE);
  options.addOption(FIND);
  options.addOption(PROPERTY);
  return options;
}
```

### 2.2 解析

在解析阶段，将处理通过命令行传递到应用程序中的文本。根据解析器实现定义的规则处理文本。

在 `CommandLineParser` 上定义的 `parse(Options, String[])` 方法接受一个`Options`实例和一个`String[]`参数，并返回 `CommandLine`。

解析阶段的结果是一个`CommandLine`实例。

```java
public static AntCliOptions parseClient(String[] args) {
  if (args.length < 1) {
    printHelp();
    throw new RuntimeException("invalid args, use ant -h/--help");
  }
  AntCliOptions instance;
  try {
    CommandLine line = PARSER.parse(CLIENT_OPTIONS, args);
    instance =  new AntCliOptions(
      line.getOptionValue(AntCliOptionsParser.LOG_FILE.getOpt(), "default.log"),
      line.getOptionValue(AntCliOptionsParser.LOGGER.getOpt(), "org.slf4j.Logger"),
      line.getOptionValue(AntCliOptionsParser.LISTENER.getOpt(), "org.spring.EventListener"));
  } catch (ParseException e) {
    throw new RuntimeException(e.getMessage());
  }
  return instance;
}


```

### 2.3 询问

审讯阶段是在应用程序查询的 `CommandLine` 决定如何执行分支取决于布尔选项采用和使用选项值提供应用程序的数据。

此阶段在用户代码中实现。`CommandLine` 上的访问器方法为用户代码提供了查询功能。

询问阶段的结果是，将在命令行上提供并根据解析器和`选项`规则进行处理的所有文本完全通知用户代码。

## 三、完整示例

```java
package cgs.commons.cli;

import org.apache.commons.cli.CommandLine;
import org.apache.commons.cli.DefaultParser;
import org.apache.commons.cli.HelpFormatter;
import org.apache.commons.cli.Option;
import org.apache.commons.cli.Options;
import org.apache.commons.cli.ParseException;

/**
调用 AntCliOptionsParser.printHelp() 打印 help info 如下:

usage: ant [options] [target [target2 [target3] ...]]
 Options:
 -help                        print this message
 -projecthelp                 print project help information
 -version                     print the version information and exit
 -quiet                       be extra quiet
 -verbose                     be extra verbose
 -debug                       print debugging information
 -emacs                       produce logging information without 																adornments
 -logfile <file>              use given file for log
 -logger <classname>      		the class which is to perform logging
 -listener <classname>   			add an instance of class as a project 															listener
 -buildfile <file>            use given buildfile
 -D<property>=<value>    			use value for given property
 -find <file>                 search for buildfile towards the root 															of the filesystem and use it
 */

public class AntCliOptionsParser {

  private static final DefaultParser PARSER = new DefaultParser();
  private static final HelpFormatter HELP_FORMATTER = new HelpFormatter();

  private static final String FORMAT_HEADER = "\u0020Options";
  private static final String FORMAT_SYNTAX = "ant [options] [target [target2 [target3] ...]]";

  // Boolean Options, e.g. -version, -h
  private static final Option HELP = new Option("h", "help", false, "print this message");
  private static final Option PROJECT_HELP = new Option("projecthelp", "print project help information" );
  private static final Option VERSION = new Option( "version", "print the version information and exit" );
  private static final Option QUIET = new Option( "quiet", "be extra quiet" );
  private static final Option VERBOSE = new Option( "verbose", "be extra verbose" );
  private static final Option DEBUG = new Option( "debug", "print debugging information" );
  private static final Option EMACS = new Option( "emacs", "produce logging information without adornments" );

  // Argument Options, e.g. run -m yarn-cluster -p 2
  private static final Option LOG_FILE = Option.builder("logfile")
    .hasArg()
    .argName("file")
    .desc("use given file for log")
    .build();
  private static final Option LOGGER = Option.builder("logger")
    .hasArg()
    .argName("classname")
    .desc("the class which is to perform logging")
    .build();
  private static final Option LISTENER = Option.builder("listener")
    .hasArg()
    .argName("classname")
    .desc("add an instance of class as a project listener")
    .build();
  private static final Option BUILD_FILE = Option.builder("buildfile")
    .hasArg()
    .argName("file")
    .desc("use given buildfile")
    .build();
  private static final Option FIND = Option.builder("find")
    .hasArg()
    .argName("file")
    .desc("search for buildfile towards the root of the filesystem and use it")
    .build();

  // Java Property Option, e.g. -D<property>=<value>
  private static final Option PROPERTY = Option.builder("D")
    .hasArgs()
    .numberOfArgs(2).argName("property=value")
    .valueSeparator('=')
    .desc("use value for given property")
    .build();

  private static final Options CLIENT_OPTIONS = getClientOptions(new Options());

  private static Options getClientOptions(Options options) {
    options.addOption(HELP);
    options.addOption(PROJECT_HELP);
    options.addOption(VERSION);
    options.addOption(QUIET);
    options.addOption(VERBOSE);
    options.addOption(DEBUG);
    options.addOption(EMACS);
    options.addOption(LOG_FILE);
    options.addOption(LOGGER);
    options.addOption(LISTENER);
    options.addOption(BUILD_FILE);
    options.addOption(FIND);
    options.addOption(PROPERTY);
    return options;
  }

  // ----------------------------------
  //  Line Parsing
  // ----------------------------------
	
  public static AntCliOptions parseClient(String[] args) {
    if (args.length < 1) {
      printHelp();
      throw new RuntimeException("invalid args, use ant -h/--help");
    }
    AntCliOptions instance;
    try {
      CommandLine line = PARSER.parse(CLIENT_OPTIONS, args);
      instance = new AntCliOptions(
        line.getOptionValue(
          AntCliOptionsParser.LOG_FILE.getOpt(), "default.log"),
        line.getOptionValue(
          AntCliOptionsParser.LOGGER.getOpt(), "org.slf4j.Logger"),
        line.getOptionValue(
          AntCliOptionsParser.LISTENER.getOpt(), "org.spring.EventListener"));
    } catch (ParseException e) {
      throw new RuntimeException(e.getMessage());
    }
    return instance;
  }

  public static void printHelp() {

    HELP_FORMATTER.printHelp(
      FORMAT_SYNTAX,
      FORMAT_HEADER,
      CLIENT_OPTIONS,
      null
    );
  }
}
```

