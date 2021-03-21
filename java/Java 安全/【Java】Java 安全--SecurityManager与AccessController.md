# 【Java】Java 安全--SecurityManager与AccessController

## 前言

### 什么是安全？

- 程序不能恶意破坏用户计算机的环境，比如特洛伊木马等可自我进行复制的恶意程序。
- 程序不可获取主机及其所在网络的私密信息。
- 程序的提供者和使用者的身份需要通过特殊验证。
- 程序所涉及的数据在传输、持久化后都应是被加密的。
- 程序的操作有相关规则限制，并且不能耗费过多的系统资源。

保护计算机上的信息不被非法获取和修改是 Java 最初的，也是最基本的设计目标，但同时还要保证Java 程序在主机上的运行不受影响。

### Java安全方面的支持

JDK 本身提供了基本的安全方面的功能，比如可配置的安全策略、生成消息摘要、生成数字签名等等。同时，Java 也有一些扩展程序，更加全面地支撑了整个安全体系。

- Java加密扩展包(JCE): 提供了密码、安全密钥交换、安全消息摘要、密钥管理系统等功能。

- Java安全套接字扩展包(JSSE): 提供了SSL(安全套接字层)的加密功能，保证了与SSL服务器或SSL客户的通信安全。

- Java鉴别与授权服务(JAAS): 可以在Java平台上提供用户鉴别，并且允许开发者根据用户提供的鉴别信任状准许或拒绝用户对程序的访问。

## 一、Java sandbox

如何理解？程序要在主机上安装，那么主机必须为该程序提供一个运行的场所（运行环境），该场所支持程序运行的同时，也限制其可以获取的资源。好比小孩子去你家玩，你需要提供一个空间让她玩耍且不会受伤，同时还要保证你女朋友新买的化妆镜不会被孩子打碎。

**Java 沙箱(sandbox)**负责保护一些系统资源，而且保护级别是不同的。

- 内部资源，如本地内存；
- 外部资源，如访问其文件系统或是在同一局域网的其他机器；
- 对于运行的组件（applet），可以访问其web服务器；
- 主机通过网络传输到磁盘的数据流。

一般来讲，沙箱的默认状态允许其中的程序访问CPU、内存等资源，以及其上装在的Web服务器。若沙箱完全开放，则其中程序的权限与主机相同。

当前最新的安全机制实现，则引入了域 (Domain) 的概念，可以理解为将沙箱细分为多个具体的小沙箱。虚拟机会把所有代码加载到不同的系统域和应用域，系统域部分专门负责与关键资源进行交互，而各个应用域部分则通过系统域的部分代理来对各种需要的资源进行访问。虚拟机中不同的受保护域 (Protected Domain)，对应不一样的权限 (Permission)。存在于不同域中的类文件就具有了当前域的全部权限，如下图所示：

