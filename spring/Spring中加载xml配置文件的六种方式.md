## Spring中加载xml配置文件的六种方式

Spring中加载xml配置文件的方式有6种, xml是最常见的spring 应用系统配置源。Spring中的几种容器都支持使用xml装配bean，包括： 

1. XmlBeanFactory
2. ClassPathXmlApplicationContext
3. FileSystemXmlApplicationContext
4. XmlWebApplicationContext

### 1. XmlBeanFactory 引用资源 

```java
Resource resource = new ClassPathResource("appcontext.xml"); 
BeanFactory factory = new XmlBeanFactory(resource); 
```



### 2. ClassPathXmlApplicationContext 编译路径 

```java
ApplicationContext factory=new ClassPathXmlApplicationContext("classpath:appcontext.xml"); 
// src目录下的 
ApplicationContext factory=new ClassPathXmlApplicationContext("appcontext.xml"); 
ApplicationContext factory=new ClassPathXmlApplicationContext(new String[] {"bean1.xml","bean2.xml"}); 
// src/conf 目录下的 
ApplicationContext factory=new ClassPathXmlApplicationContext("conf/appcontext.xml"); 
ApplicationContext factory=new ClassPathXmlApplicationContext("file:G:/Test/src/appcontext.xml"); 
```

### 3. 用文件系统的路径 

```java
ApplicationContext factory=new FileSystemXmlApplicationContext("src/appcontext.xml"); 
//使用了 classpath: 前缀,作为标志, 这样,FileSystemXmlApplicationContext 也能够读入classpath下的相对路径 
ApplicationContext factory=new FileSystemXmlApplicationContext("classpath:appcontext.xml"); 
ApplicationContext factory=new FileSystemXmlApplicationContext("file:G:/Test/src/appcontext.xml"); 
ApplicationContext factory=new FileSystemXmlApplicationContext("G:/Test/src/appcontext.xml"); 
```

### 4. XmlWebApplicationContext是专为Web工程定制的。 

```java
ServletContext servletContext = request.getSession().getServletContext(); 
ApplicationContext ctx = WebApplicationContextUtils.getWebApplicationContext(servletContext ); 
```

### 5. 使用BeanFactory 

```java
BeanDefinitionRegistry reg = new DefaultListableBeanFactory(); 
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(reg); 
reader.loadBeanDefinitions(new ClassPathResource("bean1.xml")); 
reader.loadBeanDefinitions(new ClassPathResource("bean2.xml")); 
BeanFactory bf=(BeanFactory)reg; 
```

### 6. Web 应用启动时加载多个配置文件 

通过ContextLoaderListener 也可加载多个配置文件，在web.xml文件中利用 
<context-pararn>元素来指定多个配置文件位置，其配置如下: 

```xml
<context-param> 
<!-- Context Configuration locations **for** Spring XML files --> 
	<param-name>contextConfigLocation</param-name> 
	<param-value> 
		./WEB-INF/**/Appserver-resources.xml, 
		classpath:config/aer/aerContext.xml, 
		classpath:org/codehaus/xfire/spring/xfire.xml, 
		./WEB-INF/**/*.spring.xml 
	</param-value> 
</context-param>  
```

