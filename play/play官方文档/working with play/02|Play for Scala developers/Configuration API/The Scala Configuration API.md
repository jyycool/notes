# The Scala Configuration API



Play 使用 [Typesafe配置库](hhttps://github.com/lightbend/config)，但Play还提供了一个不错的Scala包装器，该包装器 [`Configuration`](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/Configuration.html) 具有更高级的Scala功能。如果您不熟悉Typesafe config，则可能还需要阅读有关 [配置文件语法和功能](https://www.playframework.com/documentation/2.8.x/ConfigFile) 的文档。



## 访问配置

通常，您将通过[Dependency Injection](https://translate.googleusercontent.com/translate_c?depth=1&hl=zh-CN&pto=aue&rurl=translate.google.com&sl=auto&sp=nmt4&tl=zh-CN&u=https://www.playframework.com/documentation/2.8.x/ScalaDependencyInjection&usg=ALkJrhiF73P9j3m55NHr_Sr0pN08BaLNWg)获得  `Configuration`  对象，或者简单地通过将  `Configuration` 实例传递给组件来获取：

```scala
class MyController @Inject() (config: Configuration, c: ControllerComponents) extends AbstractController(c) {
  def getFoo = Action {
    Ok(config.get[String]("foo"))
  }
}
```

