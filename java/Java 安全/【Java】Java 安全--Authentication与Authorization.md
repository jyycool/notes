# 【Java】Java 安全--Authentication与Authorization

## 前言

之前一篇文章[【Java】Java 安全--SecurityManager与AccessController](/Users/sherlock/Desktop/notes/java/[Java]Java 安全--SecurityManager与AccessController.md) 中我们详细介绍了 java sandbox 机制及其实现的三大要素中的两个: 安全管理器(SecurityManager)和存取控制器(AccessController)。

但基于 policy 策略只能控制行为, 无法辨识当前程序的访问者的身份, 因此如果要实现更加细粒度的控制程序的权限, 可以配合使用 Java 平台提供的认证与授权服务 JAAS(Java Authentication and Authorization Service), 它能够控制代码对敏感或关键资源的访问，例如文件系统，网络服务，系统属性访问等，加强代码的安全性。

JAAS 主要包含认证(Authentication)与授权(Authorization)两部分，认证的目的在于可靠安全地确定当前是谁在执行代码，代码可以是一个应用，applet，bean，servlet；授权的目的在于确定了当前执行代码的用户有什么权限，资源是否可以进行访问。

JAAS 实现了标准可插拔身份验证模块(PAM)框架的Java版本。它以可插入的方式执行, 这允许应用程序独立于底层身份验证技术，可以在应用程序下插入新的或更新的身份验证技术，而不需要修改应用程序本身。

应用程序通过实例化 `LoginContext` 对象来启用身份验证过程，而 `LoginContext` 对象又引用 `Configuration` 或 `LoginModule`，用于执行身份验证。典型的 `LoginModules` 可能会提示输入和验证用户名和密码

执行代码的用户或服务经过身份验证后，JAAS 授权组件将与核心 Java SE 访问控制模型一起工作，以保护对敏感资源的访问。访问控制决策既基于执行代码的 `CodeSource`，也基于运行代码的用户或服务，后者由 `Subject` 对象表示，如果身份验证成功，则使用相关 `Principals` 和`credentials` 的 `LoginModule` 更新 `Subject`。

## 一、核心类和接口

与 JAAS 相关的核心类和接口可以分为三类:

- Common
- Authentication
- Authorization

### 1.1 Common

公共类是由 JAAS 身份验证和授权组件共享的类。关键的 JAAS 类是 `javax.security.auth.Subject`，它表示单个实体(如人)的相关信息的分组。它包含实体的`Principals`、公共 `credentials` 和私有 `credentials`

#### Subject

如果要授权访问一些资源，需要先对**资源请求主体**进行认证。JAAS框架中，使用**Subject**来描述这个**资源请求主体**与安全访问相关的信息，因此，一个Subject通常是指一个对象实体，或者一个服务。Subject中所关联的信息，主要涉及：

- 身份信息
- 密码信息
- 加密密钥/凭据信息

一个Subject可能拥有一个或多个身份，一个身份被称之为**Principal**，也就是说，一个 Subject可能关联一个或多个Principals。

一个Subject可能涉及与安全有关的**凭据**信息(密钥/票据)，称之为**Credentials**。敏感的**Credentials**需要特别的保护措施，例如，私有密钥信息(**privCredentials**)，被保存在一个私钥集合中。而关于公有秘钥信息(**pubCredentials**)，则被保存在另外一个公钥集合中。

##### API

