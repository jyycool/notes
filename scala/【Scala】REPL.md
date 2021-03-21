# 【Scala】REPL

Scala 也是基于 JVM 的一款编程语言, 并且它提供了 REPL (ReadEvalPrint Loop)，但是它并不是一门解释性语言，下面我们进入Scala REPL源码中来一探究竟。

进入 Scala 的 REPL 只需在命令行中输入 scala, 它执行了$SCALA_HOME/bin/scala 这个脚本。脚本中最后使用 java 执行了这个类 `scala.tools.nsc.MainGenericRunner`

```sh
execCommand \
  "${JAVACMD:=java}" \
  $JAVA_OPTS \
  "${java_args[@]}" \
  "${classpath_args[@]}" \
  -Dscala.home="$SCALA_HOME" \
  $OVERRIDE_USEJAVACP \
  $WINDOWS_OPT \
	scala.tools.nsc.MainGenericRunner  "$@"
```

so, scala REPL 的启动类(入口)即: `scala.tools.nsc.MainGenericRunner`

这个类代码相对较短, 只有100 多行:

```scala
package scala
package tools.nsc

import io.{ File }
import util.{ ClassPath, ScalaClassLoader }
import GenericRunnerCommand._

object JarRunner extends CommonRunner {

  def runJar(
    settings: GenericRunnerSettings, 
    jarPath: String, 
    arguments: Seq[String]): Either[Throwable, Boolean] = {

    val jar       = new io.Jar(jarPath)
    val mainClass = jar.mainClass getOrElse sys.error("Cannot find main class for jar: " + jarPath)
    val jarURLs   = ClassPath expandManifestPath jarPath
    val urls      = if (jarURLs.isEmpty) File(jarPath).toURL +: settings.classpathURLs else jarURLs

    if (settings.Ylogcp) {
      Console.err.println("Running jar with these URLs as the classpath:")
      urls foreach println
    }

    runAndCatch(urls, mainClass, arguments)
  }
}

/** An object that runs Scala code.  It has three possible
 *  sources for the code to run: pre-compiled code, a script file,
 *  or interactive entry.
 */
class MainGenericRunner {
  
  def errorFn(
    str: String, 
    e: Option[Throwable] = None, 
    isFailure: Boolean = true): Boolean = {

    if (str.nonEmpty) Console.err println str
    e foreach (_.printStackTrace())
    !isFailure
  }

  def process(args: Array[String]): Boolean = {
    val command = new GenericRunnerCommand(args.toList, (x: String) => errorFn(x))
    import command.{ settings, howToRun, thingToRun, shortUsageMsg, shouldStopWithInfo }
    // def so it's not created unless needed
    def sampleCompiler = new Global(settings)   

    def run(): Boolean = {
      def isE   = !settings.execute.isDefault
      def dashe = settings.execute.value

      def isI   = !settings.loadfiles.isDefault
      def dashi = settings.loadfiles.value

      // Deadlocks on startup under -i unless we disable async.
      if (isI)
      settings.Yreplsync.value = true

      def combinedCode  = {
        val files   = if (isI) dashi map (file => File(file).slurp()) else Nil
        val str     = if (isE) List(dashe) else Nil

        files ++ str mkString "\n\n"
      }

      def runTarget(): Either[Throwable, Boolean] = howToRun match {
        case AsObject =>
        ObjectRunner.runAndCatch(settings.classpathURLs, thingToRun, command.arguments)
        case AsScript =>
        ScriptRunner.runScriptAndCatch(settings, thingToRun, command.arguments)
        case AsJar    =>
        JarRunner.runJar(settings, thingToRun, command.arguments)
        case Error =>
        Right(false)
        case _  =>
        // We start the repl when no arguments are given.
        Right(new interpreter.ILoop process settings)
      }

      /** If -e and -i were both given, we want to execute the -e code after the
       *  -i files have been included, so they are read into strings and prepended to
       *  the code given in -e.  The -i option is documented to only make sense
       *  interactively so this is a pretty reasonable assumption.
       *
       *  This all needs a rewrite though.
       */
      if (isE) {
        ScriptRunner.runCommand(settings, combinedCode, thingToRun +: command.arguments)
      }
      else runTarget() match {
        case Left(ex) => errorFn("", Some(ex))  // there must be a useful message of hope to offer here
        case Right(b) => b
      }
    }

    if (!command.ok)
    errorFn(f"%n$shortUsageMsg")
    else if (shouldStopWithInfo)
    errorFn(command getInfoMessage sampleCompiler, isFailure = false)
    else
    run()
  }
}

object MainGenericRunner extends MainGenericRunner {
  def main(args: Array[String]): Unit = if (!process(args)) sys.exit(1)
}
```

