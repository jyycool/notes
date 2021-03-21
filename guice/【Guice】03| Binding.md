# 【Guice】03| Binding

`Guice  injector` 的工作是创建对象组装视图。您请求需要类型的实例，它确定要构建的内容，解决依赖关系并将所有内容连接在一起。要指定如何解决依赖关系，需要在 `inject` 中配置绑定。

## Creating Bindings

要创建绑定，需要 `extends AbstractModule` 并覆盖其 `configure` 方法。在方法主体中，调用 `bind()` 以指定每个绑定。这些方法经过类型检查，因此如果使用错误的类型，则编译器可以报告错误。创建模块后，请将其作为参数传递 `Guice.createInjector()` 给构建注入器。

常见的绑定有

- [linked bindings](https://github.com/google/guice/wiki/LinkedBindings)

- [instance bindings](https://github.com/google/guice/wiki/InstanceBindings)
- [@Provides methods](https://github.com/google/guice/wiki/ProvidesMethods)
- [provider bindings](https://github.com/google/guice/wiki/ProviderBindings)
- [constructor bindings](https://github.com/google/guice/wiki/ToConstructorBindings)
- [untargetted bindings](https://github.com/google/guice/wiki/UntargettedBindings)



### 1. linked bindings

`linked bindings` 将类型映射到其实现。此示例将接口 `TransactionLog` 映射到实现 `DatabaseTransactionLog`：

```java
public class BillingModule extends AbstractModule {
  @Provides
  TransactionLog provideTransactionLog(
    DatabaseTransactionLog impl){
    
   return impl;
  }
}
```

现在，当您调用 `injector.getInstance(TransactionLog.class)` 或注入器遇到 `TransactionLog` 依赖项时，它将使用 `DatabaseTransactionLog`。`linkded bindings` 将一个超类连接到其子类型，例如实现类或扩展类。您甚至可以将 `DatabaseTransactionLog`类链接到其子类：

```java
bind(DatabaseTransactionLog.class).to(MySqlDatabaseTransactionLog.class);
```

`Linked bindings` 还可以链式传递

```java
public class BillingModule extends AbstractModule {
  @Provides
  TransactionLog provideTransactionLog(
    DatabaseTransactionLog databaseTransactionLog) {
   return databaseTransactionLog;
  }

  @Provides
  DatabaseTransactionLog provideDatabaseTransactionLog(
    MySqlDatabaseTransactionLog impl) {
   return impl;
  }
}
```

在这种情况下, 当需要一个 `TransactionLog`，`inject` 将返回一个  `MySqlDatabaseTransactionLog`。



### 2. Binding Annotations

有时，你有需要针对同一类型的多个绑定。例如，您可能同时需要 PayPal 信用卡处理器和 Google Checkout 处理器。为此，支持可选的注释绑定。注释和类型一起标识唯一的绑定。`Guice` 中称为 `Key`。

#### Defining binding annotations

绑定注释是用元注释`@Qualifier` 或 `@BindingAnnotation`。

定义绑定批注需要两行代码以及若干导入。将其放在自己的`.java`文件中或在其注释的类型内。

```java
package example.pizza;

import static java.lang.annotation.ElementType.PARAMETER;
import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

import java.lang.annotation.Target;
import java.lang.annotation.Retention;
import javax.inject.Qualifier;

@Qualifier
@Target({ FIELD, PARAMETER, METHOD })
@Retention(RUNTIME)
public @interface PayPal {}

// Older code may still use Guice `BindingAnnotation` in place of the standard
// `@Qualifier` annotation. New code should use `@Qualifier` instead.
@BindingAnnotation
@Target({ FIELD, PARAMETER, METHOD })
@Retention(RUNTIME)
public @interface GoogleCheckout {}
```

您不需要了解所有这些元注释，但如果您感到好奇：

- `@Qualifier `是一个 [JSR-330](https://github.com/google/guice/wiki/JSR330) 元注释，它告诉 Guice 这是一个绑定注释。如果将多个绑定注释应用于同一成员，Guice将产生一个错误。Guice `@BindingAnnotation`在旧代码中也用于此目的。

- `@Target({FIELD, PARAMETER, METHOD}) `它指明了该注释可以用于哪些地方，例如字段, 方法, 参数上......如果不指定, 则默认可用于所有地方。

- `@Retention(RUNTIME)`  指定注释的生命周期，这在Guice中是必需的。这里的 `RUNTIME` 指的是运行时该注释依然生效。 

要依赖于带注释的绑定，请将注释应用于注入的参数：

```java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@PayPal CreditCardProcessor processor, TransactionLog transactionLog) {
    ...
  }
}
```

最后，我们创建一个使用注释的绑定：

```java
final class CreditCardProcessorModule extends AbstractModule {
  @Override
  protected void configure() {
    // This uses the optional `annotatedWith` clause in the `bind()` statement
    bind(CreditCardProcessor.class)
        .annotatedWith(PayPal.class)
        .to(PayPalCreditCardProcessor.class);
  }

  // This uses binding annotation with a @Provides method
  @Provides
  @GoogleCheckout
  CreditCardProcessor provideCheckoutProcessor(
      CheckoutCreditCardProcessor processor) {
    return processor;
  }
}
```

#### @Named

Guice 有带有 `@Named` 字符串的内置绑定注释：

```java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(
    	@Named("Checkout") CreditCardProcessor processor,
      TransactionLog transactionLog) {
    
    // ...
  }
)
```

要绑定特定名称，请使用`Names.named()` 来创建实例传递给 `annotatedWith`：

```java
final class CreditCardProcessorModule extends AbstractModule {
  @Provides
  @Named("Checkout")
  CreditCardProcessor provideCheckoutProcessor(
      CheckoutCreditCardProcessor processor) {
    return processor;
  }
}
```

除了上述的创建 `Provider` 的方式,还有一种使用 `bind` 的方式

```java
bind(CreditCardProcessor.class)
  .annotatedWith(Names.named("Checkout"))
  .to(CheckoutCreditCardProcessor.class);
```



### 3. Instance Bindings

您可以将类型绑定到该类型的特定实例。这通常仅对没有自己依赖项的对象（例如值对象）有用：

```java
bind(String.class)
  .annotatedWith(Names.named("JDBC URL"))
  .toInstance("jdbc:mysql://localhost/pizza");

bind(Integer.class)
	.annotatedWith(Names.named("login timeout seconds"))
	.toInstance(10);
```

避免 `.toInstance` 与创建复杂的对象一起使用，因为它会减慢应用程序的启动速度。您可以改用一种`@Provides`方法来代替。

您还可以使用 `bindConstant` 方式绑定常量：

```java
bindConstant()
	.annotatedWith(HttpPort.class)
	.to(8080);
```

`bindConstant` 是绑定基本类型和其他常量类型（如`String`enum和`Class`。

### 4. @Provides methods

需要使用代码创建对象时，请使用`@Provides`方法。该方法必须在 `Module` 的子类中定义，并且必须具有`@Provides`注释。该方法的 **返回值类型** 是绑定类型。任何时候只要 `injector` 需要该类型的实例，它将调用该方法。

```java
public class BillingModule extends AbstractModule {
 
  @Override
  protected void configure() {
    ...
  }

  @Provides
  TransactionLog provideTransactionLog() {
    DatabaseTransactionLog transactionLog = 
      new DatabaseTransactionLog();
    transactionLog
      .setJdbcUrl("jdbc:mysql://localhost/pizza");
    transactionLog
      .setThreadPoolSize(30);
    return transactionLog;
  }
}
```

#### With Binding Annotation

如果该 `@Provides` 方法具有诸如 `@PayPal` 或 `@Named("Checkout")` 的绑定注释，则 Guice 会绑定该注释类型。依赖关系可以作为参数传递给方法。在调用该方法之前，`injector` 将对每个绑定进行绑定。

```java
@Provides @PayPal
CreditCardProcessor providePayPalCreditCardProcessor(
@Named("PayPal API key") String apiKey) {
  
	PayPalCreditCardProcessor processor = 
			new PayPalCreditCardProcessor();
	processor.setApiKey(apiKey);
	return processor;
}
```

#### Throwing Exceptions

Guice不允许从提供者抛出异常。`@Provides `方法抛出的异常将包装在`ProvisionException `中。允许从 `@Provides` 方法中抛出任何类型的异常（运行时或检查）是一种不好的做法。如果由于某种原因需要引发异常，则可能需要使用 [ThrowingProviders extension](https://github.com/google/guice/wiki/ThrowingProviders) `@CheckedProvides`方法。



### 5. Provider Bindings

当您的 `@Provides`方法开始变得复杂时，您可以考虑将其移至自己的类中。provider 类实现Guice的 `Provider`  接口，这是一个用于提供值的简单通用接口：

```java
public interface Provider<T> {
  T get();
}
```

我们提供程序的实现类具有自己的依赖关系，它通过带 `@Inject` 注释的构造函数接收。它实现了`Provider`接口，以定义具有安全性的返回值类型：

```java
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  private final Connection connection;

  @Inject
  public DatabaseTransactionLogProvider(
    Connection connection) {
    
    this.connection = connection;
  }
	
  @Override
  public TransactionLog get() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setConnection(connection);
    return transactionLog;
  }
}
```

最后，我们使用以下 `.toProvider` 子句来绑定自定义的 Provider 实现类：

```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class)
       .toProvider(DatabaseTransactionLogProvider.class);
  }
}
```

如果您的 Provider 很复杂，请务必对其进行测试！

#### Throwing Exceptions

Guice不允许从提供者抛出异常。`@Provides `方法抛出的异常将包装在`ProvisionException `中。允许从 `@Provides` 方法中抛出任何类型的异常（运行时或检查）是一种不好的做法。如果由于某种原因需要引发异常，则可能需要使用 [ThrowingProviders extension](https://github.com/google/guice/wiki/ThrowingProviders) `@CheckedProvides`方法。



### 6. Untargeted Bindings

您可以在不指定目标的情况下创建绑定。这对于用`@ImplementedBy`或 `@ProvidedBy` 的具体类和类型最有用。非目标绑定会通知注入程序有关类型的信息，因此它可能会急切地准备依赖关系。非目标绑定没有*to*子句，如下所示：

```java
bind(MyConcreteClass.class);
bind(AnotherConcreteClass.class).in(Singleton.class);
```

指定 `binding annotations` 时，即使它是相同的具体类，也仍然必须添加目标绑定。例如：

```java
bind(MyConcreteClass.class)
	.annotatedWith(Names.named("foo"))
  .to(MyConcreteClass.class);
  
bind(AnotherConcreteClass.class)
  .annotatedWith(Names.named("foo"))
  .to(AnotherConcreteClass.class)
  .in(Singleton.class);
```

非目标绑定通常用于在 `Module` 中注册类型，以确保 Guice 知道该类型。禁用  [JIT](https://github.com/google/guice/wiki/JustInTimeBindings) 时，必须注册这些类型，才能使它们有资格注入。

### 7. Constructor Bindings