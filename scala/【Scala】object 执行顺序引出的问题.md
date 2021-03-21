# 【Scala】object 执行顺序引出的问题

## 1. Bug

今天在写一段代码, 莫名的发生了 NullPointException:

```scala
object EngineCliParser {

  private val LOG = LoggerFactory.getLogger(this.getClass)
  private val PARSER = new DefaultParser
  private val HELP_FORMATTER = new HelpFormatter
  // Make sure that the instantiated EngineCliParser is global singleton
  private val ENGINE_OPTIONS = getEngineCliOptions(new Options())

  private val ZK_SERVERS = Option.builder("zk")
  				.longOpt("zookeeper")
  				.hasArg()
  				.argName("String:hosts")
  				.required()
  				.desc("REQUIRED: The connection string for the zookeeper connection in the form host:port. Multiple hosts can be given to allow fail-over.")
          .build()

  private val ENGINE_TAG  = Option.builder("tag")
  				.longOpt("engine-tag")
  				.hasArg()
  				.argName("String:engine-id")
  				.required()
  				.desc("REQUIRED: The engine identifier must be globally unique, otherwise registering ZooKeeper will fail")
  				.build()
  
  private def getEngineCliOptions(options: Options): Options = {
    options.addOption(ZK_SERVERS)
    options.addOption(ENGINE_TAG)
  }

  //---------------------------------
  //  Line Parsing
  //---------------------------------
  def parseCommandLine(args: Array[String]): mutable.Map[String, String] = {
    require(args.nonEmpty, s"Invalid args:[${args.mkString(", ")}], please check again...")
    val engineInfo: mutable.Map[String, String] = new mutable.HashMap[String, String]()
    val tryiedParsing: Try[CommandLine] = Try(PARSER.parse(ENGINE_OPTIONS, args))
    tryiedParsing match {
      case Success(commandLine) =>
      engineInfo += ("zookeeper" -> commandLine.getOptionValue(INSTANCE.ZK_SERVERS.getOpt, "localhost:2181"))
      engineInfo += ("enginetag" -> commandLine.getOptionValue(INSTANCE.ENGINE_TAG.getOpt, ""))
      case Failure(exception) =>
      LOG.error("occur an exception during parsing command line", exception)
    }
    engineInfo
  }

  def printHelpInfo(): Unit = {
    HELP_FORMATTER.printHelp("engine", ENGINE_OPTIONS)
  }

}
```

测试代码

```scala
test("testParseCommandLine") {

	val args: Array[String] = Array("-zk", "node01:2181, node02:2181", "-tag", "engine112")
	println(EngineCliParser.parseCommandLine(args))
}
```

当执行测试代码的时候, NullPointException 发生在 `EngineCliParser # getEngineCliOptions(Options)` 中, 根据错误堆栈信息可以确定是 `ZK_SERVERS` 和 `ENGINE_TAG` 这两个变量未被初始化, 导致了 `options.addOption(ZK_SERVERS)` 该方法空指针。

这里就需要了解 Scala object 初始化的执行顺序了, 因为 object 后面的 {} 内的代码会顺序执行, `private val ENGINE_OPTIONS = getEngineCliOptions(new Options())` 的执行早于 `ZK_SERVERS` 和 `ENGINE_TAG` 这两个变量的被初始化, 所以这导致了空指针的问题。

## 2. 解决方案

### 2.1 方案一

最简单的方法就是将`private val ENGINE_OPTIONS = getEngineCliOptions(new Options())` 这段代码放到 `ZK_SERVERS` 和 `ENGINE_TAG` 这两个变量初始化之后。

因为在设计中我需要保证 `ZK_SERVERS` 和 `ENGINE_TAG` 这两个变量的全局唯一, 所以一开始的考虑就是将这两个变量放在 object 中, 如果在 Java 中直接用 static 修饰就能很 easy 的解决, 但是 scala......

### 2.2 方案二

作为一个资深强迫症感觉这个方案一不够好。代码的位置不应该影响执行结果, 于是将 `ZK_SERVERS` 和 `ENGINE_TAG` 这两个变量放到伴生类 `EngineCliParser` 中, 因为要保证全局的唯一, 这里要将伴生类的构造方法封闭起来。

```scala
class EngineCliParser private {
	
	private val ZK_SERVERS = Option.builder("zk")
			.longOpt("zookeeper")
			.hasArg()
			.argName("String:hosts")
			.required()
			.desc("REQUIRED: The connection string for the zookeeper connection in the form host:port. Multiple hosts can be given to allow fail-over.")
			.build()
	
	private val ENGINE_TAG  = Option.builder("tag")
			.longOpt("engine-tag")
			.hasArg()
			.argName("String:engine-id")
			.required()
			.desc("REQUIRED: The engine identifier must be globally unique, otherwise registering ZooKeeper will fail")
			.build()
	
	private val options = new Options
	
}
```

在伴生对象中我们构造一个全局单例的 INSTANCE:EngineCliParser, 并且将这个单例对象 INSTANCE 的 options 作为 `EngineCliParser # getEngineCliOptions(Options)` 方法的参数去构建 Option 的容器。

```scala
object EngineCliParser {

  private val LOG = LoggerFactory.getLogger(this.getClass)
  private val PARSER = new DefaultParser
  private val HELP_FORMATTER = new HelpFormatter
  // Make sure that the instantiated EngineCliParser is global singleton
  private[cli] val INSTANCE = new EngineCliParser
  private val ENGINE_OPTIONS = getEngineCliOptions(INSTANCE.options)

  private[cli] def getEngineCliOptions(options: Options): Options = {
    options.addOption(INSTANCE.ZK_SERVERS)
    options.addOption(INSTANCE.ENGINE_TAG)
  }

  //---------------------------------
  //  Line Parsing
  //---------------------------------
  def parseCommandLine(args: Array[String]): mutable.Map[String, String] = {
    require(args.nonEmpty, s"Invalid args:[${args.mkString(", ")}], please check again...")
    val engineInfo: mutable.Map[String, String] = new mutable.HashMap[String, String]()
    val tryiedParsing: Try[CommandLine] = Try(PARSER.parse(ENGINE_OPTIONS, args))
    tryiedParsing match {
      case Success(commandLine) =>
      engineInfo += ("zookeeper" -> commandLine.getOptionValue(INSTANCE.ZK_SERVERS.getOpt, "localhost:2181"))
      engineInfo += ("enginetag" -> commandLine.getOptionValue(INSTANCE.ENGINE_TAG.getOpt, ""))
      case Failure(exception) =>
      LOG.error("occur an exception during parsing command line", exception)
    }
    engineInfo
  }

  def printHelpInfo(): Unit = {
    HELP_FORMATTER.printHelp("engine", ENGINE_OPTIONS)
  }

}
```

测试 INSTANCE 是否是全局单例。

```scala
test("test Whether INSTANCE is a global singleton") {
  val e1: EngineCliParser = EngineCliParser.INSTANCE
  val e2: EngineCliParser = EngineCliParser.INSTANCE
  assert(e1 eq e2)
}
```

