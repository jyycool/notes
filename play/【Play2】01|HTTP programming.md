### 【Play2】01|HTTP programming

## 一、Actions, Controllers and Results

### 1.1 什么是 Action

Play 应用收到的大部分请求都是由 `Action` 来处理的。

`play.api.mvc.Action` 就是一个处理收到的请求然后产生结果发给客户端的函数（`play.api.mvc.Request => play.api.mvc.Result`）。

```scala
val echo = Action { request =>  Ok("Got request [" + request + "]")}
```

Action 返回一个类型为 `play.api.mvc.Result` 的值，代表了发送到 web 客户端的 HTTP 响应。在这个例子中，`OK` 构造了一个 **200 OK** 的响应，并包含一个 **text/plain** 类型的响应体。

### 1.2 构建一个 Action

`play.api.mvc.Action` 的伴生对象（companion object）提供了一些 helper 方法来构造一个 Action 值。

最简单的一个函数是 `OK`，它接受一个表达式块作为参数并返回一个 `Result`：

```scala
Action {Ok("Hello world")}
```

这是构造 Action 的最简单方法，但在这种方法里，我们并没有使用传进来的请求。实际应用中，我们常常要使用调用这个 Action 的 HTTP 请求。

因此，还有另一种 Action 构造器，它接受一个函数 `Request => Result` 作为输入：

```scala
Action {
  request =>  Ok("Got request [" + request + "]")
}
```

实践中常常把 `request` 标记为 `implicit`，这样一来，其它需要它的 API 能够隐式使用它：

```scala
Action {
  implicit request =>  Ok("Got request [" + request + "]")
}
```

最后一种创建 Action 的方法是指定一个额外的 `BodyParser` 参数：

```scala
Action(parse.json) { 
	implicit request => Ok("Got request [" + request + "]")
}
```

这份手册后面会讲到 Body 解析器（Body Parser）。现在你只需要知道，上面讲到的其它构造 Action 的方法使用的是一个默认的解析器：**任意内容 body 解析器**（Any content body parser）。

### 1.3 控制器（Controller）是 action 生成器

一个 `Controller` 就是一个产生 `Action` 值的单例对象。

定义一个 action 生成器的最简单方式就是定义一个无参方法，让它返回一个 `Action` 值：

```scala
package controllers
import play.api.mvc._
object Application extends Controller {  
  def index = Action {
    Ok("It works!")  
  }
}
```

当然，生成 action 的方法也可以带参数，并且这些参数可以在 `Action` 闭包中访问到：

```scala
def hello(name: String) = Action { 
  Ok("Hello " + name)
}
```

### 1.4 简单结果

到目前为止，我们就只对简单结果感兴趣：HTTP 结果。它包含了一个状态码，一组 HTTP 报头和发送给 web 客户端的 body。

这些结果由 `play.api.mvc.Result` 定义：

```scala
def index = Action {  
  Result(
    header = ResponseHeader(200, Map(CONTENT_TYPE -> "text/plain")),    		body = Enumerator("Hello world!".getBytes())  
  )
}
```

当然，Play 提供了一些 helper 方法来构造常见结果，比如说 `OK`。下面的代码和上面的代码是等效的：

```scala
def index = Action {  
  Ok("Hello world!")
}
```

上面两段代码产生的结果是一样的。

下面是生成不同结果的一些例子：

```scala
val ok = Ok("Hello world!")
val notFound = NotFound
val pageNotFound = NotFound(<h1>Page not found</h1>)
val badRequest = BadRequest(views.html.form(formWithErrors))
val oops = InternalServerError("Oops")
val anyStatus = Status(488)("Strange response type")
```

上面的 helper 方法都可以在 `play.api.mvc.Results` 特性（trait）和伴生对象（companion object）中找到。

### 1.5 重定向也是简单结果

重定向到一个新的 URL 是另一种简单结果。然而这些结果类型并不包含一个响应体。

同样地，有一些 helper 方法可以来创建重定向结果：

```scala
def index = Action{
  Redirect("/user/home")
}
```

默认使用的响应类型是：`303 SEE_OTHER`，当然，如果你有需要，可以自己设定状态码：

```scala
def index = Action {
	Redirect("/user/home", MOVED_PERMANENTLY)
}
```

### 1.6「TODO」 dummy 页面

你可以使用一个定义为 `TODO` 的空的 `Action` 实现，它的结果是一个标准的 `Not implemented yet` 页面：

```scala
def index(name:String) = TODO
```

## 二、HTTP routing

### 2.1 内建 HTTP 路由

**路由是一个负责将传入的 HTTP 请求转换为 Action 的组件。**

