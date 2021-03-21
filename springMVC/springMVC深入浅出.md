## SpringMVC的深入浅出

### 前言

使用SpringMVC只要引入一个依赖spring-webmvc即可同时使用Spring和SpringMVC的功能

```
<dependency>
	<groupId>org.springframework</groupId>
  <artifactId>spring-webmvc</artifactId>
  <version>${spring.version}</version>
</dependency>
```

maven中添加上述依赖后可以发现项目中除了引入mvc所需jar外, 同时还引入了Spring的DI和AOP的所需jar包

![springMVC依赖](/Users/sherlock/Desktop/notes/allPics/SpringMVC/springMVC依赖.png)



在编码过程中还需要使用到`javax.servlet.http.HttpServletRequest`这个类, 所以需要在开发环境下找到`${TOMCAT_HOME}/lib/servlet-api.jar`或者引入依赖:( 需要找到当前tomcat版本能适配的serlvet版本, 例如Tomcat-8.5.x就该使用3.1及以上版本的servlet-api.jar )

```
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>
```



### MVC 设计概述

在早期 Java Web 的开发中，统一把显示层、控制层、数据层的操作全部交给 JSP 或者 JavaBean 来进行处理，我们称之为 **Model1：**

![早期MVC](/Users/sherlock/Desktop/notes/allPics/SpringMVC/早期MVC.png)

**出现的弊端：**

- JSP 和 Java Bean 之间严重耦合，Java 代码和 HTML 代码也耦合在了一起

- 要求开发者不仅要掌握 Java ，还要有高超的前端水平

- 前端和后端相互依赖，前端需要等待后端完成，后端也依赖前端完成，才能进行有效的测试

- 代码难以复用

正因为上面的种种弊端，所以很快这种方式就被 Servlet + JSP + Java Bean 所替代了，早期的 MVC 模型**（Model2）**就像下图这样：

![早期MVC2](/Users/sherlock/Desktop/notes/allPics/SpringMVC/早期MVC2.png)

首先用户的请求会到达 Servlet，然后根据请求调用相应的 Java Bean，并把所有的显示结果交给 JSP 去完成，这样的模式我们就称为 MVC 模式。

- **M 代表 模型（Model）**
   模型是什么呢？ 模型就是数据，就是 dao,bean
- **V 代表 视图（View）**
   视图是什么呢？ 就是网页, JSP，用来展示模型中的数据
- **C 代表 控制器（controller)**
   控制器是什么？ 控制器的作用就是把不同的数据(Model)，显示在不同的视图(View)上，Servlet 扮演的就是这样的角色。

### Spring MVC 

#### SpringMVC架构

为解决持久层中一直未处理好的数据库事务的编程，又为了迎合 NoSQL 的强势崛起，Spring MVC 给出了方案：

![springMVC架构](/Users/sherlock/Desktop/notes/allPics/SpringMVC/springMVC架构.png)

#### Spring MVC 的请求

每当用户在 Web 浏览器中点击链接或者提交表单的时候，请求就开始工作了，像是邮递员一样，从离开浏览器开始到获取响应返回，它会经历很多站点，在每一个站点都会留下一些信息同时也会带上其他信息，下图为 Spring MVC 的请求流程：

![springMVC组件](/Users/sherlock/Desktop/notes/allPics/SpringMVC/springMVC组件.png)

##### 第一站：DispatcherServlet

从请求离开浏览器以后，第一站到达的就是 DispatcherServlet，看名字这是一个 Servlet，通过 J2EE 的学习，我们知道 Servlet 可以拦截并处理 HTTP 请求，DispatcherServlet 会拦截所有的请求，并且将这些请求发送给 Spring MVC 控制器。

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <!-- 拦截所有的请求 -->
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

- **DispatcherServlet 的任务就是拦截请求发送给 Spring MVC 控制器。**

##### 第二站：HandlerMapping(处理器映射)

- **问题：** 典型的应用程序中可能会有多个控制器，这些请求到底应该发给哪一个控制器呢？

所以 DispatcherServlet 会查询一个或多个处理器映射来确定请求的下一站在哪里，处理器映射会**根据请求所携带的 URL 信息来进行决策**，例如上面的例子中，我们通过配置 simpleUrlHandlerMapping 来将 /hello 地址交给 helloController 处理：

```jsx
<bean id="simpleUrlHandlerMapping"
      class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="mappings">
        <props>
            <!-- /hello 路径的请求交给 id 为 helloController 的控制器处理-->
            <prop key="/hello">helloController</prop>
        </props>
    </property>
</bean>
<bean id="helloController" class="controller.HelloController"></bean>
```

##### 第三站：控制器

一旦选择了合适的控制器， DispatcherServlet 会将请求发送给选中的控制器，到了控制器，请求会卸下其负载（用户提交的请求）等待控制器处理完这些信息：

```java
public ModelAndView handleRequest(javax.servlet.http.HttpServletRequest httpServletRequest, javax.servlet.http.HttpServletResponse httpServletResponse) throws Exception {
    // 处理逻辑
    ....
}
```

##### 第四站：返回 DispatcherServlet

当控制器在完成逻辑处理后，通常会产生一些信息，这些信息就是需要返回给用户并在浏览器上显示的信息，它们被称为**模型（Model）**。仅仅返回原始的信息时不够的——这些信息需要以用户友好的方式进行格式化，一般会是 HTML，所以，信息需要发送给一个**视图（view）**，通常会是 JSP。

控制器所做的最后一件事就是将模型数据打包，并且表示出用于渲染输出的视图名**（逻辑视图名）。它接下来会将请求连同模型和视图名发送回 DispatcherServlet。**

```java
public ModelAndView handleRequest(javax.servlet.http.HttpServletRequest httpServletRequest, javax.servlet.http.HttpServletResponse httpServletResponse) throws Exception {
    // 处理逻辑
    ....
    // 返回给 DispatcherServlet
    return mav;
}
```

##### 第五站：视图解析器

这样以来，控制器就不会和特定的视图相耦合，传递给 DispatcherServlet 的视图名并不直接表示某个特定的 JSP。（实际上，它甚至不能确定视图就是 JSP）相反，**它传递的仅仅是一个逻辑名称，这个名称将会用来查找产生结果的真正视图。**

DispatcherServlet 将会使用视图解析器（view resolver）来将逻辑视图名匹配为一个特定的视图实现，它可能是也可能不是 JSP

> 上面的例子是直接绑定到了 index.jsp 视图

##### 第六站：视图

既然 DispatcherServlet 已经知道由哪个视图渲染结果了，那请求的任务基本上也就完成了。

它的最后一站是视图的实现，在这里它交付模型数据，请求的任务也就完成了。视图使用模型数据渲染出结果，这个输出结果会通过响应对象传递给客户端。

```xml
<%@ page language="java" contentType="text/html; charset=UTF-8"
         pageEncoding="UTF-8" isELIgnored="false"%>

<h1>${message}</h1>
```

#### Spring MVC 的注解配置

