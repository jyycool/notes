## SpringMVC中的注解标签及配置标签

### 配置文件中的标签

#### web.xml

配置示例:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/applicationContext.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```



##### context-param

1. 启动一个WEB项目的时候,容器(如:Tomcat)会去读它的配置文件web.xml.读两个节点: <listener></listener> 和 <context-param></context-param>

2. 紧接着,容器创建一个ServletContext(上下文),这个WEB项目所有部分都共享这个上下文.

3. 容器将<context-param></context-param>转化为键值对,并交给ServletContext.

4. 容器创建<listener></listener>中的类实例, 即创建监听( 备注：listener定义的类可以是自定义的类但必须需要继承ServletContextListener )

5. 在监听中会有contextInitialized(ServletContextEvent args)初始化方法,在这个方法中获得ServletContext = ServletContextEvent.getServletContext();
   context-param的值 = ServletContext.getInitParameter("context-param的键");

6. 得到这个context-param的值之后,你就可以做一些操作了.注意,这个时候你的WEB项目还没有完全启动完成.这个动作会比所有的Servlet都要早.
   换句话说,这个时候,你对<context-param>中的键值做的操作,将在你的WEB项目完全启动之前被执行.

7. 举例.你可能想在项目启动之前就打开数据库.
   那么这里就可以在<context-param>中设置数据库的连接方式,在监听类中初始化数据库的连接.

8. 这个监听是自己写的一个类,除了初始化方法,它还有销毁方法.用于关闭应用前释放资源.比如说数据库连接的关闭.

> **由上面的初始化过程可知容器对于web.xml的加载过程是:**``context-param >> listener >> fileter >> servlet``



##### init-param

```xml
<servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
      	<init-param>
         	<param-name>contextConfigLocation</param-name>
         	<param-value>classpath: springmvc.xml</param-value>
      	</init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
```

##### load-on-startup



### 注解

#### @RequestMapping

用于建立***<u>请求URL与处理请求方法</u>***之间的对应关系, 可用于类或方法上

> 当做用于类上的时候表示类中的所有响应请求的方法都是以该地址作为父路径。

RequestMapping注解有六个属性，下面分成三类进行说明。

1. ***value和method***

   - ***value***

     指定请求的实际地址，指定的地址可以是具体地址、可以RestFul动态获取、也可以使用正则设置

   - ***method*** 

     指定请求的method类型， 分为***GET***、***POST***、***PUT***、***DELETE***等

2. ***consumes和produces***

   - ***consumes***

     指定处理请求的提交内容类型( Content-Type ), 例如application/json, text/html

   - ***produces***

     指定返回内容类型, 仅当request请求头中的(Accept)类型中包含该指定类型才返回

3. ***params和headers***

   - ***params***

     指定request中必须包含某些参数值是，才让该方法处理。

   - ***headers***

     指定request中必须包含某些指定的header值，才能让该方法处理请求。
     

#### @RequestParam

把请求中指定名称的参数传递给控制器的形参

RequestParam有三个属性，分别如下：

1. value 请求参数的参数名,作为参数映射名称；
2. required 该参数是否必填，默认true(必填)，当设置成必填时，若没有传入参数，报错
3. defaultValue 设置请求参数的默认值；