1. 构造方法

   ```java
   public Subject()
   public Subject(boolean readOnly, 			// 只读的 Subject
                  Set principals,				// 身份信息
                  Set pubCredentials, 		// 公有秘钥
                  Set privCredentials);	// 私有秘钥
   ```

   - 无参构造

     创建具有空 Principal 集以及空的公共和私有 Credential 集的 Subject 实例。

     无参构造方法生成的 Subject 默认是非只读的, 即: isReadOnly() 方法返回 false

   - 有参构造

     创建指定 Principal 集和 Credential 集的 Subject 实例。

   两个构造方法都会将指定集合中的 Principal 和 Credential 集复制到新构造的集合(SecureSet)中。新构造的集合(SecureSet)在允许后续修改之前会检查该 Subject 是否是只读的。

   新创建的集合还通过确保调用者拥有足够的权限来防止非法修改。

   - 要修改主体集，调用方必须具有 `AuthPermission("modifyPrincipals")`。
   - 要修改公共凭据集，调用方必须具有`AuthPermission("modifyPublicCredentials")`。
   - 要修改私有凭据集，调用者必须具有`AuthPermission("modifyPrivateCredentials")`。

   所以如果 Subject 是只读的，那么它的 Principal 和 Credential  集合是不可变的。

   如果应用程序实例化了一个 LoginContext，而没有将 Subject 传递给 LoginContext 构造函数，那么 LoginContext 实例化了一个新的空 Subject。开发人员不必实例化 subject。

   ```java
   public boolean isReadOnly();
   ```

2. Principal 集和 Credential 集

   - 获取与 Subject 相关的 Principal 集: 

     ```java
     // 返回 Subject 中的所有 principals
     public Set getPrincipals();
     // 返回指定类 c 的实例或类 c 的子类的实例化的 principals。
     public Set getPrincipals(Class c);
     
     // 上述两个方法的 subject 如果没有任何关联的 principals，则返回一个空集合。
     ```

   - 获取与 Subject 相关的 Credential 集的方式与上述方式类似:

     ```java
     public Set getPublicCredentials();
     public Set getPublicCredentials(Class c);
     public Set getPrivateCredentials();
     public Set getPrivateCredentials(Class c);
     ```

   - 添加 Principal 和 Credential(只有非只读的 Subject 可以这么做) 

     ```java
     Subject subject;
     Principal principal;
     Object credential;
     ......
     // add a Principal and credential to the Subject
     subject.getPrincipals().add(principal);
     subject.getPublicCredentials().add(credential);
     ```

     

3. readOnly 

   - 设置 readOnly 属性

     ```java
     public void setReadOnly()
     ```

     将该 Subject 设置为只读。将不允许对该 Subject 的 Principal集和 Credential 集进行修改(add and remove)。对这个 Subject 的 Credential 集进行销毁操(destroy)作仍然被允许。

     后续尝试修改 Subject 的 Principal集和 Credential 集将导致抛出IllegalStateException。

     `此外，一旦 Subject 是只读的，它就不能被重新设置为可写。`

   - 获取 readOnly 属性

     ```java
     public boolean isReadOnly()
     ```

   

4. getSubject(AccessControlContext)

   ```java
   public static Subject getSubject(AccessControlContext acc)
   ```

   获取与提供的 AccessControlContext 相关联的 Subject。AccessControlContext 可能包含许多 Subject(来自嵌套的doAs调用)。在这种情况下，返回与AccessControlContext 关联的最近的 Subject。

