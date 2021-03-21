## 关于web.xml配置的那些事儿

### 1. 简介

web.xml文件是Java web项目中的一个配置文件，主要用于配置欢迎页、Filter、Listener、Servlet等，但并不是必须的，一个java web项目没有web.xml文件照样能跑起来。Tomcat容器`${TOMCAT_HOME}/conf`目录下也有作用于全局web应用web.xml文件，当一个web项目要启动时，Tomcat会首先加载项目中的web.xml文件，然后通过其中的配置来启动项目，只有配置的各项没有错误时，项目才能正常启动。

那么web.xml文件中到底有些什么内容呢？我们要如何去配置它以适应我们的项目呢？
首先让我们从Tomcat加载资源的顺序开始，一步步分析web.xml文件的作用。

> 这里顺便提一下web项目的war包结构;

```java
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

### 2. Tomcat加载资源顺序

Tomcat启动时加载资源主要有三个阶段：

- 第一阶段：JVM相关资源

  ```
  (1)$JAVA_HOME/jre/lib/ext/*.jar
  (2)系统classpath环境变量中的*.jar和*.class 
  ```

- 第二阶段：Tomcat自身相关资源

  ```
  (1)$CATALINA_HOME/common/classes/*.class  
  (2)$CATALINA_HOME/commons/endorsed/*.jar   
  (3)$CATALINA_HOME/commons/i18n/*.jar   
  (4)$CATALINA_HOME/common/lib/*.jar   
  (5)$CATALINA_HOME/server/classes/*.class   
  (6)$CATALINA_HOME/server/lib/*.jar   
  (7)$CATALINA_BASE/shared/classes/*.class   
  (8)$CATALINA_BASE/shared/lib/*.jar 
  ```

- 第三阶段：Web应用相关资源

  ```
  (1)具体应用的webapp目录: /WEB-INF/classes/*.class   
  (2)具体应用的webapp: /WEB-INF/lib/*.jar
  ```

> 在同一个文件夹下，jar包是按顺序从上到下依次加载,由ClassLoader的双亲委托模式加载机制我们可以知道，假设两个包名和类名完全相同的class文件不再同一个jar包，如果一个class文件已经被加载java虚拟机里了，那么后面的相同的class文件就不会被加载了。
>
> 但tomcat的加载运行机制与JAVA虚拟机的父类委托机制稍有不同。 
> 下面来做详细叙述: 
>
> 1. 首先加载${TOMCAT_HOME}/lib目录下的jar包 
>
> 2. 然后加载${TOMCAT_HOME}/webapps/项目名/WEB-INF/lib的jar包 
>
> 3. 最后加载的是${TOMCAT_HOME}/webapps/项目名/WEB-INF/classes下的类文件 
>
>    值得注意的关键是：tomcat按上述顺序依次加载资源，当后加载的资源与之前加载的资源重复时，后加载的资源会继续加载并将之前的资源覆盖。



### 3.web.xml配置文件简介

servlet和JSP规范在发展阶段中出现了很多的web.xml配置版本，如3.0、3.1、4.0等，版本的变更会改变web.xml对应的配置代码。下图是来自Tomcat官网的Servlet和JSP规范规范与的Apache Tomcat版本之间的对应关系：

![webXML](/Users/sherlock/Desktop/notes/allPics/SpringMVC/webXML.jpeg)



下面是各个版本的web.xml配置示例代码：

> servlet 2.5 [Tomcat 6.0.x(archived)]

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
   version="2.5">
   ...
</web-app>
```

> servlet 3.0 [Tomcat 7.0.x]

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
  version="3.0" metadata-complete="true">
  ...
</web-app>
```

> servlet 3.1 [Tomcat 8.0.x (superseded) & 8.5.x]

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
  version="3.1" metadata-complete="true">
  ...
</web-app>
```

> servlet 4.0 [Tomcat 9.0.x]

```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
  version="4.0" metadata-complete="true">
  ...
</web-app>
```

在tomcat目录`${CATALINA_HOME}/conf`下和web应用目录`${CATALINA_HOME}/webapps/WebDemo`(WebDemo为web应用名)下都有web.xml这个文件，但是内容不一样。

Tomcat在激活、加载、部署web应用时，会解析加载`${CATALINA_HOME}/conf`目录下所有web应用通用的web.xml，然后解析加载web应用目录中的`WEB-INF/web.xml`。
其实根据他们的位置，我们就可以知道，conf/web.xml文件中的设定会应用于所有的web应用程序，而某些web应用程序的WEB-INF/web.xml中的设定只应用于该应用程序本身。

如果没有WEB-INF/web.xml文件，tomcat会输出找不到的消息，但仍然会部署并使用web应用程序，servlet规范的作者想要实现一种能迅速并简易设定新范围的方法，以用作测试，因此，这个web.xml并不是必要的，不过通常最好还是让每一个上线的web应用程序都有一个自己的WEB-INF/web.xml。

### 3.web.xml元素配置详解

#### 3.1 web.xml各组件加载顺序

`ServletContext >> context-param >> listener >> filter >> servlet`

#### 3.2 web.xml中标签说明

```xml
<web-app> 
    <display-name></display-name>定义了WEB应用的名字 
    <description></description> 声明WEB应用的描述信息 
  
    <context-param></context-param> context-param元素声明应用范围内的初始化参数。 
    <filter></filter> 过滤器元素将一个名字与一个实现javax.servlet.Filter接口的类相关联。 
    <filter-mapping></filter-mapping> 一旦命名了一个过滤器，就要利用filter-mapping元素把它与一个或多个servlet或JSP页面相关联。 
    
    <listener></listener>servlet API的版本2.3增加了对事件监听程序的支持，事件监听程序在建立、修改和删除会话或servlet环境时得到通知。Listener元素指出事件监听程序类。 
    
    <servlet></servlet> 在向servlet或JSP页面制定初始化参数或定制URL时，必须首先命名servlet或JSP页面。Servlet元素就是用来完成此项任务的。 
    
    <servlet-mapping></servlet-mapping> 服务器一般为servlet提供一个缺省的URL：http://host/webAppPrefix/servlet/ServletName.
    
    但是，常常会更改这个URL，以便servlet可以访问初始化参数或更容易地处理相对URL。在更改缺省URL时，使用servlet-mapping元素。 
    
    <session-config></session-config> 如果某个会话在一定时间内未被访问，服务器可以抛弃它以节省内存。 可通过使用HttpSession的setMaxInactiveInterval方法明确设置单个会话对象的超时值，或者可利用session-config元素制定缺省超时值。 
    
    <mime-mapping></mime-mapping>如果Web应用具有想到特殊的文件，希望能保证给他们分配特定的MIME类型，则mime-mapping元素提供这种保证。 
    
    <welcome-file-list></welcome-file-list> 指示服务器在收到引用一个目录名而不是文件名的URL时，使用哪个文件。 
    
    <error-page></error-page> 在返回特定HTTP状态代码时，或者特定类型的异常被抛出时，能够制定将要显示的页面。 
    
    <taglib></taglib> 对标记库描述符文件（Tag Libraryu Descriptor file）指定别名。此功能使你能够更改TLD文件的位置，而不用编辑使用这些文件的JSP页面。 
    
    <resource-env-ref></resource-env-ref>声明与资源相关的一个管理对象。 
    <resource-ref></resource-ref> 声明一个资源工厂使用的外部资源。 
    <security-constraint></security-constraint> 制定应该保护的URL。它与login-config元素联合使用 
    <login-config></login-config> 指定服务器应该怎样给试图访问受保护页面的用户授权。它与sercurity-constraint元素联合使用。 
    
    <security-role></security-role>给出安全角色的一个列表，这些角色将出现在servlet元素内的security-role-ref元素的role-name子元素中。分别地声明角色可使高级IDE处理安全信息更为容易。 
    
    <env-entry></env-entry>声明Web应用的环境项。 
    <ejb-ref></ejb-ref>声明一个EJB的主目录的引用。 
    <ejb-local-ref></ejb-local-ref>声明一个EJB的本地主目录的应用。 
</web-app>
```

#### 3.3 相应元素的配置

1. Web应用图标：指出IDE和GUI工具用来表示Web应用的大图标和小图标

   ```xml
   <icon> 
       <small-icon>/images/app_small.gif</small-icon> 
       <large-icon>/images/app_large.gif</large-icon> 
   </icon> 
   ```

1. Web 应用名称：提供GUI工具可能会用来标记这个特定的Web应用的一个名称

   ```xml
   <display-name>Tomcat Example</display-name>
   ```

2. Web 应用描述： 给出于此相关的说明性文本

   ```xml
   <disciption>Tomcat Example servlets and JSP pages.</disciption>
   ```

3. 上下文参数：声明应用范围内的初始化参数。

   ```xml
   <context-param> 
       <param-name>ContextParameter</para-name> 
       <param-value>test</param-value> 
       <description>It is a test parameter.</description> 
   </context-param> 
   ```

   在servlet里面可以通过getServletContext().getInitParameter("context/param")得到。

   

4. 过滤器配置：将一个名字与一个实现javaxs.servlet.Filter接口的类相关联。

   ```xml
   <filter> 
       <filter-name>setCharacterEncoding</filter-name> 
       <filter-class>com.myTest.setCharacterEncodingFilter</filter-class> 
       <init-param> 
           <param-name>encoding</param-name> 
           <param-value>GB2312</param-value> 
       </init-param> 
   </filter> 
   <filter-mapping> 
       <filter-name>setCharacterEncoding</filter-name> 
       <url-pattern>/*</url-pattern> 
   </filter-mapping>
   ```

5. 监听器配置

   ```xml
   <listener> 
       <listerner-class>listener.SessionListener</listener-class> 
   </listener>
   ```

6. Servlet配置 
   ***基本配置***

   ```xml
   <servlet> 
   <servlet-name>snoop</servlet-name> 
   <servlet-class>SnoopServlet</servlet-class> 
   </servlet> 
   <servlet-mapping> 
   <servlet-name>snoop</servlet-name> 
   <url-pattern>/snoop</url-pattern> 
   </servlet-mapping> 
   ```

   ***高级配置***

   ```xml
      <servlet> 
          <servlet-name>snoop</servlet-name> 
          <servlet-class>SnoopServlet</servlet-class> 
          <init-param> 
              <param-name>foo</param-name> 
              <param-value>bar</param-value> 
          </init-param> 
          <run-as> 
              <description>Security role for anonymous access</description> 
              <role-name>tomcat</role-name> 
          </run-as> 
      </servlet> 
      <servlet-mapping> 
          <servlet-name>snoop</servlet-name> 
          <url-pattern>/snoop</url-pattern> 
      </servlet-mapping> 
   ```

   ***元素说明*** 

   ```xml
   <servlet></servlet> 用来声明一个servlet的数据，主要有以下子元素： 
   	<servlet-name></servlet-name> 指定servlet的名称 
   	<servlet-class></servlet-class> 指定servlet的类名称 
   	<jsp-file></jsp-file> 指定web站台中的某个JSP网页的完整路径 
   	<init-param>用来定义参数，可有多个init-param。在servlet类中通过getInitParamenter(String name)方法访问初始化参数 
       <param-name></param-name>
       <param-value></param-value>
   	</init-param> 
   	<load-on-startup></load-on-startup>指定当Web应用启动时，装载Servlet的次序。当值为正数或零时：Servlet容器先加载数值小的servlet，再依次加载其他数值大的servlet. 当值为负或未定义：Servlet容器将在Web客户首次访问这个servlet时加载它 
   <servlet-mapping>用来定义servlet所对应的URL，包含两个子元素
     <servlet-name></servlet-name> 指定servlet的名称 
     <url-pattern></url-pattern> 指定servlet所对应的URL 
   </servlet-mapping>  
   
   ```

7. 会话超时配置（单位为分钟）

   ```xml
   <session-config> 
   	<session-timeout>120</session-timeout> 
   </session-config> 
   ```

8. MIME类型配置

   ```xml
   <mime-mapping> 
     <extension>htm</extension> 
     <mime-type>text/html</mime-type> 
   </mime-mapping> 
   ```

9. 指定欢迎文件页配置

   ```xml
   <welcome-file-list> 
     <welcome-file>index.jsp</welcome-file> 
     <welcome-file>index.html</welcome-file> 
     <welcome-file>index.htm</welcome-file> 
   </welcome-file-list> 
   ```

10. 配置错误页面
    方法1：通过错误码来配置error-page, 当系统发生404错误时，跳转到错误处理页面NotFound.jsp。

    ```xml
    <error-page> 
      <error-code>404</error-code> 
      <location>/NotFound.jsp</location> 
    </error-page> 
    ```


    方法2：通过异常的类型配置error-page, 当系统发生java.lang.NullException（即空指针异常）时，跳转到错误处理页面error.jsp

    ```xml
    <error-page> 
      <exception-type>java.lang.NullException</exception-type> 
      <location>/error.jsp</location> 
    </error-page> 
    ```

11. TLD配置

    ```xml
    <taglib> 
      <taglib-uri>http://jakarta.apache.org/tomcat/debug-taglib</taglib-uri> 
      <taglib-location>/WEB-INF/jsp/debug-taglib.tld</taglib-location> 
    </taglib>
    ```

    如果MyEclipse一直在报错,应该把<taglib> 放到 <jsp-config>中 

    ```xml
    <jsp-config> 
      <taglib> 
          <taglib-uri>http://jakarta.apache.org/tomcat/debug-taglib</taglib-uri> 
          <taglib-location>/WEB-INF/pager-taglib.tld</taglib-location> 
      </taglib> 
    </jsp-config> 
    ```

12. 资源管理对象配置

    ```xml
    <resource-env-ref> 
    	<resource-env-ref-name>jms/StockQueue</resource-env-ref-name> 
    </resource-env-ref> 
    ```

13. 资源工厂配置

    ```xml
    <resource-ref> 
      <res-ref-name>mail/Session</res-ref-name> 
      <res-type>javax.mail.Session</res-type> 
      <res-auth>Container</res-auth> 
    </resource-ref> 
    ```

    配置数据库连接池就可在此配置：

    ```xml
    <resource-ref> 
      <description>JNDI JDBC DataSource of shop</description> 
      <res-ref-name>jdbc/sample_db</res-ref-name> 
      <res-type>javax.sql.DataSource</res-type> 
      <res-auth>Container</res-auth> 
    </resource-ref> 
    ```

14. 安全限制配置

    ```xml
    <security-constraint> 
    <display-name>Example Security Constraint</display-name> 
    <web-resource-collection> 
        <web-resource-name>Protected Area</web-resource-name> 
        <url-pattern>/jsp/security/protected/*</url-pattern> 
        <http-method>DELETE</http-method> 
        <http-method>GET</http-method> 
        <http-method>POST</http-method> 
        <http-method>PUT</http-method> 
    </web-resource-collection> 
    <auth-constraint> 
        <role-name>tomcat</role-name> 
        <role-name>role1</role-name> 
    </auth-constraint> 
    </security-constraint> 
    ```

15. 登陆验证配置

    ```xml
    <login-config> 
    <auth-method>FORM</auth-method> 
    <realm-name>Example-Based Authentiation Area</realm-name> 
    <form-login-config> 
        <form-login-page>/jsp/security/protected/login.jsp</form-login-page> 
        <form-error-page>/jsp/security/protected/error.jsp</form-error-page> 
    </form-login-config> 
    </login-config> 
    ```

16. 安全角色：security-role元素给出安全角色的一个列表，这些角色将出现在servlet元素内的security-role-ref元素的role-name子元素中。 
    分别地声明角色可使高级IDE处理安全信息更为容易。

    ```xml
    <security-role> 
    	<role-name>tomcat</role-name> 
    </security-role> 
    ```

17. Web环境参数：env-entry元素声明Web应用的环境项

    ```xml
    <env-entry> 
      <env-entry-name>minExemptions</env-entry-name> 
      <env-entry-value>1</env-entry-value> 
      <env-entry-type>java.lang.Integer</env-entry-type> 
    </env-entry> 
    ```

18. EJB 声明

    ```xml
    <ejb-ref> 
      <description>Example EJB reference</decription> 
      <ejb-ref-name>ejb/Account</ejb-ref-name> 
      <ejb-ref-type>Entity</ejb-ref-type> 
      <home>com.mycompany.mypackage.AccountHome</home> 
      <remote>com.mycompany.mypackage.Account</remote> 
    </ejb-ref> 
    ```

19. 本地EJB声明

    ```xml
    <ejb-local-ref> 
      <description>Example Loacal EJB reference</decription> 
      <ejb-ref-name>ejb/ProcessOrder</ejb-ref-name> 
      <ejb-ref-type>Session</ejb-ref-type> 
      <local-home>com.mycompany.mypackage.ProcessOrderHome</local-home> 
      <local>com.mycompany.mypackage.ProcessOrder</local> 
    </ejb-local-ref> 
    ```

20. 配置DWR

    ```xml
    <servlet> 
      <servlet-name>dwr-invoker</servlet-name> 
      <servlet-class>uk.ltd.getahead.dwr.DWRServlet</servlet-class> 
    </servlet> 
    <servlet-mapping> 
      <servlet-name>dwr-invoker</servlet-name> 
      <url-pattern>/dwr/*</url-pattern> 
    </servlet-mapping> 
    ```

### 4.spring项目中web.xml基础配置

配置基于servlet 3.1

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd" version="3.1" metadata-complete="true">

    <display-name>demo</display-name>
    <description>demo</description>
    
    <!-- 在Spring框架中是如何解决从页面传来的字符串的编码问题的呢？  
    下面我们来看看Spring框架给我们提供过滤器CharacterEncodingFilter  
     这个过滤器就是针对于每次浏览器请求进行过滤的，然后再其之上添加了父类没有的功能即处理字符编码。  
      其中encoding用来设置编码格式，forceEncoding用来设置是否理会 request.getCharacterEncoding()方法，设置为true则强制覆盖之前的编码格式。-->
    <filter>
        <filter-name>characterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>characterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <!-- 项目中使用Spring 时，applicationContext.xml配置文件中并没有BeanFactory，要想在业务层中的class 文件中直接引用Spring容器管理的bean可通过以下方式-->
    <!--1、在web.xml配置监听器ContextLoaderListener-->
    <!--ContextLoaderListener的作用就是启动Web容器时，自动装配ApplicationContext的配置信息。因为它实现了ServletContextListener这个接口，在web.xml配置这个监听器，启动容器时，就会默认执行它实现的方法。  
    在ContextLoaderListener中关联了ContextLoader这个类，所以整个加载配置过程由ContextLoader来完成。  
    它的API说明  
    第一段说明ContextLoader可以由 ContextLoaderListener和ContextLoaderServlet生成。  
    如果查看ContextLoaderServlet的API，可以看到它也关联了ContextLoader这个类而且它实现了HttpServlet    这个接口  
    第二段，ContextLoader创建的是 XmlWebApplicationContext这样一个类，它实现的接口是WebApplicationContext->ConfigurableWebApplicationContext->ApplicationContext->  
    BeanFactory这样一来spring中的所有bean都由这个类来创建  
     IUploaddatafileManager uploadmanager = (IUploaddatafileManager)  
     ContextLoaderListener.getCurrentWebApplicationContext().getBean("uploadManager");-->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <!--2、部署applicationContext的xml文件-->
    <!--如果在web.xml中不写任何参数配置信息，默认的路径是"/WEB-INF/applicationContext.xml，  
    在WEB-INF目录下创建的xml文件的名称必须是applicationContext.xml。  
    如果是要自定义文件名可以在web.xml里加入contextConfigLocation这个context参数：  
    在<param-value> </param-value>里指定相应的xml文件名，如果有多个xml文件，可以写在一起并以“,”号分隔。  
    也可以这样applicationContext-*.xml采用通配符，比如这那个目录下有applicationContext-ibatis-base.xml，  
    applicationContext-action.xml，applicationContext-ibatis-dao.xml等文件，都会一同被载入。  
    在ContextLoaderListener中关联了ContextLoader这个类，所以整个加载配置过程由ContextLoader来完成。-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/applicationContext.xml</param-value>
    </context-param>
    <!--如果你的DispatcherServlet拦截"/"，为了实现REST风格，拦截了所有的请求，那么同时对*.js,*.jpg等静态文件的访问也就被拦截了。-->
    <!--方案一：激活Tomcat的defaultServlet来处理静态文件-->
    <!--要写在DispatcherServlet的前面， 让 defaultServlet先拦截请求，这样请求就不会进入Spring了，我想性能是最好的吧。-->
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.css</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.swf</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.gif</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.jpg</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.png</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.js</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.html</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.xml</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.json</url-pattern>
    </servlet-mapping>
    <servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>*.map</url-pattern>
    </servlet-mapping>
    <!--使用Spring MVC,配置DispatcherServlet是第一步。DispatcherServlet是一个Servlet,,所以可以配置多个DispatcherServlet-->
    <!--DispatcherServlet是前置控制器，配置在web.xml文件中的。拦截匹配的请求，Servlet拦截匹配规则要自已定义，把拦截下来的请求，依据某某规则分发到目标Controller(我们写的Action)来处理。-->
    <servlet>
        <servlet-name>DispatcherServlet</servlet-name>
        <!--在DispatcherServlet的初始化过程中，框架会在web应用的 WEB-INF文件夹下寻找名为[servlet-name]-servlet.xml 的配置文件，生成文件中定义的bean。-->
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!--指明了配置文件的文件名，不使用默认配置文件名，而使用dispatcher-servlet.xml配置文件。-->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <!--其中<param-value>**.xml</param-value> 这里可以使用多种写法-->
            <!--1、不写,使用默认值:/WEB-INF/<servlet-name>-servlet.xml-->
            <!--2、<param-value>/WEB-INF/classes/dispatcher-servlet.xml</param-value>-->
            <!--3、<param-value>classpath*:dispatcher-servlet.xml</param-value>-->
            <!--4、多个值用逗号分隔-->
            <param-value>classpath:spring/dispatcher-servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
        <!--是启动顺序，让这个Servlet随Servletp容器一起启动。-->
    </servlet>
    <servlet-mapping>
        <!--这个Servlet的名字是dispatcher，可以有多个DispatcherServlet，是通过名字来区分的。每一个DispatcherServlet有自己的WebApplicationContext上下文对象。同时保存的ServletContext中和Request对象中.-->
        <!--ApplicationContext是Spring的核心，Context我们通常解释为上下文环境，我想用“容器”来表述它更容易理解一些，ApplicationContext则是“应用的容器”了:P，Spring把Bean放在这个容器中，在需要的时候，用getBean方法取出-->
        <servlet-name>DispatcherServlet</servlet-name>
        <!--Servlet拦截匹配规则可以自已定义，当映射为@RequestMapping("/user/add")时，为例,拦截哪种URL合适？-->
        <!--1、拦截*.do、*.htm， 例如：/user/add.do,这是最传统的方式，最简单也最实用。不会导致静态文件（jpg,js,css）被拦截。-->
        <!--2、拦截/，例如：/user/add,可以实现现在很流行的REST风格。很多互联网类型的应用很喜欢这种风格的URL。弊端：会导致静态文件（jpg,js,css）被拦截后不能正常显示。 -->
        <url-pattern>/</url-pattern>
        <!--会拦截URL中带“/”的请求。-->
    </servlet-mapping>
    <welcome-file-list>
        <!--指定欢迎页面-->
        <welcome-file>login.html</welcome-file>
    </welcome-file-list>
    <error-page>
        <!--当系统出现404错误，跳转到页面nopage.html-->
        <error-code>404</error-code>
        <location>/nopage.html</location>
    </error-page>
    <error-page>
        <!--当系统出现java.lang.NullPointerException，跳转到页面error.html-->
        <exception-type>java.lang.NullPointerException</exception-type>
        <location>/error.html</location>
    </error-page>
    <session-config>
        <!--会话超时配置，单位分钟-->
        <session-timeout>360</session-timeout>
    </session-config>
</web-app>
```

### 5.流行的web.xml零配置

servlet3.0+规范后，允许servlet，filter，listener不必声明在web.xml中，而是以硬编码的方式存在，实现容器的零配置。

spring框架提供了一些类如`WebApplicationInitializer`用来配置web.xml，这意味着我们可以舍弃web.xml，仅仅在主程序代码中进行配置。

spring的“约定大于配置”思想也体现在这里，用java注解来优化过去繁杂的xml文件配置，大大提高了开发者的编程速度和体验。

现在很流行的spring boot框架，只要几步就可以搭建一个java web项目，根本无需自己手动配置web.xml，框架已经为你提供了足够多的注解、接口及实现类，让我们能够专注于应用本身。

有关spring对web.xml配置的隐藏实现细节，这里就不详细展开了，欢迎关注我的下一篇博客。