这个类中的核心方法:

```scala
val howToRun = targetAndArguments match {
  case Nil      => AsRepl
  case hd :: _  => waysToRun find (_.name == settings.howtorun.value) getOrElse guessHowToRun(hd)
}

def runTarget(): Either[Throwable, Boolean] = howToRun match {
  case AsObject =>
    ObjectRunner.runAndCatch(settings.classpathURLs, thingToRun, command.arguments)
  case AsScript =>
    ScriptRunner.runScriptAndCatch(settings, thingToRun, command.arguments)
  case AsJar    =>
    JarRunner.runJar(settings, thingToRun, command.arguments)
  case Error =>
    Right(false)
  case _  =>
    // We start the repl when no arguments are given.
    Right(new interpreter.ILoop process settings)
}
```

在 scala 命令为无参的情况下会打开 `Right(new interpreter.ILoop process settings)`，这就是Scala REPL的入口了，我们继续进入到 `process` 方法中一探究竟：

```scala
// start an interpreter with the given settings
 def process(settings: Settings): Boolean = savingContextLoader {
   this.settings = settings
   createInterpreter()

   // sets in to some kind of reader depending on environmental cues
   in = in0.fold(chooseReader(settings))(r => SimpleReader(r, out, interactive = true))
   globalFuture = future {
     intp.initializeSynchronous()
     loopPostInit()
     !intp.reporter.hasErrors
   }
   loadFiles(settings)
   printWelcome()

   try loop() match {
     case LineResults.EOF => out print Properties.shellInterruptedString
     case _               =>
   }
   catch AbstractOrMissingHandler()
   finally closeInterpreter()

   true
 }

 /** Create a new interpreter. */
 def createInterpreter() {
   if (addedClasspath != "")
     settings.classpath append addedClasspath

   intp = new ILoopInterpreter
 }

 private def loopPostInit() {
   // Bind intp somewhere out of the regular namespace where
   // we can get at it in generated code.
   intp.quietBind(NamedParam[IMain]("$intp", intp)(tagOfIMain, classTag[IMain]))
   // Auto-run code via some setting.
   ( replProps.replAutorunCode.option
       flatMap (f => io.File(f).safeSlurp())
       foreach (intp quietRun _)
   )
   // classloader and power mode setup
   intp.setContextClassLoader()
   if (isReplPower) {
     replProps.power setValue true
     unleashAndSetPhase()
     asyncMessage(power.banner)
   }
   // SI-7418 Now, and only now, can we enable TAB completion.
   in.postInit()
 }

 @tailrec final def loop(): LineResult = {
   import LineResults._
   readOneLine() match {
     case null => EOF
     case line => if (try processLine(line) catch crashRecovery) loop() else ERR
   }
 }
```

在这里仅对几个我认为比较有意思的方法来介绍下：

1. `process(Settings)` 方法为 Scala REPL的启动方法，它进行了设置 setting、创建解释器、获取输入源、初始化解释器、打印欢迎标语、进入主 `ReadEvalPrint`循环等操作。

2、`createInterpreter()` 方法很简单，创建了一个解释器类 `ILoopInterpreter` 赋值到了`intp`成员变量中，这个`intp`就是Scala REPL的灵魂。