5. doAs

   ```java
   public static Object doAs(final Subject subject,
          	final java.security.PrivilegedAction action);
   
   public static Object doAs(final Subject subject,
          	final java.security.PrivilegedExceptionAction action)
     			throws java.security.PrivilegedActionException;
   ```

   作为一个特定的 Subject 进行工作。这个方法首先通过 AccessController#getContext 获取当前线程的 AccessControlContext，然后使用检索到的上下文和新的SubjectDomainCombiner(使用提供的 Subject 构造)实例化一个新的AccessControlContext。最后，该方法调用 AccessController。将提供的PrivilegedAction 以及新构造的 AccessControlContext传递给它。

   - 示例

     假设名为 "sher" 的人已经通过 `LoginContext` (请参阅`LoginContext`) 进行了身份验证, 因此，`Subject` 中填充了`com.ibm.security.Principal` 类的 `Principal`。这个 `Principal` 的名字叫 sher, 还假设已经安装了 SecurityManager，并且访问控制策略中存在以下内容:

     ```java
     // grant "sher" permission to read the file "foo.txt"
     grant Principal com.ibm.security.Principal "sher" {
       permission java.io.FilePermission "foo.txt", "read";
     };
     ```

     下面是示例应用程序代码

     ```java
     class ExampleAction implements java.security.PrivilegedAction {
       public Object run() {
         java.io.File f = new java.io.File("foo.txt");
     
         // the following call invokes a security check
         if (f.exists()) {
           System.out.println("File foo.txt exists");
         }
         return null;
       }
     }
     
     public class Example {
       public static void main(String[] args) {
     
         // Authenticate the subject, "sher".
         // This process is described in the
         // LoginContext class.
     
         Subject sher;
         // Set sher to the Subject created during the
         // authentication process
     
         // perform "ExampleAction" as "sher"
         Subject.doAs(sher, new ExampleAction());
       }
     }
     ```

     在执行过程中，ExampleAction调用f.exists()时将遇到安全检查。但是，由于ExampleAction 作为 "sher" 运行，并且策略(上面)将必要的文件权限授予 "sher"，因此 ExampleAction 将通过安全检查。如果策略中的 grant 语句被更改(例如，添加不正确的 CodeBase 或将主体更改为 "shery")，则会抛出SecurityException。

6. doAsPrivileged

   ```java
   public static <T> T doAsPrivileged(Subject subject,
   					PrivilegedAction<T> action,
          		AccessControlContext acc);
   
   public static <T> T doAsPrivileged(Subject subject,
            	PrivilegedExceptionAction<T> action,
            	AccessControlContext acc)
   	throws PrivilegedActionException
   ```

   doAsPrivilege 方法的行为与 doAs 方法完全相同，只是 doAsPrivilege 需要通过传入一个 AccessControlContext 而不是将提供的 subject 与当前线程的AccessControlContext 关联。通过这种方式，可以通过与当前上下文不同的AccessControlContext 来控制操作。

   AccessControlContext 包含自实例化 AccessControlContext 以来执行的所有代码的信息，包括代码位置和策略授予代码的权限。为了使访问控制检查成功，策略必须为 AccessControlContext 引用的每个代码项授予所需的权限。

   如果提供给 doAsPrivilege 的 AccessControlContext 为 null，则操作不受单独的AccessControlContext 的限制。比如在服务器环境中。服务器可以对多个传入请求进行身份验证，并为每个请求执行单独的 doAs 操作。要启动每个 doAs 操作，并且不受当前服务器 AccessControlContext 的限制，服务器可以调用 doAsPrivileged 并传入 null 的AccessControlContext。

#### Principal

java.security.Principal 是一个接口, 它代表了 Subject 的某个身份，而一个 Subject 可能有多个 principal 。例如，一个人可能有一个 `name Principal(“John Doe”)` 和一个 `SSN Principal(“123-45-6789”)`。

Principal 接口已知的实现类有: 

- [Identity](../../java/security/Identity.html)
- [IdentityScope](../../java/security/IdentityScope.html)
- [JMXPrincipal](../../javax/management/remote/JMXPrincipal.html)
- [KerberosPrincipal](../../javax/security/auth/kerberos/KerberosPrincipal.html)
- [Signer](../../java/security/Signer.html)
- [X500Principal](../../javax/security/auth/x500/X500Principal.html)

#### Credential

Subject 除了关联的 Principal 外，Subject 还可以拥有与安全性相关的属性，这些属性称为凭据(Credential)。凭据可能包含用于向新服务验证主题的信息。这些凭证包括密码、Kerberos票据和公钥证书。凭据还可能包含仅使主体能够执行某些活动的数据。例如，加密密钥表示使主体能够签名或加密数据的凭据。公共凭证类和私有凭证类不是核心 JAAS 类库的一部分。因此，任何类都可以表示凭证

公共凭证类和私有凭证类不是核心JAAS类库的一部分。然而，开发人员可以选择让他们的凭据类实现两个与凭据相关的接口:`Refreshable` 和 `Destroyable`。

