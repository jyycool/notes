# (Configuration)配置文件的语法和特性

Play应用程序的配置文件必须在中定义`conf/application.conf`。它使用 [HOCON格式](https://github.com/lightbend/config/blob/master/HOCON.md)。

除`application.conf`文件外，配置还来自其他两个地方。

- 默认设置是加载的从类路径上找到的`reference.conf`文件。大多数Play JAR 都包含`reference.conf`具有默认的配置。而`application.conf`中的相同配置将覆盖`reference.conf`文件中的设置。
- 也可以使用系统属性来设置配置。系统属性会覆盖`application.conf`中的设置。

使用Config的惯用方式是在`reference.conf`或`application.conf`中定义所有配置。如果配置中的 key 没有合理的默认值，则通常将其设置`null`为表示“无值”。



## 指定备选的配置文件

在程序运行时, 默认是会加载在类路径上的 `application.conf`, 但以下的系统变量可以强制加载一个不同的配置源.

- `config.resource` : 指定类路径上的其他源文件, 类似于 `xxx.conf`

- `config.file` : 指定文件系统中源文件路径, 类似于 `path/to/xxx.conf`

以上的系统配置会指定源文件来代替 `application.conf` , 但如果仍需要使用  `application.conf` 中的一些配置值, 你可以在你的 `xxx.conf` 配置文件顶部通过 `include "application.conf"` 中将  `application.conf`  包括进来, 之后你可以在你的 `xxx.conf` 中覆盖任何配置值。



## 配置 Controller

多亏了依赖注入`@Inject()` , 因此可以在控制器中使用默认的或者自定义的配置

```scala
import javax.inject._
import play.api.Configuration

class MyController @Inject()(config: Configuration) {
  //...
}
```

##  

## 配置 Akka

Akka使用与Play应用程序定义的配置文件相同的配置文件。这意味着您可以在`application.conf`文件中配置Akka的任何内容。在Play中，Akka从`play.akka`中设置而不是读取其`akka`配置。



## 在 run 命令中使用配置

使用`run`命令运行应用程序时，需要了解一些有关配置的特殊信息。

### 额外的配置 `devSettings`

你可以在 `build.sbt`中为 `run` 命令设置额外的配置.

当你部署应用时, 这些配置不会被使用.

```scala
PlayKeys.devSettings += "play.server.http.port" -> "8080"
```

### `application.conf`中配置 HTTP server

在 `run` 模式下，Play 的 HTTP server 在编译应用程序之前启动。这意味着HTTP server 启动时无法访问配置文件`application.conf`。如果要在使用 `run` 启动应用且覆盖 HTTP server 设置，则不能使用该`application.conf` 文件。相反，您需要使用 系统属性 或 `devSettings` 上面显示的设置。以下示例时启动时配置 HTTP server port：

```sh
> run -Dhttp.port=1234
```

其他 server 设置可以在[此处](https://www.playframework.com/documentation/2.8.x/ProductionConfiguration#Server-configuration-options)看到。如您在这些服务器设置中所见，http(s) port(s) 和 address 将回退到 config `PLAY_HTTP_PORT`，`PLAY_HTTPS_PORT`和 `PLAY_HTTP_ADDRESS` , 如果尚未定义端口或地址（例如通过）`PlayKeys.devSettings`。因为这些配置键是替换键，所以您也可以通过环境变量来定义它们，例如，在Linux中使用Bash时：

```
export PLAY_HTTP_PORT=9001
export PLAY_HTTPS_PORT=9002
export PLAY_HTTP_ADDRESS=127.0.0.1
```

如果您需要针对开发模式（与 `run` 命令行一起使用的模式）定制Akka配置，则还有一个特定的*命名空间*`run`。您需要在 `PlayKeys.devSettings`加上前缀 `play.akka.dev-mode`，例如：

```sbt
PlayKeys.devSettings += "play.akka.dev-mode.akka.cluster.log-info" -> "off"
```

如果使用的Akka ActorSystem运行开发模式与应用程序本身使用的ActorSystem之间存在某些冲突，则此功能特别有用。



## HOCON 语法

HOCON与JSON有相似之处；您当然可以在[http://json.org/](https://translate.googleusercontent.com/translate_c?depth=1&hl=zh-CN&pto=aue&rurl=translate.google.com&sl=auto&sp=nmt4&tl=zh-CN&u=https://json.org/&usg=ALkJrhhRmsLr3g4EZeSdhvJ4bTL6tlTwbQ)上找到JSON规范。

### 和 Json相同的特性

- 文件必须是有效的 UF-8

- 带引号的字符串与JSON字符串的格式相同

- 值具有可能的类型：字符串，数字，对象，数组，布尔值，空

- 允许的数字格式与JSON匹配；与JSON中一样，未表示某些可能的浮点值，例如`NaN`



### 注释

`// `或 `#`与下一个换行符之间的任何内容均被视为注释并被忽略，除非 `// `或 `#` 包含在带引号的字符串中。



### 省略根括号

JSON文档的根必须有一个数组或对象。空文件和所有仅包含非数组非对象值（例如字符串）的文件一样，都是无效文档。

在HOCON中，如果文件不是以方括号或大括号开头，则将其解析为好像用`{}`大括号括起来一样。

如果HOCON文件省略了开头 `{` 但仍然有结束符则无效 `}` ; 花括号必须平衡。

### 键值分隔符

`=` 可应用在 JSON 中任何允许字符使用 `:` 的地方，即: `HOCON` 中使用 `=` 将键与值分开。

如果键后跟`{`，则可以省略`:`或`=`。所以 `"foo" {}` 意味着 `"foo" : {}"`

### 逗号

数组中的值和对象中的字段之间不必有逗号，只要它们之间至少有一个ASCII换行符（`\n`，十进制值10）即可。

数组中的最后一个元素或对象中的最后一个字段后可以跟一个逗号。多余的逗号将被忽略。

- `[1,2,3,]`和`[1,2,3]`是相同的数组。
- `[1\n2\n3]`和`[1,2,3]`是相同的数组。
- `[1,2,3,,]` 无效，因为它有两个尾随逗号。
- `[,1,2,3]` 无效，因为它有一个初始逗号。
- `[1,,2,3]` 无效，因为它连续有两个逗号。
- 这些相同的逗号规则适用于对象中的字段。

### 重复键

JSON 规范未阐明应如何处理同一对象中的重复键。在HOCON中，除非两个值都是对象，否则稍后出现的重复键将覆盖先前出现的重复键。如果两个值都是对象，则将对象合并。

注意：如果您假设JSON需要重复的键才能具有行为，那么这将使HOCON成为JSON的非超集。这里的假设是重复的键是无效的JSON。

合并对象：

- 将仅在两个对象之一中存在的字段添加到合并的对象中。
- 对于两个对象中都存在的非对象值字段，必须使用在第二个对象中找到的字段。
- 对于两个对象中都存在的对象值字段，应根据这些相同规则递归合并对象值。

可以通过先将键设置为另一个值来防止对象合并。这是因为合并总是一次完成两个值。如果将键设置为对象，非对象，然后是对象，则首先非对象退回到该对象（非对象总是获胜），然后该对象退回到该非对象（不合并） ，则object是新值）。因此，这两个对象永远不会看到对方。

这两个等效：

```json
{
	"foo" : { "a" : 42 },
	"foo" : { "b" : 43 }
}
//===========================
{
	"foo" : { "a" : 42, "b" : 43 }
}
```

这两个是等效的：

```json
{
	"foo" : { "a" : 42 },
  "foo" : null,
  "foo" : { "b" : 43 }
}
//============================
{
  "foo" : { "b" : 43 }
}
```

中间设置`"foo"`到`null`防止对象合并。



### 路径作为键

如果键是具有多个元素的路径表达式，则将其扩展以为除最后一个路径元素之外的每个路径元素创建一个对象。最后一个路径元素及其值将成为最嵌套对象中的一个字段。

换一种说法：

```
foo.bar : 42
```

等效于：

```
foo { bar : 42 }
```

和：

```
foo.bar.baz : 42
```

等效于：

```
foo { bar { baz : 42 } }
```

这些值以通常的方式合并。这意味着：

```
a.x : 42, a.y : 43
```

等效于：

```
a { x : 42, y : 43 }
```

因为路径表达式的工作方式类似于值串联，所以键中可以包含空格：

```
a b c : 42
```

等效于：

```
"a b c" : 42
```

因为路径表达式始终会转换为字符串，所以即使通常具有其他类型的单个值也将变为字符串。

- `true : 42` 是 `"true" : 42`
- `3.14 : 42` 是 `"3.14" : 42`

作为特殊规则，未加引号的字符串 `include` 可能不会在键中以路径表达式开头，因为它具有特殊的解释（请参见下文）。



### 替换

替代是引用配置树其他部分的一种方式。

如上所述，语法为`${pathexpression}`或`${?pathexpression}`其中，`pathexpression`是路径表达式。此路径表达式具有与对象键相同的语法。

`?`在`${?pathexpression}`中, 前面一定不能有空白; 这三个字符`${?`必须完全一样，分组在一起

对于在配置树中找不到的替代，实现可以尝试通过查看系统环境变量或其他外部配置源来解决它们。（在后面的部分中将详细介绍环境变量。）

不会在带引号的字符串中解析替换。要获取包含替换的字符串，必须在未加引号的部分中使用值串联和替换：

```
key : ${animal.favorite} is my favorite animal
```

或者，您可以引用非替代部分：

```
key : ${animal.favorite}" is my favorite animal"
```

通过在配置中查找路径来解决替换。该路径从根配置对象开始，即它是“绝对”而不是“相对”。



### include

#### include 语法

一个***包括声明*** 由未加引号的 `include`，并紧随其后一个***单引号字符***。包含语句可以代替对象字段出现。

如果未加引号的字符串`include`出现在期望对象键的路径表达式的开头，则不会将其解释为路径表达式或键。

相反，***下一个值必须是带引号的字符串。带引号的字符串被解释为要包含的文件名或资源名称***。





### 持续时间格式

持续时间支持的单位字符串区分大小写，并且必须小写。确实支持以下字符串：

- `ns`，`nanosecond`，`nanoseconds`
- `us`，`microsecond`，`microseconds`
- `ms`，`millisecond`，`milliseconds`
- `s`，`second`，`seconds`
- `m`，`minute`，`minutes`
- `h`，`hour`，`hours`
- `d`，`day`，`days`



### 期间格式

支持的单位字符串`java.time.Period`区分大小写，并且必须小写。确实支持以下字符串：

- `d`，`day`，`days`
- `w`，`week`，`weeks`
- `m`，`mo`，`month`，`months`
- `y`，`year`，`years`



### 时间金额格式

使用上面的单位字符串可以是a`java.time.Period`或a `java.time.Duration`。这将有利于作为一个持续时间，这意味着`m`手段分钟，所以你应该使用较长的形式（`mo`，`month`，`months`）来指定个月。



### 字节格式的大小

对于单个字节，完全支持以下字符串：

- `B`，`b`，`byte`，`bytes`

对于十次幂，完全支持以下字符串：

- `kB`，`kilobyte`，`kilobytes`
- `MB`，`megabyte`，`megabytes`
- `GB`，`gigabyte`，`gigabytes`
- `TB`，`terabyte`，`terabytes`
- `PB`，`petabyte`，`petabytes`
- `EB`，`exabyte`，`exabytes`
- `ZB`，`zettabyte`，`zettabytes`
- `YB`，`yottabyte`，`yottabytes`

对于2的幂，完全支持以下字符串：

- `K`，`k`，`Ki`，`KiB`，`kibibyte`，`kibibytes`
- `M`，`m`，`Mi`，`MiB`，`mebibyte`，`mebibytes`
- `G`，`g`，`Gi`，`GiB`，`gibibyte`，`gibibytes`
- `T`，`t`，`Ti`，`TiB`，`tebibyte`，`tebibytes`
- `P`，`p`，`Pi`，`PiB`，`pebibyte`，`pebibytes`
- `E`，`e`，`Ei`，`EiB`，`exbibyte`，`exbibytes`
- `Z`，`z`，`Zi`，`ZiB`，`zebibyte`，`zebibytes`
- `Y`，`y`，`Yi`，`YiB`，`yobibyte`，`yobibytes`