3、`loopPostInit()` 中，Scala 干了一件很有意思的事，它将刚刚创建的解释器类 `intp` 绑定到了解释器中名为`$intp`的变量上，这也就意味着，我们在 Scala REPL 里可以使用 `$intp` 来引用这个解释器类实例

下面是在 scala REPL 中测试使用 $intp

```sh
scala> $intp
res0: scala.tools.nsc.interpreter.IMain = scala.tools.nsc.interpreter.ILoop$ILoopInterpreter@3be4f71

scala> $intp.definedSymbolList
res1: List[$intp.memberHandlers.intp.global.Symbol] = List(value $intp, value res0)

scala> $intp.interpret("val sum = 10 * 5")
sum: Int = 50
res3: scala.tools.nsc.interpreter.IR.Result = Success

scala> sum
res4: Int = 50
```

我们在 shell 或者 cmd 里输入的语句，最终都会调用 `$intp.interpret` 进行编译执行：`loop() -> processLine(line) -> command(line) -> interpretStartingWith(line) -> intp.interpret(code)`。

`interpret()` 方法的定义如下：

```scala
def interpret(line: String): IR.Result = interpret(line, synthetic = false)
def interpretSynthetic(line: String): IR.Result = interpret(line, synthetic = true)
def interpret(line: String, synthetic: Boolean): IR.Result = compile(
  		line, synthetic) match {
   	case Left(result) => result
   	case Right(req)   => new WrappedRequest(req).loadAndRunReq
 }
```

我们可以看到`interpret(String)`方法最终调用了下面重载的两参数`interpret`方法，它一共进行了两个步骤：

1. 将代码解析成`Request`类 
2. 加载执行`Reques`类

下面我们从第一步开始分析。我们进入到`compile(line, synthetic) -> requestFromLine(line, synthetic)`中：

```scala
private def compile(line: String, synthetic: Boolean): Either[IR.Result, Request] = {
  if (global == null) Left(IR.Error)
  // ① 入口
  else requestFromLine(line, synthetic) match {
    case Left(result) => Left(result)
    case Right(req)   =>
    // null indicates a disallowed statement type; otherwise compile and
    // fail if false (implying e.g. a type error)

    //#####下一句中：!req.compile很关键#####
    if (req == null || !req.compile) Left(IR.Error) else Right(req)
  }
}

private[interpreter] def requestFromLine(line: String, synthetic: Boolean): Either[IR.Result, Request] = {
  val content = line
	// ② 解析用户输入
  val trees: List[global.Tree] = parse(content) match {
    case parse.Incomplete(_)     => return Left(IR.Incomplete)
    case parse.Error(_)          => return Left(IR.Error)
    case parse.Success(trees) => trees
  }

  //中间一大段省略
	
  
  Right(buildRequest(line, trees))
}

// ③ 构造 Request
private[interpreter] def buildRequest(line: String, trees: List[Tree]): Request = {
  executingRequest = new Request(line, trees)
  executingRequest
}
```

代码中先调用 `parse(content)` 将表达式字符串解析成 AST，然后使用原始代码和 AST 构建 `Reques` 对象。在 `Request` 对象中它声明了一个成员变量：`val lineRep = new ReadEvalPrint()`，你输入的这行表达式就是由这个类执行的。每次执行会生成一个新的 REPL 对象来承载这一次的代码执行结果。

**在 compile 最后一行 if 判断条件中，`!req.compile` 将 `lazy` 的 `compile` 成员变量触发执行。**

```scala
/** Compile the object file.  Returns whether the compilation succeeded.
 *  If all goes well, the "types" map is computed. */
lazy val compile: Boolean = {
  // error counting is wrong, hence interpreter may overlook failure - so we reset
  reporter.reset()

  // compile the object containing the user's code
  lineRep.compile(ObjectSourceCode(handlers)) && {
    // extract and remember types
    typeOf
    typesOfDefinedTerms

    // Assign symbols to the original trees
    // TODO - just use the new trees.
    defHandlers foreach { dh =>
      val name = dh.member.name
      definedSymbols get name foreach { sym =>
        dh.member setSymbol sym
        repldbg("Set symbol of " + name + " to " + symbolDefString(sym))
      }
    }

    // compile the result-extraction object
    val handls = if (printResults) handlers else Nil
    withoutWarnings(lineRep compile ResultObjectSourceCode(handls))
  }
}
```