###### Refreshable

javax.security.auth.Refreshable 接口提供了凭据刷新自身的功能。例如，具有特定时间限制生命周期的凭据可以实现此接口，以允许调用者在其有效刷新刷新。该接口有两种抽象方法:

- isCurrent()

  ```java
  boolean isCurrent();
  ```

  此方法确定凭据是当前的还是有效的

- refresh()

  ```java
   void refresh() throws RefreshFailedException;
  ```

  此方法更新或扩展凭据的有效性

###### Destroyable

javax.security.auth.Destroyable 接口提供了销毁凭据内内容的功能。该接口有两个抽象方法:

- isDestroyed()

  确定凭据是否已被销毁

  ```java
  boolean isDestroyed();
  ```

- destroy()

  销毁并清除与此凭据关联的信息。对该凭证上的某些方法的后续调用将导致抛出IllegalStateException

  ```java
  void destroy() throws DestroyFailedException;
  ```

### 1.2 Authentication

认证是验证 `Subject` 身份的过程，必须以安全的方式执行; 否则，犯罪者可能会冒充他人来访问系统。身份验证通常涉及到 `Subject` 展示某种形式的证据来证明其身份。这些证据可能是只有受试者可能知道或拥有的信息(例如密码或指纹)，也可能是只有受试者能够生成的信息(例如使用私钥签名的数据)。

要对 `Subject`(用户或服务)进行身份验证，需要执行以下步骤:

1. 应用程序实例化 LoginContext, 一旦调用者实例化了 LoginContext，它就会调用 login 方法来验证主题。
2. LoginContext 查询一个 Configuration，以加载为该应用程序配置的所有LoginModules 。
3. 应用程序调用 LoginContext 的 login 方法
4. login 方法调用所有加载的 LoginModules 。每个 LoginModule 都尝试对 Subject进行身份验证, 成功后, LoginModules 将相关 Principals 和 Credentials 与表示正在验证的主题的 Subject 对象关联。
5. LoginContext 将身份验证状态返回给应用程序
6. 如果身份验证成功，应用程序将从 LoginContext 获取 Subject。

> 从执行流程中我们可以发现, 并不需要开发人员实例化 Subject, 因为对 Principals 和Credentials 验证通过后, Principals 和 Credentials 会被自动关联到 LoginContext 中的 Subject 中。

#### LoginContext

LoginContext 类描述了用于验证 Subject 的基本方法，并提供了一种独立于底层身份验证技术的开发应用程序的方法。配置指定在特定应用程序中使用的身份验证技术或 LoginModule。可以在应用程序下插入不同的 LoginModule，而不需要对应用程序本身进行任何修改。

除了支持可插入的身份验证之外，这个类还支持堆叠身份验证的概念。应用程序可以配置为使用多个LoginModule。例如，可以在应用程序下配置 Kerberos LoginModule 和 智能卡LoginModule。

典型的实例化方式就是使用 name 和 CallbackHandler 实例化 LoginContext。LoginContext 使用 name 作为配置的索引，以确定应该使用哪些 LoginModule，以及为了使整个身份验证成功，必须成功使用哪些 LoginModule。CallbackHandler 被传递给底层的LoginModule，这样它们就可以与用户进行通信和交互(例如，通过图形用户界面提示输入用户名和密码)。

一旦调用者实例化了 LoginContext，它就会调用 login 方法来验证主题。login 方法调用已配置的模块执行各自类型的身份验证(用户名/密码、智能卡pin验证、Kerberos等)。注意，如果身份验证失败，LoginModule 将不会尝试身份验证重试，也不会引入延迟。这些任务属于LoginContext 调用者。

如果登录方法返回而不抛出异常，则整个身份验证成功。然后，调用者可以通过调用getSubject方法检索新认证的主题。可以通过调用主题各自的getprincipal、getPublicCredentials和getPrivateCredentials方法来检索与主题相关联的主体和凭证。

