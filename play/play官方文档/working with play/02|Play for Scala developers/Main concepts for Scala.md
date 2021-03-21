# Main concepts for Scala

本节向您介绍在 Scala 中编写 Play 应用程序的最常见方面。您将了解有关处理HTTP请求，发送HTTP响应，使用不同类型的数据，使用数据库等的知识。

> Scala 和 Java 的 Play API 分为不同的程序包。所有的 Scala API 都在 `play.api ` 软件包中。所有 Java API 都在之下 `play` 包中。例如，Scala MVC API 在之下`play.api.mvc`，而 Java MVC API 在`play.mvc` 之下。



## Configuration API

1. [The Configuration API](https://www.playframework.com/documentation/2.8.x/ScalaConfig)



## HTTP programming

1. [Actions, Controllers and Results](https://www.playframework.com/documentation/2.8.x/ScalaActions)
2. [HTTP Routing](https://www.playframework.com/documentation/2.8.x/ScalaRouting)
3. [Manipulating HTTP results](https://www.playframework.com/documentation/2.8.x/ScalaResults)
4. [Session and Flash scopes](https://www.playframework.com/documentation/2.8.x/ScalaSessionFlash)
5. [Body parsers](https://www.playframework.com/documentation/2.8.x/ScalaBodyParsers)
6. [Actions composition](https://www.playframework.com/documentation/2.8.x/ScalaActionsComposition)
7. [Content negotiation](https://www.playframework.com/documentation/2.8.x/ScalaContentNegotiation)
8. [Handling errors](https://www.playframework.com/documentation/2.8.x/ScalaErrorHandling)



## Asynchronous HTTP programming

1. [Asynchronous results](https://www.playframework.com/documentation/2.8.x/ScalaAsync)
2. [Streaming HTTP responses](https://www.playframework.com/documentation/2.8.x/ScalaStream)
3. [Comet](https://www.playframework.com/documentation/2.8.x/ScalaComet)
4. [WebSockets](https://www.playframework.com/documentation/2.8.x/ScalaWebSockets)



## The Twirl template engine

1. [Templates syntax](https://www.playframework.com/documentation/2.8.x/ScalaTemplates)
2. [Dependency Injection with Templates](https://www.playframework.com/documentation/2.8.x/ScalaTemplatesDependencyInjection)
3. [Common use cases](https://www.playframework.com/documentation/2.8.x/ScalaTemplateUseCases)
4. [Custom format](https://www.playframework.com/documentation/2.8.x/ScalaCustomTemplateFormat)



## Form submission and validation

1. [Handling form submission](https://www.playframework.com/documentation/2.8.x/ScalaForms)
2. [Protecting against CSRF](https://www.playframework.com/documentation/2.8.x/ScalaCsrf)
3. [Custom Validations](https://www.playframework.com/documentation/2.8.x/ScalaCustomValidations)
4. [Custom Field Constructors](https://www.playframework.com/documentation/2.8.x/ScalaCustomFieldConstructors)



## Working with Json

1. [JSON basics](https://www.playframework.com/documentation/2.8.x/ScalaJson)
2. [JSON with HTTP](https://www.playframework.com/documentation/2.8.x/ScalaJsonHttp)
3. [JSON Reads/Writes/Format Combinators](https://www.playframework.com/documentation/2.8.x/ScalaJsonCombinators)
4. [JSON automated mapping](https://www.playframework.com/documentation/2.8.x/ScalaJsonAutomated)
5. [JSON Transformers](https://www.playframework.com/documentation/2.8.x/ScalaJsonTransformers)



## Working with XML

1. [Handling and serving XML requests](https://www.playframework.com/documentation/2.8.x/ScalaXmlRequests)



## Handling file upload

1. [Direct upload and multipart/form-data](https://www.playframework.com/documentation/2.8.x/ScalaFileUpload)



## Accessing an SQL database

1. [Accessing an SQL Database](https://www.playframework.com/documentation/2.8.x/AccessingAnSQLDatabase)
2. Using Slick to access your database
   1. [Using Play Slick](https://www.playframework.com/documentation/2.8.x/PlaySlick)
   2. [Play Slick migration guide](https://www.playframework.com/documentation/2.8.x/PlaySlickMigrationGuide)
   3. [Play Slick advanced topics](https://www.playframework.com/documentation/2.8.x/PlaySlickAdvancedTopics)
   4. [Play Slick FAQ](https://www.playframework.com/documentation/2.8.x/PlaySlickFAQ)
3. [Using Anorm to access your database](https://www.playframework.com/documentation/2.8.x/ScalaAnorm)



## Using the Cache

1. [Using the Cache](https://www.playframework.com/documentation/2.8.x/ScalaCache)



## Calling REST APIs with Play WS

1. [The Play WS API](https://www.playframework.com/documentation/2.8.x/ScalaWS)
2. [Connecting to OpenID services](https://www.playframework.com/documentation/2.8.x/ScalaOpenID)
3. [Accessing resources protected by OAuth](https://www.playframework.com/documentation/2.8.x/ScalaOAuth)



## Integrating with Akka

1. [Integrating with Akka](https://www.playframework.com/documentation/2.8.x/ScalaAkka)



## Internationalization with Messages

1. [Internationalization with Messages](https://www.playframework.com/documentation/2.8.x/ScalaI18N)



## Dependency Injection

1. [Dependency Injection with Guice](https://www.playframework.com/documentation/2.8.x/ScalaDependencyInjection)
2. [Compile Time Dependency Injection](https://www.playframework.com/documentation/2.8.x/ScalaCompileTimeDependencyInjection)



## Application Settings

1. [Application Settings](https://www.playframework.com/documentation/2.8.x/ScalaApplication)
2. [HTTP request handlers](https://www.playframework.com/documentation/2.8.x/ScalaHttpRequestHandlers)
3. [Essential Actions](https://www.playframework.com/documentation/2.8.x/ScalaEssentialAction)
4. [HTTP filters](https://www.playframework.com/documentation/2.8.x/ScalaHttpFilters)



## Testing your application

1. [Testing your Application](https://www.playframework.com/documentation/2.8.x/ScalaTestingYourApplication)
2. [Testing with ScalaTest](https://www.playframework.com/documentation/2.8.x/ScalaTestingWithScalaTest)
3. [Writing functional tests with ScalaTest](https://www.playframework.com/documentation/2.8.x/ScalaFunctionalTestingWithScalaTest)
4. [Testing with specs2](https://www.playframework.com/documentation/2.8.x/ScalaTestingWithSpecs2)
5. [Writing functional tests with specs2](https://www.playframework.com/documentation/2.8.x/ScalaFunctionalTestingWithSpecs2)
6. [Testing with Guice](https://www.playframework.com/documentation/2.8.x/ScalaTestingWithGuice)
7. [Testing with compile-time Dependency Injection](https://www.playframework.com/documentation/2.8.x/ScalaTestingWithComponents)
8. [Testing with databases](https://www.playframework.com/documentation/2.8.x/ScalaTestingWithDatabases)
9. [Testing web service clients](https://www.playframework.com/documentation/2.8.x/ScalaTestingWebServiceClients)



## Logging

1. [Logging](https://www.playframework.com/documentation/2.8.x/ScalaLogging)