下面我们进入`interpret()`方法的第二步：`new WrappedRequest(req).loadAndRunReq`

```scala
def loadAndRunReq = classLoader.asContext {
  val (result, succeeded) = req.loadAndRun

  /** To our displeasure, ConsoleReporter offers only printMessage,
   *  which tacks a newline on the end.  Since that breaks all the
   *  output checking, we have to take one off to balance.
   */
  if (succeeded) {
    if (printResults && result != "")
      printMessage(result stripSuffix "\n")
    else if (isReplDebug) // show quiet-mode activity
      printMessage(result.trim.lines map ("[quiet] " + _) mkString "\n")

    // Book-keeping.  Have to record synthetic requests too,
    // as they may have been issued for information, e.g. :type
    recordRequest(req)
    IR.Success
  }
  else {
    // don't truncate stack traces
    printUntruncatedMessage(result)
    IR.Error
  }
}

/** load and run the code using reflection */
def loadAndRun: (String, Boolean) = {
  try   { ("" + (lineRep call sessionNames.print), true) }
  catch { case ex: Throwable => (lineRep.bindError(ex), false) }
}123456789101112131415161718192021222324252627282930
```

在这段代码中，REPL调用call将当前行的输出打印在控制台上，之后调用`recordRequest(req)`将当前的`Request`类存储起来`prevRequests += req`。在命令行中，我们可以通过以下命令来查看之前的表达式：

```shell
scala> $intp.prevRequestList
res6: List[$intp.Request] =
List(Request(line=val $intp = value, 1 trees), Request(line=val res0 =
$intp, 1 trees), Request(line=val a = 1, 1 trees), Request(line=val res2 =
$intp.definedSymbolList, 1 trees), Request(line=val b = 5, 1 trees), Request(line=val res3 =
$intp.interpret("val b = 5"), 1 trees), Request(line=val res4 =
b, 1 trees), Request(line=val c = 6, 1 trees), Request(line=val res5 =
c, 1 trees))


scala> $intp.lastRequest
res8: $intp.Request =
Request(line=val res6 =
$intp.prevRequestList, 1 trees)1234567891011121314
```

到这里，Scala REPL的简单执行流程就已经介绍完了，更深入的还有`ClassBasedWrapper`与`ObjectBasedWrapper`、AST结构等，就不展开介绍了。有兴趣的同学可以自己阅读源码进行了解。

## Scala REPL参数

Scala REPL的启动东有很多的参数可以设置，我比较常用的有以下两个：

1、`-Dscala.repl.debug=true`可以让Scala REPL将代码的生成编译过程打印出来，可以方便看出Scala REPL的代码生成与引用方式，也能看出`ClassBasedWrapper`与`ObjectBasedWrapper`状态下的差别。

2、`-Yrepl-class-based`：既然说到了`ClassBasedWrapper`与`ObjectBasedWrapper`，那就要用到这个参数了。Scala在2.12之前是只有ObjectBasedWrapper的，由于Spark使用ObjectBasedWrapper在类初始化阶段会出引起一些问题，所以Spark自己实现了ClassBasedWrapper并给Scala库提了case，Scala在2.12后支持了 ClassBasedWrapper 的REPL方式。

## Spark Shell简介

最后再简单说说Spark Shell的实现。Spark Shell的实现是建立在Scala REPL的基础上的，在Scala 2.12之前，由于Scala REPL不支持ClassBasedWrapper的方式，因此Spark官方自己实现了一套ClassBasedWrapper的行表达式封装方式。在Scala 2.12之后，Spark Shell源码逐渐简洁，到了Scala 2.12时，Spark Shell源码就只剩下了一个`SparkILoop.scala`文件。