为了登出主题，调用者调用登出方法。与登录方法一样，此注销方法调用已配置模块的注销方法。

LoginContext不应该用于验证多个主题。应该使用单独的LoginContext对每个不同的主题进行身份验证。

`LoginContext`提供了四个构造函数可供选择:

```java
public LoginContext(String name) throws LoginException;

public LoginContext(String name, 
                    Subject subject) 
  					throws LoginException;

public LoginContext(String name, 
                    CallbackHandler callbackHandler)
  					throws LoginException;

public LoginContext(String name, 
                    Subject subject,
         						CallbackHandler callbackHandler) 
  					throws LoginException
```

所有构造函数都共享一个公共参数:`name`。`LoginContext`使用这个参数作为登录配置的索引，以确定为实例化`LoginContext`的应用程序配置了哪些`LoginModule`
 不接受Subject作为输入参数的构造函数实例化一个新Subject。
 实际身份验证通过调用以下方法进行:



```java
 public void login() throws LoginException;
```

当调用login时，将调用所有配置的loginmodule来执行身份验证。如果认证成功，可以使用以下方法检索Subject(现在可以保存主体、公共凭证和私有凭证):



```cpp
 public Subject getSubject();
```

要注销`Subject`并删除其经过身份验证的主体和凭据，提供以下方法:



```java
public void logout() throws LoginException;
```

下面的代码示例演示了验证和注销主题所需的调用:



```csharp
 // let the LoginContext instantiate a new Subject
    LoginContext lc = new LoginContext("entryFoo");
    try {
        // authenticate the Subject
        lc.login();
        System.out.println("authentication successful");

        // get the authenticated Subject
        Subject subject = lc.getSubject();

        ...

        // all finished -- logout
        lc.logout();
    } catch (LoginException le) {
        System.err.println("authentication unsuccessful: " +
            le.getMessage());
    }
```

#### LoginModule

LoginModule 是 Java 提供的身份验证技术的标准接口(类似于JDBC)。不同的验证服务在应用程序下插入 LoginModule 的实现就可以提供特定类型的身份验证服务。

当应用程序写入 LoginContext API 时，身份验证技术提供者实现 LoginModule 接口。配置指定与特定登录应用程序一起使用的 LoginModule。因此，可以在应用程序下插入不同的LoginModule，而不需要对应用程序本身进行任何修改。

`LoginContext 负责读取配置并实例化适当的 LoginModule。`每个 LoginModule 都使用一个 Subject、一个 CallbackHandler、共享的 LoginModule 状态和特定的 LoginModule 的选项进行初始化。Subject 表示当前正在进行身份验证的主体，如果身份验证成功，则使用相关的凭据更新该主体。LoginModules 使用 CallbackHandler 与用户通信。例如，CallbackHandler可以用来提示输入用户名和密码。注意，CallbackHandler 可能是空的。LoginModules 绝对需要一个 CallbackHandler 来验证主题，它可能会抛出一个 LoginException。LoginModules可以选择使用共享状态在它们自己之间共享信息或数据。

特定于 LoginModule 的选项表示管理员或用户在登录配置中为这个 LoginModule 配置的选项。这些选项由 LoginModule 本身定义，并控制其中的行为。例如，LoginModule 可以定义支持调试/测试功能的选项。选项使用 key-value 语法定义，例如 debug=true。LoginModule 将选项存储为映射，以便可以使用键检索值。注意，LoginModule 选择定义的选项数量没有限制。

调用应用程序将身份验证过程视为单个操作。但是，LoginModule 中的身份验证过程分为两个不同的阶段。

- 在第一阶段，LoginModule 的登录方法由 LoginContext 的登录方法调用。然后，LoginModule 的登录方法执行实际的身份验证(例如提示并验证密码)，并将其身份验证状态保存为私有状态信息。一旦完成，LoginModule 的登录方法将返回 true(如果成功)或 false(如果应该忽略它)，或者抛出一个 LoginException 来指定失败。在失败的情况下，LoginModule 绝不能重试身份验证或引入延迟。这些任务的责任属于应用程序。如果应用程序尝试重试身份验证，将再次调用 LoginModule 的 login 方法。

