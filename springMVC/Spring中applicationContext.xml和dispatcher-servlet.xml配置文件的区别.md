## Spring中applicationContext.xml和dispatcher-servlet.xml配置文件的区别

在SpringMVC项目中我们一般会引入applicationContext.xml和dispatcher-servlet.xml两个配置文件，这两个配置文件具体的区别是什么呢？

### 官方文档

>Spring lets you define multiple contexts in a parent-child hierarchy. 
>The applicationContext.xml defines the beans for the "root webapp context", i.e. the context associated with the webapp. 
>The spring-servlet.xml (or whatever else you call it) defines the beans for one servlet's app context. There can be many of these in a webapp, one per Spring servlet (e.g. spring1-servlet.xml for servlet spring1, spring2-servlet.xml for servlet spring2). 
>Beans in spring-servlet.xml can reference beans in applicationContext.xml, but not vice versa. 
>All Spring MVC controllers must go in the spring-servlet.xml context. 
>In most simple cases, the applicationContext.xml context is unnecessary. It is generally used to contain beans that are shared between all servlets in a webapp. If you only have one servlet, then there's not really much point, unless you have a specific use for it.  

使用翻译后如下

>Spring允许您在父子层次结构中定义多个上下文。
>applicationContext.xml定义了“root webapp上下文”的bean，即与webapp相关联的上下文。
>spring-servlet.xml( 或你自定义的其他任何名称的xml )定义了一个servlet应用程序上下文的bean。在一个web应用程序中可以有许多这样的组件，每个Spring一个(例如，spring1对应servlet就是spring1的spring1-servlet.xml, spring2对应servlet就是spring2的spring2-servlet.xml)。
>spring-servlet.xml中可以引用在applicationContext中定义的bean。反之则不行。
>所有Spring MVC控制器都必须放在Spring-servlet.xml文中。
>在大多数简单的情况下，applicationContext.xml上下文是不必要的。它通常用于包含web应用程序中所有servlet之间共享的bean。如果您只有一个servlet，那么没有什么意义，除非您有它的特定用途。

### 用途不同

- applicationContext.xml文件通常用于加载spring系统级别的组件, 比如bean的初始化
- spring-mvc.xml文件通常用于加载controller层需要的类, 比如拦截器, mvc标签加载的类。

> ***注意:*** 
>
> ​		例如: 我们在applicationContext.xml中开启了构造型注解的包扫描 \<context:component-scan> , 在dispatch-servlet.xml也中需要开启mvc环境的@Controller注解的包扫描\<context:component-scan>, dispatch-servlet.xml中包扫描注解不会去引用或干扰applicationContext.xml中的包扫描注解



### 加载位置不同

- applicationContext.xml加载在标签中，作为FrameworkServlet的参数属性。
- spring-mvc.xml文件当做DispatcherServlet的参数属性进行加载。

> applicationContext.xml是全局的，应用于多个servelet，配合listener一起使用。
>  web.xml中配置如下：

```xml
<!-- 配置监听器 -->
<listener>        
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>
```

> spring-mvc.xml是SpringMVC的配置，web.xml中配置如下：

```xml
<!--配置SpringMVC DispatcherServlet-->
<servlet>
  <servlet-name>DispatcherServlet</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  <init-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:config/spring-mvc.xml</param-value>
  </init-param>
  <load-on-startup>1</load-on-startup>
  <async-supported>true</async-supported>
</servlet>
```

- ***application-context.xml这个一般是采用非SpringMVC架构，用来加载Application Context。***
- ***如果直接采用SpringMVC，只需要把所有相关配置放到spring-mvc.xml中就好，一般SpringMVC项目用不到多个serverlet。***

可见， applicationContext.xml 和 dispatch-servlet.xml形成了两个父子关系的上下文。

1.  一个bean如果在两个文件中都被定义了(比如两个文件中都定义了component scan扫描相同的package)， spring会在application context和 servlet context中都生成一个实例，他们处于不同的上下文空间中，他们的行为方式是有可能不一样的。

2. 如果在application context和 servlet context中都存在同一个 @Service 的实例， controller（在servlet context中） 通过 @Resource引用时， 会优先选择servlet context中的实例。

3. 最好是：在applicationContext和dispatcher-servlet定义的bean最好不要重复， dispatcher-servlet最好只是定义controller类型的bean。
   

> applicationContext.xml  是spring 全局配置文件，用来控制spring 特性的
>
> dispatcher-servlet.xml 是spring mvc里面的，控制器、拦截uri转发view
>
> 使用applicationContext.xml文件时是需要在web.xml中添加listener的：



### web.xml配置文件的说明

```xml
<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

> ContextLoaderListener是Spring的监听器，它的作用就是启动Web容器时，自动装配ApplicationContext的配置信息。因为它实现了ServletContextListener这个接口，在web.xml配置这个监听器，启动容器时，就会默认执行它实现的方法。



```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/applicationContext.xml</param-value>
     <!--<param-value>classpath*:config/applicationContext.xml</param-value> -->       
</context-param>
```

> 这段配置是用于指定applicationContext.xml配置文件的位置，可通过context-param加以指定。



> 如果applicationContext.xml配置文件存放在src目录下，那么在web.xml中的配置就如下所示：

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>
```

> 如果applicationContext.xml配置文件存放在WEB-INF下面，那么在web.xml中的配置就如下所示：

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>WEB-INF/applicationContext*.xml</param-value>
</context-param>
```



需要注意的是，部署到应用服务器后，src目录下的配置文件会和class文件一样，自动copy到应用的classes目录下，spring的配置文件在启动时，加载的是WEB-INF目录下的applicationContext.xml运行时使用的是WEB-INF/classes目录下的applicationContext.xml。因此，不管applicationContext.xml配置文件存放在src目录下，还是存放在WEB-INF下面，都可以用下面这种方式来配置路径：

```xml
<context-param>
   <param-name>contextConfigLocation</param-name>
   <param-value>WEB-INF/applicationContext*.xml</param-value>
</context-param>
```

***<u>注意：</u>***

1. classpath是指 WEB-INF文件夹下的classes目录
2. classpath与classpath*的区别:
   - classpath：指classes路径下文件
   - classpath*:除了classes路径下文件，还包含jar包中文件



### war包内部结构

```kotlin
webapp.war
    |—index.jsp
    |
    |— images
    |— META-INF
    |— WEB-INF
          |— web.xml                   // WAR包的描述文件
          |
          |— classes
          |        action.class        // java类文件
          |
          |— lib
                  other.jar             // 依赖的jar包
                  share.jar

代码结构里的java和resources打包后都会放到WEB-INF/classes里面
WEB-INF里的文件会直接放在WEB-INF里面
```