![](https://user-gold-cdn.xitu.io/2018/8/7/16513234ccb06e9e?imageslim)

**sandbox 的实现**取决于下面三方面的内容：

- 安全管理器(SecurityManager)，利用其提供的机制，可以使 Java API 确定与安全相关的操作是否允许执行。
- 存取控制器(AccessController)，安全管理器默认实现的基础。
- 类装载器(ClassLoader)，可以实现安全策略和类的封装。

从 Java API 的角度去看，应用程序的安全策略是由安全管理器去管理的。安全管理器决定应用是否可以执行某项操作。这些操作具体是否可以执行的依据，是看其能否对一些比较重要的系统资源进行访问，而这项验证由存取控制器进行管控。这么看来，存取控制器是安全管理器的基础实现，安全管理器能做的，存取控制器也可以做。那么问题来了，为什么还需要安全管理器？

>Java2 以前是没有存取控制器的，那个时候安全管理器利用其内部逻辑决定应用的安全策略，若要调整安全策略，必须修改安全管理器本身。
>
>Java2 开始，安全管理器将这些工作交由存取控制器，存取控制器可以利用策略文件灵活地指定安全策略，同时还提供了一个更简单的方法，实现了更细粒度地将特定权限授予特定的类。
>
>因此，Java2 之前的程序都是利用安全管理器的接口实现系统安全的，这意味着安全管理器是不能修改的，那么引入的存取控制器并不能完全替代安全管理器。两者的关系如下图：

![](https://user-gold-cdn.xitu.io/2018/8/14/1653703994e6f17c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)


## 二、安全管理器(SecurityManager)

安全管理器是 Java API 和应用程序之间的“第三方权威机构”。好比贷款时，银行会根据央行征信系统查询用户的信用情况决定是否放款。Java 应用程序请求 Java API 完成某个操作，Java API会向安全管理器询问是否可以执行，安全管理器若不希望执行该操作，会抛一个异常给 Java API, 否则 Java API 将完成操作并正常返回。

### 2.1 初识 SecurityManager

SecurityManager 类是 Java API 中一个相当关键的类，它为其他 Java API 提供相应的接口，使之可以检查某项操作能否执行，充当了安全管理器的角色。我们从下面的代码来看安全管理器是如何工作的？

```java
public static void main(String[] args) {
  String s;
  try {
    FileReader fr = new FileReader(new File("E:\\test.txt"));
    BufferedReader br = new BufferedReader(fr);
    while ((s = br.readLine()) != null)
      System.out.println(s);
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```

第一步，在创建 FileReader 对象的时候会先根据 File 对象创建 FileInputStream 实例，源码如下：

```java
public FileInputStream(File file) throws FileNotFoundException {
  String name = (file != null ? file.getPath() : null);
  SecurityManager security = System.getSecurityManager();
  if (security != null) {
    security.checkRead(name);
  }
  if (name == null) {
    throw new NullPointerException();
  }
  if (file.isInvalid()) {
    throw new FileNotFoundException("Invalid file path");
  }
  fd = new FileDescriptor();
  fd.attach(this);
  path = name;
  open(name);
}
```

第二步，Java API 希望创建一个读取 File 的字节流对象，首先必须获取当前系统的安全管理器，然后通过安全管理器进行操作校验，若通过，再调用私有方法真正执行操作（open() 是FileInputStream 类的私有实例方法），若校验失败，则抛出一个安全异常，层层上抛，直至用户面前。

```java
public void checkRead(String file) {
  checkPermission(new FilePermission(file,SecurityConstants.FILE_READ_ACTION));
}

public void checkPermission(Permission perm) {
  java.security.AccessController.checkPermission(perm);
}
```

上面便是此处涉及 SecurityManager 的两个方法源码(jdk1.8)。可以看到，SecurityManager 对访问文件的校验，最终是交由存取控制器实现的，AccessController 在检查权限期间则会抛出一个 AccessControlException 异常告诉调用者校验失败。该异常继承自SecurityException，SecurityException 又继承自 RuntimeException，因此AccessControlException 是一个运行期异常。通常，调用方法往往涉及到一系列其他方法的调用，一旦出现安全异常，异常会顺着调用链传向顶部方法，最后线程中断结束。

### 2.2 操作 SecurityManager

一般情况下，安全管理器是默认没有被安装的。因此，上面创建 FileInputStream 的源码中，security == null, 是不会执行 checkRead 的（感兴趣的同学可以在main方法里直接使用System 提供的方法进行验证）。System 类为用户操作安全管理器提供了两个方法。

>`public static SecurityManager getSecurityManager()` 
>
>该方法用于获得当前安装的安全管理器引用，若未安装，返回 null。
>
>`public static void setSecurityManager(final SecurityManager s)`
>
>该方法用于将指定的安全管理器的实例设置为系统的安全管理器。

上面读取 test.txt 的代码时可以正常执行的，控制台会一行一行打印文件的内容。在配置上自定义的安全管理器（继承 SecurityManager，重写 checkRead 方法）后，再看执行结果。

```java
public class Main {

  class SecurityManagerImpl extends SecurityManager {

    public void checkRead(String file) {
      throw new SecurityException();
    }
  }

  public static void main(String[] args) {
    System.out.println("CurrentSecurityManager is " + System.getSecurityManager());
    Main m = new Main();
    System.setSecurityManager(m.new SecurityManagerImpl());
    System.out.println("CurrentSecurityManager is " + System.getSecurityManager());
    String s;
    try {
      FileReader fr = new FileReader(new File("E:\\test.txt"));
      BufferedReader br = new BufferedReader(fr);
      while ((s = br.readLine()) != null) {
        System.out.println(s);
      }
    }catch (IOException e) {
      e.printStackTrace();
    }
  }
}
```

执行结果：

```java
CurrentSecurityManager is null
CurrentSecurityManager is Main$SecurityManagerImpl@135fbaa4
Exception in thread "main" java.lang.SecurityException
	at Main$SecurityManagerImpl.checkRead(Main.java:10)
	at java.io.FileInputStream.<init>(FileInputStream.java:127)
	at java.io.FileReader.<init>(FileReader.java:72)
	at Main.main(Main.java:21)
```

注：如果想要 java 环境启用默认的管理器:

- 一种方式如上设置默认安全管理器的实例。
- 一种方式也可以在配置 JVM 运行参数的时候加上 `-Djava.security.manager`。

一般推荐后者，因为可以不用去改动代码，同时可以灵活的通过再配置一个`-Djava.security.policy="x:/xxx.policy"`参数的方式指定安全策略文件。

### 2.3 使用 SecurityManager

安全管理器提供了各个方面的安全检查的公共方法，允许任意调用。核心 Java API 中有很多方法，直接或间接调用安全管理器提供的方法实现各自的安全检查操作。在安全检查中还存在一个概念，可信类与不可信类。显然，一个类不是可信类就是不可信类。

> 如果一个类是核心 Java API 类，或者它显示地拥有执行某项操作的权限，那么这个类就是可信类。

#### 2.3.1 文件访问相关的安全检查方法

这里的文件访问指的是局域网中文件访问的处理，并不单单是本地磁盘上的文件访问。

> ```java
> public void checkRead(FileDescriptor fd)
> public void checkRead(String file)
> public void checkRead(String file, Object context)
> ```
>
> **检查程序能否读取指定文件**。不同入参代表不同的校验方式。第一个方法校验当前保护域是否拥有名为readFileDescriptor的运行时权限，第二个方法检验当前保护域是否拥有指定文件的读权限，第三个方法和第二个方法相同，不同的是在指定的存取控制器上下文中检验。

> ```java
> public void checkWrite(FileDescriptor fd)
> public void checkWrite(String file)
> ```
>
> **检查是否允许程序写指定文件**。第一个方法校验当前保护域是否拥有名为writeFileDescriptor的运行时权限，第二个方法检验当前保护域是否拥有指定文件的写权限。

> ```java
> public void checkDelete(String file)
> ```
>
> **检查是否允许程序删除指定文件**。检验当前保护域是否拥有指定文件的删除权限。

下表简单列出了 Java API 中直接调用了 checkRead()、checkWrite()和 checkDelete() 的方法。

![img](https://user-gold-cdn.xitu.io/2018/8/14/16537925baa1c91e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 2.3.2 网络访问相关的安全检查方法

Java中的网络访问一般是通过打开一个网络套接字实现的。网络套接字在逻辑上分为客户套接字和服务器套接字两类。

> ```java
> public void checkConnect(String host, int port)
> public void checkConnect(String host, int port, Object context)
> ```
>
> 检查程序能否向指定的主机上指定的端口打开一个客户套接字。检验当前保护域是否拥有指定主机名和端口的连接权限。

> ```java
> public void checkListen(int port)
> ```
>
> 检查程序能否创建一个监听特定端口的服务器套接字。

> ```java
> public void checkAccept(String host, int port)
> ```
>
> 检查程序能否在当前服务器套接字上接收指定主机和端口发出的客户连接。

> ```java
> public void checkMulticast(InetAddress maddr)
> ```
>
> 检查程序能否在指定的多播地址上创建一个多播套接字。

> ```java
> public void checkSetFactory()
> ```
>
> 检查程序能否修改默认的套接字实现。使用Socket创建套接字时，会由套接字工厂获得一个新的套接字。程序可以通过安装套接字工厂扩展不同语义的套接字，这就要求保护域拥有名为setFactory的运行时权限。

#### 2.3.3 保护Java虚拟机的安全检查方法

对于不可信类，有必要去提供一些方法避免它们绕过安全管理器和 Java API,从而保证 Java 虚拟机的安全。

> ```java
> public void checkCreateClassLoader()
> ```
>
> 检查当前保护域是否拥有creatClassLoader的运行时权限，确定程序能否创建一个类加载器。

> ```java
> public void checkExec(String cmd)
> ```
>
> 检查保护域是否拥有指定命令的执行权限，确定程序能否执行一个系统命令。

> ```java
> public void checkLink(String lib)
> ```
>
> 检查程序能否程序能否链入虚拟机中链接共享库(本地代码通过该库执行)。

> ```java
> public void checkExit(int status)
> ```
>
> 检查程序是否有权限关闭虚拟机。

> ```java
> public void checkPermission(Permission perm)
> public void checkPermission(Permission perm, Object context)
> ```
>
> 检查当前保护域（可以理解为当前线程）是否拥有指定的权限。

#### 2.3.4 保护程序线程的安全检查方法

一个Java程序的运行依赖于很多线程。除了程序本身的线程，虚拟机会自动为用户创建很多系统级的线程，比如垃圾回收、管理相关接口的输入输出请求等等。不可信类是不能管理这些影响程序的线程的。

> ```java
> public void checkAccess(Thread t)
> public void checkAccess(ThreadGroup g)
> ```
>
> 检查是否允许修改指定线程（线程组及组内线程）的状态。

#### 3.5 保护系统资源的安全检查方法

Java程序是可以访问一些系统级的资源的，比如打印任务、剪贴板、系统属性等等。出于安全考虑，不可信类是不能访问这些资源的。

> ```java
> public void checkPrintJobAccess()
> ```
>
> 检查程序能否访问用户打印机（queuePrintJob-运行时权限）

> ```java
> public void checkSystemClipboardAccess()
> ```
>
> 检查程序是否可以访问系统剪贴板（accessClipboard-AWT权限）

> ```java
> public void checkAwtEventQueueAccess()
> ```
>
> 检查程序能否获得系统时间队列（accessEventQueue-AWT权限）

> ```java
> public void checkPropertiesAccess()
> public void checkPropertyAccess(String key)
> ```
>
> 检查程序嫩否获取Java虚拟机拥有的系统属性信息

> ```java
> public boolean checkTopLevelWindow(Object window)
> ```
>
> 检查程序能否在桌面新建一个窗口

#### 3.6 保护Java 安全机制本身的安全检查方法

> ```java
> public void checkMemberAccess(Class<?> clazz, int which)
> ```
>
> 反射时检查程序能否访问类的成员。

> ```java
> public void checkSecurityAccess(String target)
> ```
>
> 检查程序能否执行安全有关的操作。

> ```java
> public void checkPackageAccess(String pkg)
> public void checkPackageDefinition(String pkg)
> ```
>
> 在使用类装载器装载某个类且指定了包名时，会检查程序能否访问指定包下的内容。

## 三、存取控制器

核心 Java API 由安全管理器提供安全策略，但是大多数安全管理器的实现是基于存取控制器的。

### 3.1 建立存取控制器的基础

- 代码源(CodeSource)：对于从其上装载Java类的地址，需要用代码源进行封装
- 权限(Permission)：要实现某个特定操作，需要权限封装相应的请求
- 策略(Policy)：对指定代码源授予相应的权限，策略可以表示为对所有权限的封装
- 保护域(ProtectionDomain)：对代码源及该代码源相应权限的封装

#### 3.1.1 CodeSource

代码源对象表示从其上装载的类的 URL 地址，以及类的签名相关信息，由类装载器负责创建和管理。

```java
public CodeSource(URL url, Certificate certs[])
```

构造器函数，针对指定url装载得到的代码，创建一个代码源对象。第二个参数是证书数组，可选，用来指定公开密钥，该密钥可以实现对代码的签名。

```java
public boolean implies(CodeSource codesource)
```

按照权限类的（Permission）的要求，判断当前代码源能否用来表示参数所指定的代码源。一个代码源能表示另一个代码源的条件是，前者必须包括后者的所有证书，而且由前者的URL可以获得后者地URL。

#### 3.1.2 Permission

Permission 类的实例对象就是权限对象，它是存取控制器处理的基本实体。Permission 类是一个抽象类，不同的实现类在安全策略文件中体现为不同的权限类型。Permission类的一个实例代表一个特定的权限，一组特定的权限则由Permissions的一个实例表示。

要实现自定义权限类的时候需要继承Permission类的，其抽象方法如下：

```java
//校验权限参数对象拥有的权限名和权限操作是否符合创建对象时的设置是否一致
public abstract boolean implies(Permission permission);
//比较两个权限对象的类型、权限名以及权限操作
public abstract boolean equals(Object obj);
public abstract int hashCode();
//返回创建对象时设置的权限操作，未设置返回空字符串
public abstract String getActions();
```

Java 本身包括了一些 Permission 类: 具体参考[官网](http://docs.oracle.com/javase/7/docs/technotes/guides/security/spec/security-spec.doc3.html#17001)

| class                               | desc                        |
| ----------------------------------- | --------------------------- |
| java.security.AllPermission         | 所有权限的集合              |
| java.util.PropertyPermission        | 系统/环境属性权限           |
| java.lang.RuntimePermission         | 运行时权限                  |
| java.net.SocketPermission           | Socket权限                  |
| java.io.FilePermission              | 文件权限,包括读写,删除,执行 |
| java.io.SerializablePermission      | 序列化权限                  |
| java.lang.reflect.ReflectPermission | 反射权限                    |
| java.security.UnresolvedPermission  | 未解析的权限                |
| java.net.NetPermission              | 网络权限                    |
| java.awt.AWTPermission              | AWT权限                     |
| java.sql.SQLPermission              | 数据库sql权限               |
| java.security.SecurityPermission    | 安全控制方面的权限          |
| java.util.logging.LoggingPermission | 日志控制权限                |
| javax.net.ssl.SSLPermission         | 安全连接权限                |
| javax.security.auth.AuthPermission  | 认证权限                    |
| javax.sound.sampled.AudioPermission | 音频系统资源的访问权限      |

#### 3.1.3 Policy

存取控制器需要确定权限应用于哪些代码源，从而为其提供相应的功能，这就是所谓的安全策略。

Java 使用了 Policy 对安全策略进行了封装，默认的安全策略类由sun.security.provider.PolicyFile 提供，该类基于jdk中配置的策略文件（%JAVA_HOME%/ jre/lib/security/java.policy）进行对特定代码源的权限配置。默认配置如下：

```java
// Standard extensions get all permissions by default

grant codeBase "file:${{java.ext.dirs}}/*" {
  permission java.security.AllPermission;
};

// default permissions granted to all domains

grant {
  // Allows any thread to stop itself using the java.lang.Thread.stop()
  // method that takes no argument.
  // Note that this permission is granted by default only to remain
  // backwards compatible.
  // It is strongly recommended that you either remove this permission
  // from this policy file or further restrict it to code sources
  // that you specify, because Thread.stop() is potentially unsafe.
  // See the API specification of java.lang.Thread.stop() for more
  // information.
  permission java.lang.RuntimePermission "stopThread";
  // allows anyone to listen on dynamic ports
  permission java.net.SocketPermission "localhost:0", "listen";

  // "standard" properies that can be read by anyone

  permission java.util.PropertyPermission "java.version", "read";
  permission java.util.PropertyPermission "java.vendor", "read";
  permission java.util.PropertyPermission "java.vendor.url", "read";
  permission java.util.PropertyPermission "java.class.version", "read";
  permission java.util.PropertyPermission "os.name", "read";
  permission java.util.PropertyPermission "os.version", "read";
  permission java.util.PropertyPermission "os.arch", "read";
  permission java.util.PropertyPermission "file.separator", "read";
  permission java.util.PropertyPermission "path.separator", "read";
  permission java.util.PropertyPermission "line.separator", "read";

  permission java.util.PropertyPermission "java.specification.version", "read";
  permission java.util.PropertyPermission "java.specification.vendor", "read";
  permission java.util.PropertyPermission "java.specification.name", "read";

  permission java.util.PropertyPermission "java.vm.specification.version", "read";
  permission java.util.PropertyPermission "java.vm.specification.vendor", "read";
  permission java.util.PropertyPermission "java.vm.specification.name", "read";
  permission java.util.PropertyPermission "java.vm.version", "read";
  permission java.util.PropertyPermission "java.vm.vendor", "read";
  permission java.util.PropertyPermission "java.vm.name", "read";
};
```

第一个 grant 定义了系统属性 ${{java.ext.dirs}} 路径下的所有的 class 及 jar(/*号表示所有 class 和 jar，如果只是 / 则表示所有 class 但不包括 jar)拥有所有的操作权限(java.security.AllPermission)，java.ext.dirs 对应路径为 %JAVA_HOME%/jre/lib/ext目录

第二个 grant 后面定义了所有 JAVA 程序都拥有的权限，包括停止线程、启动Socket 服务器、读取部分系统属性。

Policy 类提供 `addStaticPerms(PermissionCollection perms, PermissionCollection statics) `方法添加特定权限集给策略对象内部的权限集，也提供`public PermissionCollection getPermissions(CodeSource codesource)` 方法设置安全策略的权限集给来自特定代码源的类。

虚拟机中任何情况下只能安装一个安全策略类的实例，但是可以通过 `Policy.setPolicy(Policy p)` 替换当前系统的安全策略，也可以通过 `Policy.getPolicy()` 获得程序当前的安全策略类。

自定义 policy 文件语法可以参考[官网](http://docs.oracle.com/javase/7/docs/technotes/guides/security/PolicyFiles.html)

#### 3.1.4 ProtectionDomain

保护域就是一个授权项，可以理解为是代码源和对应权限的组合。虚拟机中每个类都属于且仅属于一个保护域，由代码源指定的地址装载得到，同时代码源所在保护域包含的权限集规定了一些权限，这个类就拥有这些权限。保护域的构造方法如下：

```java
public ProtectionDomain(CodeSource codesource,PermissionCollection permissions)
```

### 3.2 存取控制器(AccessController)

AccessController 类的构造器是私有的，因此不能对其进行实例化。它向外部提供了一些静态方法，其中最关键的就是`checkPermission(Permission p)`，该方法基于当前安装的 Policy 对象，判定当前保护欲是否拥有指定权限。安全管理器 SecurityManager 提供的一系列check***的方法，最后基本都是通过 `AccessController.checkPermission(Permission p)` 完成。

```java
public static void main(String[] args) {
  System.setSecurityManager(new SecurityManager());
  SocketPermission sp = new SocketPermission(
    "127.0.0.1:6000", "connect");
  try {
    AccessController.checkPermission(sp);
    System.out.println("Ok to open socket");
  } catch (AccessControlException ace) {
    System.out.println(ace);
  }
}
```

上面的代码首先安装了默认的安全管理器，然后实例化了一个连接本地6000端口的权限对象，最后通过存取控制器检查。 

打印结果如下：

```java
java.security.AccessControlException: access denied ("java.net.SocketPermission" "127.0.0.1:6000" "connect,resolve")
```

存取控制器抛出了一个异常，提示没有连接该地址的权限。在默认的安全策略文件上配置此端口的连接权限：

```java
permission java.net.SocketPermission "127.0.0.1:6000", "connect";
```

打印结果：

```java
Ok to open socket
```

实际工作中，可能会面临多个项目之间的方法调用。假设有两个项目 A 和 B，A 项目中的 TestA 类中有 testA() 方法内部调用了 B 项目中的 TestB 类的 testB() 方法，去打开一个项目 B 所在服务器的套接字。

在权限校验时，要想此种调用正常操作。需要在 A 项目所在虚拟机的安全策略文件中配置 TestA 类打开项目 B 所在服务器制定端口的连接权限，同时还要在 B 项目所在虚拟机的安全策略文件中配置 TestB 类打开同一地址及端口的连接权限。

这种操作方式固然可以，但是显然太复杂且不可预计。AccessController 提供了doPivileged()方法，为调用者临时开放权限，但是要求被调用者必须有对应操作的权限。