- 在第二阶段，如果 LoginContext 的整体身份验证成功(相关的 REQUIRED, REQUISITE, SUFFICIENT 和 OPTIONAL 的 LoginModule 成功)，那么将调用 LoginModule 的提交方法。LoginModule 的 commit 方法检查其私有保存的状态，以查看自己的身份验证是否成功。如果整个 LoginContext 身份验证成功，并且 LoginModule 自己的身份验证成功，那么 commit 方法将相关 Principal (经过身份验证的身份验证)和 Credential (身份验证数据，如加密密钥)与位于 LoginModule 中的 Subject 关联起来。

如果 LoginContext 的整体身份验证失败(相关的REQUIRED、REQUISITE、SUFFICIENT 和OPTIONAL LoginModule 没有成功)，那么将调用每个 LoginModule 的 abort 方法。在这种情况下，LoginModule 将删除/销毁最初保存的任何身份验证状态。

注销一个 Subject 只涉及一个阶段。LoginContext 调用 LoginModule 的注销方法。LoginModule 的注销方法然后执行注销过程，例如从 Subject  中删除 Principal 或Credential 或删除记录的会话信息。

LoginModule 实现必须有一个不带参数的构造函数。这允许加载 LoginModule 的类实例化它。

一些典型的实现模块包括：

- `Krb5LoginModule` 基于Kerberos的登录/认证服务模块
- `JndiLoginModule` 基于用户名和密码的登录/认证服务模块
- `KeyStoreLoginModule` 基于KeyStore的登录/认证服务模块

LoginModule中涉及到两个主要方法为 login 与 commit:

- login

  对Subject进行认证。

  这个过程中主要涉及到用户名和密码信息提示，校验用户密码。认证结果将会在LoginModule层面暂时保存。

- commit

  Commit过程首先确认LoginModule中保存的认证结果。认证成功之后，Commit方法将Subject内对应的Principals以及Credentials关联起来。如果Login方法认证失败的话，则该方法将会清理在LoginModule中保存的认证结果信息。

#### CallbackHandler

在某些情况下，`LoginModule`必须与用户通信才能获得身份验证信息[javax.security.auth.callback.CallbackHandler](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.oracle.com%2Fen%2Fjava%2Fjavase%2F11%2Fdocs%2Fapi%2Fjava.base%2Fjavax%2Fsecurity%2Fauth%2Fcallback%2FCallbackHandler.html)就是用于此目的。应用程序实现`CallbackHandler`接口并将其传递给`LoginContext`，后者将其直接转发给底层`LoginModule`
 `LoginModule`使用`CallbackHandler`收集用户的输入(如密码或智能卡密码)或向用户提供信息(如状态信息),通过允许应用程序指定`CallbackHandler`，底层`LoginModule`可以实现不同的应用程序与用户交互方式。例如，GUI应用程序的CallbackHandler实现可能会显示一个窗口来请求用户输入。另一方面，非gui工具的`CallbackHandler`实现可能只是直接从命令行提示用户输入。

`CallbackHandler`是一个接口，有一个方法可以实现:



```css
 void handle(Callback[] callbacks)
         throws java.io.IOException, UnsupportedCallbackException;
```

`LoginModule`向`CallbackHandler` 的`handle`方法传递一系列适当的回调函数，例如用户名的`NameCallback`和密码的`PasswordCallback`, `CallbackHandler`执行请求的用户交互并在回调函数中设置适当的值。例如，要处理`NameCallback`, `CallbackHandler`可能会提示输入名称，然后获取输入`name`，并调用`NameCallback`的`setName`方法来存储`name`。

#### Callback

`javax.security.auth.callback`包包含回调接口和几个实现。LoginModules可以将一个回调数组直接传递给`CallbackHandler`的`handle`方法。