下面简单介绍下Spark Shell是怎样基于Scala REPL实现的：

```scala
class SparkILoop(in0: Option[BufferedReader], out: JPrintWriter)
    extends ILoop(in0, out) {
  def this(in0: BufferedReader, out: JPrintWriter) = this(Some(in0), out)
  def this() = this(None, new JPrintWriter(Console.out, true))

  val initializationCommands: Seq[String] = Seq(
    """
    @transient val spark = if (org.apache.spark.repl.Main.sparkSession != null) {
        org.apache.spark.repl.Main.sparkSession
      } else {
        org.apache.spark.repl.Main.createSparkSession()
      }
    @transient val sc = {
      val _sc = spark.sparkContext
      if (_sc.getConf.getBoolean("spark.ui.reverseProxy", false)) {
        val proxyUrl = _sc.getConf.get("spark.ui.reverseProxyUrl", null)
        if (proxyUrl != null) {
          println(
            s"Spark Context Web UI is available at ${proxyUrl}/proxy/${_sc.applicationId}")
        } else {
          println(s"Spark Context Web UI is available at Spark Master Public URL")
        }
      } else {
        _sc.uiWebUrl.foreach {
          webUrl => println(s"Spark context Web UI available at ${webUrl}")
        }
      }
      println("Spark context available as 'sc' " +
        s"(master = ${_sc.master}, app id = ${_sc.applicationId}).")
      println("Spark session available as 'spark'.")
      _sc
    }
    """,
    "import org.apache.spark.SparkContext._",
    "import spark.implicits._",
    "import spark.sql",
    "import org.apache.spark.sql.functions._"
  )

  def initializeSpark() {
    intp.beQuietDuring {
      savingReplayStack { // remove the commands from session history.
        initializationCommands.foreach(command)
      }
    }
  }

  /** Print a welcome message */
  override def printWelcome() {
    import org.apache.spark.SPARK_VERSION
    echo("""Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version %s
      /_/
         """.format(SPARK_VERSION))
    val welcomeMsg = "Using Scala %s (%s, Java %s)".format(
      versionString, javaVmName, javaVersion)
    echo(welcomeMsg)
    echo("Type in expressions to have them evaluated.")
    echo("Type :help for more information.")
  }

  /** Available commands */
  override def commands: List[LoopCommand] = standardCommands

  /**
   * We override `createInterpreter` because we need to initialize Spark *before* the REPL
   * sees any files, so that the Spark context is visible in those files. This is a bit of a
   * hack, but there isn't another hook available to us at this point.
   */
  override def createInterpreter(): Unit = {
    super.createInterpreter()
    initializeSpark()
  }

  override def resetCommand(line: String): Unit = {
    super.resetCommand(line)
    initializeSpark()
    echo("Note that after :reset, state of SparkSession and SparkContext is unchanged.")
  }

  override def replay(): Unit = {
    initializeSpark()
    super.replay()
  }

}1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768697071727374757677787980818283848586878889
```

在上面代码中，最明显的就是`printWelcome`方法里那显眼的Spark banner了。除了这个，最重要的就是`initializationCommands`成员变量。我们在使用Spark Shell时，所能用到的`spark`、`sc`以及一些隐式转换，如Spark SQL中的`$`符号，都是这个initializationCommands中代码在Scala REPL中执行的结果。因此在Scala 2.12之后，Spark Shell仅仅为Scala REPL的简单引用，并在其解释器中预加载好其运行环境而已。在Scala 2.12之前的代码会很长，有兴趣的同学可以自己下载`Spark 1.X`的源码进行阅读了解。

> Scala REPL其实就是一个运行时的代码编译加载执行过程，对JVM的方法区是一个考验。其原理在《编译原理》里都有介绍，这里就不深究了。在java里，也有一些运行时的实时代码编译加载执行包，比如Spark SQL所使用的`janino`包。