一个 HTTP 请求通常被 MVC 框架视为一个事件（event）。这个事件包含了两个主要信息：

- 请求路径(例如：`/clients/1542`，`/photos/list`)，其中包括了查询串(如 ?page=1&max=3 ）
- HTTP 方法(例如：GET，POST 等)

路由定义在了 `conf/routes` 文件里，该文件会被编译。也就是说你可以直接在你的浏览器中看到这样的路由错误：

![Route Error in Browser](https://www.playframework.com/documentation/2.3.x/resources/manual/scalaGuide/main/http/images/routesError.png)

### 2.2 路由文件的语法

Play 的路由使用 `conf/routes` 作为配置文件。这个文件列出了该应用需要的所有路由。

**每个路由都包含了一个 HTTP 方法和一个 URI 模式，两者被关联到一个 Action 生成器。**

让我们来看下路由的定义：

```scala
GET   /clients/:id			controllers.Clients.show(id: Long)
```

每个路由都以 HTTP 方法开头，后面跟着一个 URI 模式。最后是一个方法的调用。

你也可以使用 `#` 字符添加注释：

```scala
# 显示一个 client
GET   /clients/:id			controllers.Clients.show(id: Long)
```

#### 2.2.1 HTTP 方法

HTTP 方法可以是任何 HTTP 支持的有效方法（`GET`，`POST`，`PUT`，`DELETE`，`HEAD`）。

#### 2.2.2 URI 模式

URI 模式定义了路由的请求路径。其中部分的请求路径可以是动态的。

##### 静态路径

例如，为了能够准确地匹配 `GET /clients/all` 请求，你可以这样定义路由：

```scala
GET   /clients/all			controllers.Clients.list()
```

##### 动态匹配

如果你想要在定义的路由中根据 ID 获取一个 client，则需要添加一个动态部分：

```scala
GET   /clients/:id 			controllers.Clients.show(id: Long)
```

> 注意：一个 URI 模式可以有多个动态的部分。

动态部分的默认匹配策略是通过正则表达式 `[^/]+` 定义的，这意味着任何被定义为 `:id` 的动态部分只会匹配一个 URI。

##### 跨 / 字符的动态匹配

如果你想要一个动态部分能够匹配多个由 `/` 分隔开的 URI 路径，你可以用 `*id` 来定义，此时匹配的正则表达式则为 `.+`：

```scala
GET   /file/*name			controllers.Application.download(name)
```

对于像 `GET /files/images/logo.png` 这样的请求，动态部分 `name` 匹配到的是 `images/logo.png`。

##### 使用自定义正则表达式做动态匹配

你也可以为动态部分自定义正则表达式，语法为 `$id<regex>`：

```scala
GET   /items/$id<[0-9]+>     controllers.Items.show(id: Long)
```

#### 2.2.3 调用 Action 生成器方法

路由定义的最后一部分是个方法的调用。这里必须定义一个返回 `play.api.mvc.Action` 的合法方法，一般是一个控制器 action 方法。

如果该方法不接收任何参数，则只需给出完整的方法名：

```scala
GET   / 				controllers.Application.homePage()
```

如果这个 action 方法定义了参数，Play 则会在请求 URI 中查找所有参数。从 URI 路径自身查找，或是在查询串里查找：

```scala
# 从路径中提取参数 pageGET   /:page                 controllers.Application.show(page)
# 从查询串中提取参数 pageGET   /                      controllers.Application.show(page)
```

这是定义在 `controllers.Application` 里的 `show` 方法：

```scala
def show(page: String) = Action {  
  loadContentFromDatabase(page).map { htmlContent =>    			  
    Ok(htmlContent).as("text/html")  
  }.getOrElse(NotFound)
}
```

##### 参数类型

对于类型为 `String` 的参数来说，可以不写参数类型。如果你想要 Play 帮你把传入的参数转换为特定的 Scala 类型，则必须显式声明参数类型：

```
GET   /clients/:id           controllers.Clients.show(id: Long)
```

同样的，`controllers.Clients` 里的 `show` 方法也需要做相应更改：

```scala
def show(id: Long) = Action {  
  Client.findById(id).map { client =>    
    Ok(views.html.Clients.display(client))  
  }.getOrElse(NotFound)
}
```

##### 设定参数固定值

有时候，你会想给参数设置一个固定值：

```scala
# 从路径中提取参数 page，或是为 / 设置固定值
GET   /     		controllers.Application.show(page = "home")
GET   /:page    controllers.Application.show(page)
```

##### 设置参数默认值

你也可以给参数提供一个默认值，当传入的请求中找不到任何相关值时，就使用默认参数：

```scala
# 分页链接，如 /clients?page=3
GET   /clients  controllers.Clients.list(page: Int ?= 1)
```

##### 可选参数

同样的，你可以指定一个可选参数，可选参数可以不出现在请求中：

```scala
# 参数 version 是可选的。如 /api/list-all?version=3.0
GET   /api/list-all  controllers.Api.list(version: Option[String])
```

### 2.3 路由优先级

多个路由可能会匹配到同一个请求。如果出现了类似的冲突情况，第一个定义的路由（以定义顺序为准）会被启用。

### 2.4 反向路由

路由同样可以通过 Scala 的方法调用来生成 URL。这样做能够将所有的 URI 模式定义在一个配置文件中，让你在重构应用时更有把握。

Play 的路由会为路由配置文件里定义的每个控制器，在 `routes` 包中生成一个「反向控制器」，其中包含了同样的 action 方法和签名，不过返回值类型为 `play.api.mvc.Call` 而非 `play.api.mvc.Action`。

`play.api.mvc.Call` 定义了一个 HTTP 调用，并且提供了 HTTP 请求方法和 URI。

例如，如果你创建了这样一个控制器：

```scala
package controllers
import play.api._
import play.api.mvc._
object Application extends Controller {  
  def hello(name: String) = Action {    
    Ok("Hello " + name + "!")  
  }
}
```

并且在 `conf/routes` 中设置了该方法：

```scala
# Hello action
GET   /hello/:name		controllers.Application.hello(name)
```

接着你就可以调用 `controllers.routes.Application` 的 `hello` action 方法，反向得到相应的 URL：

```scala
// 重定向到 /hello/Bobdef 
helloBob = Action {  
  Redirect(routes.Application.hello("Bob"))
}
```

## 三、Manipulating results

### 3.1 改变默认的 Content-Type

Content-Type 能够从你所指定的响应体的 Scala 值自动推断出来。

例如：

```scala
val textResult = Ok("Hello World!")
```

将会自动设置 `Content-Type` 报头为 `text/plain`， 而：

```scala
val xmlResult = Ok(<message>Hello World!</message>)
```

会设置 Content-Type 报头为 `application/xml`。

> 提示：这是由 `play.api.http.ContentTypeOf` 类型类（type class）完成的。

这相当有用，但是有时候你想去改变它。只需要调用 Result 的 `as(newContentType)` 方法来创建一个新的、类似的、具有不同 `Content-Type` 报头的 Result：

```scala
val htmlResult = Ok(<h1>Hello World!</h1>).as("text/html")
```

或者用下面这种更好的方式：

```scala
val htmlResult2 = Ok(<h1>Hello World!</h1>).as(HTML)
```

> 注意：使用 `HTML` 代替 `"text/html"` 的好处是会为你自动处理字符集，这时实际的 Content-Type 报头会被设置为 `text/html; charset=utf-8`。稍后我们就能看到。

### 3.2 处理 HTTP 报头

你还能添加（或更新）结果的任意 HTTP 报头：

```scala
val result = Ok("Hello World!").withHeaders(  
  CACHE_CONTROL -> "max-age=3600",  
  ETAG -> "xx"
)
```

需要注意的是，如果一个 Result 已经有一个 HTTP 报头了，那么新设置的会覆盖前面的。

### 3.3 设置和丢弃 Cookie

Cookie 是一种特殊的 HTTP 报头，但是我们提供了一系列 helper 方法来简化操作。

你可以像下面那样很容易地添加 Cookie 到 HTTP 响应中：

```scala
val result = Ok("Hello world").withCookies(  
  Cookie("theme", "blue")
)
```

或者，要丢弃先前存储在 Web 浏览器中的 Cookie：

```scala
val result2 = result.discardingCookies(DiscardingCookie("theme"))
```

你也可以在同一个响应中同时添加和丢弃 Cookie：

```scala
val result3 = result
									.withCookies(Cookie("theme","blue"))
									.discardingCookies(DiscardingCookie("skin"))
```

### 3.4 改变基于文本的 HTTP 响应的字符集

对于基于文本的 HTTP 响应，正确处理好字符集是很重要的。Play 处理它的方式是采用 `utf-8` 作为默认字符集。

字符集一方面将文本响应转换成相应的字节来通过网络 Socket 进行传输，另一方面用正确的 `;charset=xxx` 扩展来更新 `Content-Type` 报头。

字符集由 `play.api.mvc.Codec` 类型类自动处理。仅需要引入一个隐式的 `play.api.mvc.Codec` 实例到当前作用域中，从而改变各种操作所用到的字符集：

```scala
object Application extends Controller {  
  implicit val myCustomCharset = Codec.javaSupported("iso-8859-1")  
  def index = Action {Ok(<h1>Hello World!</h1>).as(HTML)  }
}
```

这里，因为作用域中存在一个隐式字符集的值，它会被应用到 `Ok(...)` 方法来将 XML 消息转化成 `ISO-8859-1` 编码的字节，同时也用于生成 `text/html;charset=iso-8859-1` Content-Type 报头。

现在，如果你想知道 `HTML` 方法是怎么工作的，以下是它的定义：

```scala
def HTML(implicit codec: Codec) = {  
  "text/html; charset=" + codec.charset
}
```

如果你的 API 中需要以通用的方式来处理字符集，你可以按照上述方法进行操作。

## 四、Session 和 Flash 域

### 4.1 它们在 Play 中有何不同

如果你必须跨多个 HTTP 请求来保存数据，你可以把数据存在 Session 或是 Flash 域中。存储在 Session 中的数据在整个会话期间可用，而存储在 Flash 域中数据只对下一次请求有效。

Session 和 Flash 数据不是由服务器来存储，而是以 Cookie 机制添加到每一次后续的 HTTP 请求。这也就意味着数据的大小是很受限的（最多 4KB），而且你只能存储字符串类型的值。在 Play 中默认的 Cookie 名称是 `PLAY_SESSION`。默认的名称可以在 application.conf 中通过配置 `session.cookieName` 的值来修改。

> 如果 Cookie 的名字改变了，可以使用上一节中「设置和丢弃 Cookie」提到的同样的方法来使之前的 Cookie 失效。

当然了，Cookie 值是由密钥签名了的，这使得客户端不能修改 Cookie 数据（否则它会失效）。

Play 的 Session 不能当成缓存来用。假如你需要缓存与某一会话相关的数据，你可以使用 Play 内置的缓存机制并在用户会话中存储一个唯一的 ID 与之关联。

> 技术上来说，Session 并没有超时控制，它在用户关闭 Web 浏览器后就会过期。如果你的应用需要一种功能性的超时控制，那就在用户 Session 中存储一个时间戳，并在应用需要的时候用它（例如：最大的会话持续时间，最大的非活动时间等）。

### 4.2 存储数据到 Session 中

由于 Session 是个 Cookie 也是个 HTTP 头，所以你可用操作其他 Result 属性那样的方式操作 Session 数据：

```scala
Ok("Welcome!").withSession("connected" -> "user@gmail.com")
```

这会替换掉整个 Session。假如你需要添加元素到已有的 Session 中，方法是先添加元素到传入的 Session 中，接着把它作为新的 Session：

```scala
Ok("Hello World!").withSession(  
  request.session + ("saidHello" -> "yes")
)
```

你可以用同样的方式从传入的 Session 中移除任何值：

```scala
Ok("Theme reset!").withSession(request.session - "theme")
```

### 4.3 从 Session 中读取值

你可以从 HTTP 请求中取回传入的 Session：

```scala
def index = Action { 
  request =>  request.session.get("connected").map { 
    user => Ok("Hello " + user)
  }.getOrElse {
    Unauthorized("Oops, you are not connected")
  }
}
```

### 4.4 丢弃整个 Session

有一个特殊的操作用来丢弃整个 Session：

```scala
Ok("Bye").withNewSession
```

### 4.5 Flash 域

Flash 域工作方式非常像 Session，但有两点不同：

- 数据仅为一次请求而保留。
- Flash 的 Cookie 未被签名，这留给了用户修改它的可能。

> Flash 域只应该用于简单的非 Ajax 应用中传送 success/error 消息。由于数据只保存给下一次请求，而且在复杂的 Web 应用中无法保证请求的顺序，所以在竞争条件（race conditions）下 Flash 可能不那么好用。

下面是一些使用 Flash 域的例子：

```scala
def index = Action { 
  implicit request =>  Ok {   
    request.flash.get("success").getOrElse("Welcome!")  
  }
}
def save = Action {  
  Redirect("/home").flashing("success" -> "The item has been created")
}
```

为了在你的视图中使用 Flash 域，需要加上 Flash 域的隐式转换：

```scala
@()(implicit flash: Flash) ...
@flash.get("success").getOrElse("Welcome!") ...
```

如果出现 `could not find implicit value for parameter flash: play.api.mvc.Flash` 的错误，像下面这样加上 `implicit request=>` 就解决了：

```scala
def index() = Action {  
  implicit request => Ok(views.html.Application.index())
}
```

## 五、请求体解析器（Body parsers）

### 5.1 什么是请求体解析器？

一个 HTTP 的 PUT 或 POST 请求包含一个请求体。请求体可以是请求头中 `Content-Type` 指定的任何格式。在 Play 中，一个请求体解析器将请求体转换为对应的 Scala 值。

然而，HTTP 请求体可能非常的大，**请求体解析器**不能等到所有的数据都加载进内存再去解析它们。`BodyParser[A]`实际上是一个 `Iteratee[Array[Byte]，A]`，也就是说它一块一块的接收数据（只要 Web 浏览器在上传数据）并计算出类型为 `A` 的值作为结果。

让我们来看几个例子。

- 一个 **text** 类型的请求体解析器能够把逐块的字节数据连接成一个字符串，并把计算得到的字符串作为结果（`Iteratee[Array[Byte]，String]`）。
- 一个 **file** 类型的请求体解析器能够把逐块的字节数据存到一个本地文件，并以 `java.io.File` 引用作为结果（`Iteratee[Array[Byte]，File]`）。
- 一个 **s3** 类型的请求体解析器能够把逐块的字节数据推送给 Amazon S3 并以 S3 对象 ID 作为结果 (`Iteratee[Array[Byte]，S3ObjectId]`)。

此外，**请求体解析器**在开始解析数据之前已经访问了 HTTP 请求头，因此有机会可以做一些预先检查。例如，请求体解析器能检查一些 HTTP 头是否正确设置了，或者检查用户是否有权限上传一个大文件。

> 注意：这就是为什么请求体解析器不是一个真正的`Iteratee[Array[Byte]，A]`，确切的说是一个`Iteratee[Array[Byte]，Either[Result，A]]`，也就是说它在无法为请求体计算出正确的值时，可以直接发送 HTTP 结果（典型的像 `400 BAD_REQUEST`、`412 PRECONDITION_FAILED` 或者 `413 REQUEST_ENTITY_TOO_LARGE`)。

一旦请求体解析器完成了它的工作且返回了类型为 `A` 的值时，相应的 `Action` 函数就会被执行，此时计算出来的请求体的值也已被传入到请求中。

### 5.2 更多关于 Action 的内容

前面我们说 `Action` 是一个 `Request => Result` 函数，这不完全对。让我们更深入地看下 `Action` 这个特性（trait）：

```scala
trait Action[A] extends (Request[A] => Result) {  
  def parser: BodyParser[A]
}
```

首先我们看到有一个泛型类 A，然后一个 Action 必须定义一个 `BodyParser[A]`。`Request[A]` 的定义如下：

```scala
trait Request[+A] extends RequestHeader {
  def body: A
}
```

`A` 是请求体的类型，我们可以使用任何 Scala 类型作为请求体，例如 `String`，`NodeSeq`，`Array[Byte]`，`JsonValue`，或是 `java.io.File`，只要我们有一个请求体解析器能够处理它。

总的来说，`Action[A]` 用返回类型为 `BodyParser[A]` 的方法去从 HTTP 请求中获取类型为 `A` 的值，并构建出 `Request[A]` 类型的对象传递给 Action 代码。

### 5.3 默认的请求体解析器：AnyContent

在我们前面的例子中还从未指定过请求体解析器，那它是怎么工作的呢？如果你不指定自己的请求体解析器，Play 就会使用默认的，它把请求体处理成一个 `play.api.mvc.AnyContent` 实例。

默认的请求体解析通过查看 `Content-Type` 报头来决定要处理的请求体类型：

- **text/plain**：`String`
- **application/json**：`JsValue`
- **application/xml**，**text/xml** 或者 **application/XXX+xml**：`NodeSeq`
- **application/form-url-encoded**：`Map[String，Seq[String]]`
- **multipart/form-data**：`MultipartFormData[TemporaryFile]`
- 任何其他的类型：`RawBuffer`

例如：

```scala
def save = Action { request =>  
  val body: AnyContent = request.body  
  val textBody: Option[String] = body.asText  
  // Expecting text body  
  textBody.map { text =>    
    Ok("Got: " + text)  
  }.getOrElse {    
    BadRequest("Expecting text/plain request body")  
  }
}
```

### 5.4 指定一个请求体解析器

Play 中可用的请求体解析器定义在 `play.api.mvc.BodyParsers.parse` 中。

例如，定义了一个处理 text 类型请求体的 Action（像前面示例中那样）：

```scala
def save = Action(parse.text) { 
  request =>  Ok("Got: " + request.body)
}
```

你知道代码是如何变简单的吗？这是因为如果发生了错误，`parse.text` 这个请求体解析器会发送一个 `400 BAD_REQUEST` 的响应。我们在 Action 代码中没有必要再去做检查。我们可以放心地认为 `request.body` 中是合法的 `String`。

或者，我们也可以这么用：

```scala
def save = Action(parse.tolerantText) { 
  request =>  Ok("Got: " + request.body)
}
```

这个方法不会检查 `Content-Type` 报头，而总是把请求体加载为字符串。

> 小贴士: 在 Play 中所有的请求体解析都提供有 `tolerant` 样式的方法。

这是另一个例子，它将把请求体存为一个文件：

```
def save = Action(parse.file(to = new File("/tmp/upload"))) { request =>  Ok("Saved the request content to " + request.body)}
```

### 5.5 组合请求体解析器

在前面的例子中，所有的请求体都会存到同一个文件中。这会产生难以预料的问题，不是吗？我们来写一个定制的请求体解析器，它会从请求会话中得到用户名，并为每个用户生成不同的文件：

```scala
val storeInUserFile = parse.using { request =>  	
  request.session.get("username").map { user =>    
    file(to = new File("/tmp/" + user + ".upload"))  
  }.getOrElse {    
    sys.error("You don't have the right to upload here")  
  }
}
def save = Action(storeInUserFile) { 
  request =>  Ok("Saved the request content to " + request.body)
}
```

> 注意：这里我们并没有真正的写自己的请求体解析器，只不过是组合了现有的。这样做通常足够了，能应付多数情况。从头写一个`请求体解析器`会在高级主题部分涉及到。

### 5.6 最大内容长度

基于文本的请求体解析器（如`text`，`json`，`xml` 或 `formUrlEncoded`）会有一个最大内容长度，因为它们要加载所有内容到内存中。

默认的最大内容长度是 100KB，但是你也可以在代码中指定它：

```scala
// Accept only 10KB of data.
def save = Action(parse.text(maxLength = 1024 * 10)) { 
  request =>  Ok("Got: " + text)
}
```

> 小贴士：默认的内容大小可在 `application.conf` 中定义：
> `parsers.text.maxLength=128K`

你可以在任何请求体解析器中使用 `maxLength`：

```scala
// Accept only 10KB of data.
def save = Action(parse.maxLength(1024 * 10, storeInUserFile)) { 
  request =>  Ok("Saved the request content to " + request.body)
}
```

## 六、Actions composition

这一章引入了几种定义通用 action 的方法

### 6.1 自定义action构造器

[之前](https://www.bookstack.cn/read/play-for-scala-developers/$ScalaActions.md)，我们已经介绍了几种声明一个action的方法 - 带有请求参数，无请求参数和带有body解析器（body parser）等。事实上还有其他一些方法，我们会在[异步编程](https://www.bookstack.cn/read/play-for-scala-developers/ch02-ScalaAsync.md)中介绍。

这些构造 action 的方法实际上都是有由一个命名为 `ActionBuilder` 的特性（trait）所定义的，而我们用来声明所有 action 的 `Action` 对象只不过是这个特性（trait）的一个实例。通过实现自己的`ActionBuilder`，你可以声明一些可重用的 action 栈，并以此来构建 action。

让我们先来看一个简单的日志装饰器例子。在这个例子中，我们会记录每一次对该 action 的调用。

第一种方式是在 `invokeBlock` 方法中实现该功能，每个由 `ActionBuilder` 构建的 action 都会调用该方法：

```scala
import play.api.mvc._
object LoggingAction extends ActionBuilder[Request] {  
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {    
    Logger.info("Calling action")    
    block(request)  
  }
}
```

现在我们就可以像使用`Action`一样来使用它了：

```scala
def index = LoggingAction {  
  Ok("Hello World")
}
```

`ActionBuilder`提供了其他几种构建action的方式，该方法同样适用于如声明一个自定义body解析器（body parser）等方法：

```scala
def submit = LoggingAction(parse.text) { request =>  
  Ok("Got a bory " + request.body.length + " bytes long")
}
```

### 6.2 组合action

在大多数的应用中，我们会有多个action构造器，有些用来做各种类型的验证，有些则提供了多种通用功能等。这种情况下，我们不想为每个类型的action构造器都重写日志action，这时就需要定义一种可重用的方式。

可重用的action代码可以通过嵌套action来实现：

```scala
import play.api.mvc._
case class Logging[A](action: Action[A]) extends Action[A] {  
  def apply(request: Request[A]): Future[Result] = {   
    Logger.info("Calling action")    
    action(request)  
  }  
  lazy val parser = action.parser
}
```

我们也可以使用`Action`的action构造器来构建，这样就不需要定义我们自己的action类了：

```scala
import play.api.mvc._
def logging[A](action: Action[A]) = Action.async(action.parser) { 	
  request =>  Logger.info("Calling action")  
  action(request)
}
```

Action同样可以使用`composeAction`方法混入（mix in）到action构造器中：

```scala
object LoggingAction extends ActionBuilder[Request] {  
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {    
    block(request)  
  }  
  override def composeAction[A](action: Action[A]) = new Logging(action)
}
```

现在构造器就能像之前那样使用了：

```scala
def index = LoggingAction { 
  Ok("Hello World")
}
```

我们也可以不用action构造器来混入（mix in）嵌套action：

```scala
def index = Logging {  
  Action {Ok("Hello World")}
}
```

### 6.3 更多复杂的action

到现在为止，我们所演示的action都不会影响传入的请求。我们当然也可以读取并修改传入的请求对象：

```scala
import play.api.mvc._
def xForwardedFor[A](action: Action[A]) = Action.async(action.parser) { 
  request =>  
  val newRequest = request.headers.get("X-Forwarded-For").map { 
    xff => new WrappedRequest[A](request) {      
      override def remoteAddress = xff    
    }  
  } getOrElse request action(newRequest)
}

```

> 注意： Play已经内置了对X-Forwarded-For头的支持

我们可以阻塞一个请求：

```scala
import play.api.mvc._
def onlyHttps[A](action: Action[A]) = Action.async(action.parser) { 
  request => request.headers.get("X-Forwarded-Proto").collect {    
    case "https" => action(request)  
  } getOrElse {    
    Future.successful(Forbidden("Only HTTPS requests allowed"))  
  }
}
```

最后，我们还可以修改返回的结果：

```scala
import play.api.mvc._
import play.api.libs.concurrent.Execution.Implicits._
def addUaHeader[A](action: Action[A]) = Action.async(action.parser) { 
  request =>  action(request).map(_.withHeaders(
    "X-UA-Compatible" -> "Chrome=1"))
}
```

### 6.4 不同的请求类型

当组合action允许在HTTP请求和响应的层面进行一些额外的操作时，你自然而然的就会想到构建数据转换的管道（pipeline），为请求本身增加上下文（context）或是执行一些验证。你可以把`ActionFunction`当做是一个应用在请求上的方法，该方法参数化了传入的请求类型和输出类型，并将其传至下一层。每个action方法可以是一个模块化的处理，如验证，数据库查询，权限检查，或是其他你想要在action中组合并重用的操作。

Play还有一些预定义的特性（trait），它们实现了`ActionFunction`，并且对不同类型的操作都非常有用：

- `ActionTransformer`可以更改请求，比如添加一些额外的信息。
- `ActionFilter`可选择性的拦截请求，比如在不改变请求的情况下处理错误。
- `ActionRefiner`是以上两种的通用情况
- `ActionBuilder`是一种特殊情况，它接受`Request`作为参数，所以可以用来构建action。

你可以通过实现`invokeBlock`方法来定义你自己的`ActionFunction`。通常为了方便，会定义输入和输出类型为`Request`（使用`WrappedRequest`），但这并不是必须的。

#### 验证

Action方法最常见的用例之一就是验证。我们可以简单的实现自己的验证action转换器（transformer），从原始请求中获取用户信息并添加到`UserRequest`中。需要注意的是这同样也是一个`ActionBuilder`，因为其输入是一个`Request`：

```scala
import play.api.mvc._
class UserRequest[A](val username: Option[String], request: Request[A]) extends WrappedRequest[A](request)
object UserAction extends ActionBuilder[UserRequest] with ActionTransformer[Request, UserRequest] {  
  def transform[A](request: Request[A]) = Future.successful {    
    new UserRequest(request.session.get("username"), request)  
  }
}
```

Play提供了内置的验证action构造器。更多信息请参考[这里](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.mvc.Security$$AuthenticatedBuilder$)

> 注意：内置的验证 action 构造器只是一个简便的 helper，目的是为了用尽可能少的代码为一些简单的用例添加验证功能，其实现和上面的例子非常相似。如果你有更复杂的需求，推荐实现你自己的验证 action

#### 为请求添加信息

现在让我们设想一个REST API，处理类型为`Item`的对象。在`/item/:itemId`的路径下可能有多个路由，并且每个都需要查询该`item`。这种情况下，将逻辑写在action方法中非常有用。

首先，我们需要创建一个请求对象，将`Item`添加到`UserRequest`中：

```scala
import play.api.mvc._
class ItemRequest[A](val item: Item, request: UserRequest[A]) extends WrappedRequest[A](request) {  
  def username = request.username
}
```

现在，创建一个action修改器（refiner）查找该item并返回`Either`一个错误（`Left`）或是一个新的`ItemRequest`（`Right`）。注意这里的action修改器（refiner）定义在了一个方法中，用来获取该item的id：

```scala
def ItemAction(itemId: String) = new ActionRefiner[UserRequest, ItemRequest] {  
  def refine[A](input: UserRequest[A]) = Future.successful {    
    ItemDao.findById(itemId)
    		.map(new ItemRequest(_, input))      
    		.toRight(NotFound)  
  }
}
```

#### 验证请求

最后，我们希望有个action方法能够验证是否继续处理该请求。例如，我们可能需要检查`UserAction`中获取的user是否有权限使用`ItemAction`中得到的item，如果不允许则返回一个错误：

```scala
object PermissionCheckAction extends ActionFilter[ItemRequest] {  
  def filter[A](input: ItemRequest[A]) = Future.successful {    
    if (!input.item.accessibleByUser(input.username))     		 
    	Some(Forbidden)    
    else      
    	None  
  }
}
```

#### 合并起来

现在我们可以将所有这些action方法链起来（从`ActionBuilder`开始）， 使用`andThen`来创建一个action：

```scala
def tagItem(itemId: String, tag: String) = (UserAction andThen ItemAction(itemId) andThen PermissionCheckAction) { 
  request => request.item.addTag(tag)    
  Ok("User " + request.username + " tagged " + request.item.id)  
}
```

Play 同样支持 [全局过滤API](https://www.playframework.com/documentation/2.3.x/ScalaHttpFilters)，对于全局的过滤非常有用。

## 七、Content negotiation

内容协商（Content negotiation）这种机制使得将相同的资源（URI）提供为不同的表示这件事成为可能。这一点非常有用，比如说，在写 Web 服务的时候，支持几种不同的输出格式（XML，Json等）。服务器端驱动的协商是通过使用 `Accept*` 请求报头（header）来做的。你可以在[这里](http://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html)找到更多关于内容协商的信息。

### 7.1 语言

你可以通过 `play.api.mvc.RequestHeader#acceptLanguages` 方法来获取针对一个请求的可接受语言列表，该方法从 `Accept-Language` 报头获取这些语言并根据它们的品质值来排序。Play 在 `play.api.mvc.Controller#lang` 方法中使用它，该方法为你的 action 提供了一个隐式的 `play.api.i18n.Lang` 值，因此它会自动选择最可能的语言（如果你的应用支持的话，否则会使用你应用的默认语言）。

### 7.2 内容

与上面相似，`play.api.mvc.RequestHeader#acceptedTypes` 方法给出针对一个请求的可接受结果的 MIME 类型列表。该方法从 `Accept` 请求报头获取这些 MIME 类型并根据它们的品质因子进行排序。

事实上，`Accept` 报头并不是真的包含 MIME 类型，而是媒体种类（比如一个请求如果接受的是所有文本结果，那媒体种类可设置为 `text/*`。而 `*/*` 表示所有类型的结果都是可接受的。）。控制器（Controller）提供了一个高级方法 `render` 来帮助你处理媒体种类。例如，考虑以下的 action 定义：

```scala
val list = Action { 
  implicit request => val items = Item.findAll render {    
    case Accepts.Html() => Ok(views.html.list(items))    
    case Accepts.Json() => Ok(Json.toJson(items))  
  }
}
```

`Accepts.Html()` 和 `Accepts.Json()` 是两个提取器（extractor），用于测试提供的媒体种类是 `text/html` 还是 `application/json`。`render` 方法接受一个类型为 `play.api.http.MediaRange => play.api.mvc.Result`的部分函数（partial function）作为参数，并按照优先顺序将它应用在 `Accept` 报头中的每个媒体种类，如果所有可接受的媒体种类都不被你的函数支持，那么会返回一个 `NotAcceptable` 结果。

例如，如果一个客户端发出的请求有如下的 `Accept` 报头：`*/*;q=0.5,application/json`，意味着它接受任意类型的结果，但更倾向于要 JSON 类型的，上面那段代码就会给它返回 JSON 类型的结果。如果另一个客户端发出的请求的 `Accept` 报头是 `application/xml`，这意味着它只接受 XML 的结果，上述代码会返回 `NotAcceptable`。

### 7.3 请求提取器（Request extractors）

参见 API 文档中 `play.api.mvc.AcceptExtractors.Accepts` 对象，了解 Play 在 `render` 方法中所支持的 MIME 类型。使用 `play.api.mvc.Accepting` case 类，你可以很容易地为给定的 MIME 类型创建一个你自己的提取器。例如，下面的代码创建了一个提取器，用于检查媒体类型是否配置 `audio/mp3` MIME 类型。

```scala
val AcceptsMp3 = Accepting("audio/mp3") render {  
  case AcceptsMp3() => ???
}
```