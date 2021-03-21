## Drools7规则引擎

### 一、Drools简介

#### 1.1 什么是规则引擎

规则引擎是由推理引擎发展而来，是一种嵌入在应用程序中的组件，实现了将业务决策 从应用程序代码中分离出来，并使用预定义的语义模块编写业务决策。接受数据输入，解释 业务规则，并根据业务规则做出业务决策。

大多数规则引擎都支持规则的次序和规则冲突检验，支持简单脚本语言的规则实现，支 持通用开发语言的嵌入开发。目前业内有多个规则引擎可供使用，其中包括商业和开放源码 选择。开源的代表是 Drools，商业的代表是 Visual Rules ,I Log。

#### 1.2 Drools规则引擎

Drools（JBoss Rules ）具有一个易于访问企业策略、易于调整以及易于管理的开源业 务规则引擎，符合业内标准，速度快、效率高。业务分析师或审核人员可以利用它轻松查看 业务规则，从而检验是否已编码的规则执行了所需的业务规则。

JBoss Rules 的前身是 Codehaus 的一个开源项目叫 Drools。现在被纳入 JBoss 门下，更 名为 JBoss Rules，成为了 JBoss 应用服务器的规则引擎。

Drools 是为 Java 量身定制的基于 Charles Forgy 的 RETE 算法的规则引擎的实现。具有 了 OO 接口的 RETE,使得商业规则有了更自然的表达。

#### 1.3 Drools使用概览

Drools 是 Java 编写的一款开源规则引擎，实现了 Rete 算法对所编写的规则求值，支持 声明方式表达业务逻辑。使用 DSL(Domain Specific Language)语言来编写业务规则，使得规 则通俗易懂，便于学习理解。支持 Java 代码直接嵌入到规则文件中。

Drools 主要分为两个部分：一是 Drools 规则，二是 Drools 规则的解释执行。规则的编 译与运行要通过 Drools 提供的相关 API 来实现。而这些 API 总体上游可分为三类：规则编 译、规则收集和规则的执行。

Drools 是业务规则管理系统（BRMS）解决方案，涉及以下项目：

- Drools Workbench：业务规则管理系统
- Drools Expert：业务规则引擎
- Drools Fusion：事件处理
- JBPM：工作流引擎
- OptaPlanner：规划引擎

#### 1.4 Drools版本信息

目前 Drools 发布的最新版本为 7.0.0.Final，其他版本正在研发过程中。官方表示后 续版本会加快迭代速度。本系列也是基于此版本进行讲解。

**从 Drools6.x 到 7 版本发生重大的变化项：**

- @PropertyReactive不需要再配置，在Drools7中作为默认配置项。同时向下兼容。
- Drools6版本中执行sum方法计算结果数据类型问题修正。
- 重命名TimedRuleExecutionOption。
- 重命名和统一配置文件。

**Drools7 新功能：**

- 支持多线程执行规则引擎，默认为开启，处于试验阶段。
- OOPath改进，处于试验阶段。
- OOPath Maven插件支持。
- 事件的软过期。
- 规则单元RuleUnit。

### 二、Drools7

#### 2.1 API解析

逐步了解一下使用到的 API 的使用说明及相关概念性知识。

##### 2.1.1 什么是KIE

KIE（Knowledge Is Everything），知识就是一切的简称。JBoss 一系列项目的总称，在《Drools 使用概述》章节已经介绍了 KIE 包含的大部分项目。它们之间有一定的关联，通用一些 API。 比如涉及到构建（building）、部署（deploying）和加载（loading）等方面都会以 KIE 作为前 缀来表示这些是通用的 API。

##### 2.1.2 KIE生命周期

无论是 Drools 还是 JBPM，生命周期都包含以下部分：

- 编写：编写规则文件，比如：DRL，BPMN2、决策表、实体类等。
- 构建：构建一个可以发布部署的组件，对于KIE来说是JAR文件。
- 测试：部署之前对规则进行测试。
- 部署：利用Maven仓库将jar部署到应用程序。
- 使用：程序加载jar文件，通过KieContainer对其进行解析创建KieSession。
- 执行：通过KieSession对象的API与Drools引擎进行交互，执行规则。
- 交互：用户通过命令行或者UI与引擎进行交互。
- 管理：管理KieSession或者KieContainer对象。

##### 2.1.3 FACT对象

Fact 对象是指在使用 Drools 规则时，将一个普通的 JavaBean 对象插入到规则引擎的 WorkingMemory 当中的对象。规则可以对 Fact 对象进行任意的读写操作。Fact 对象不是对 原来的 JavaBean 对象进行 Clone，而是使用传入的 JavaBean 对象的引用。规则在进行计算 时需要的应用系统数据设置在 Fact 对象当中，这样规则就可以通过对 Fact 对象数据的读写 实现对应用数据的读写操作。

Fact 对象通常是一个具有 getter 和 setter 方法的 POJO 对象，通过 getter 和 setter 方法 可以方便的实现对 Fact 对象的读写操作，***<u>所以我们可以简单的把 Fact 对象理解为规则与 应用系统数据交互的桥梁或通道。</u>***

当 Fact 对象插入到 WorkingMemory 当中后，会与当前 WorkingMemory 当中所有的规 则进行匹配，同时返回一个 FactHandler 对象。FactHandler 对象是插入到 WorkingMemory 当中 Fact 对象的引用句柄，通过 FactHandler 对象可以实现对 Fact 对象的删除及修改等操 作。

前面的实例中通过调用 insert 方法将 Product 对象插入到 WorkingMemory 当中，Product 对象插入到规则中之后就是所谓的 FACT 对象。***<u>如果需要插入多个 FACT 对象，多次调用 insert 方法，并传入对应 FACT 对象即可。</u>***

##### 2.1.4 KieServices

该接口提供了很多方法，可以通过这些方法访问 KIE 关于构建和运行的相关对象，比如 说可以获取 KieContainer，利用 KieContainer 来访问 KBase 和 KSession 等信息；可以获取 KieRepository 对象，利用 KieRepository 来管理 KieModule 等。

KieServices 就是一个中心，通过它来获取的各种对象来完成规则构建、管理和执行等操 作。

```java
// 通过单例创建 KieServices
KieServices kieServices = KieServices.Factory.get();

// 获取 KieContainer
KieContainer kieContainer = kieServices.getKieClasspathContainer();

// 获取 KieRepository
KieRepository kieRepository = kieServices.getRepository();
```

![KieServices](/Users/sherlock/Desktop/notes/allPics/Drools/KieServices.png)

##### 2.1.5 KieContainer

可以理解 KieContainer 就是一个 KieBase 的容器。提供了获取 KieBase 的方法和创建 KieSession 的方法。***<u>其中获取 KieSession 的方法内部依旧通过 KieBase 来创建 KieSession。</u>***

```java
// 通过单例创建 KieServices
KieServices kieServices = KieServices.Factory.get();

// 获取 KieContainer
KieContainer kieContainer = kieServices.getKieClasspathContainer();

// 获取 KieBase
KieBase kieBase = kieContainer.getKieBase();

// 创建 KieSession
KieSession kieSession = kieContainer.newKieSession("session-name");
```

![KieContainer](/Users/sherlock/Desktop/notes/allPics/Drools/KieContainer.png)

##### 2.1.6 KieBase

KieBase 就是一个知识仓库，包含了若干的规则、流程、方法等，在 Drools 中主要就是规则和方法，KieBase 本身并不包含运行时的数据之类的，如果需要执行规则 KieBase 中的规则的话，就需要根据 KieBase 创建 KieSession。

```java
// 获取 KieBase
KieBase kieBase = kieContainer.getKieBase();
KieSession kieSession = kieBase.newKieSession();
StatelessKieSession statelessKieSession = kieBase.newStatelessKieSession();
```

![KieBase](/Users/sherlock/Desktop/notes/allPics/Drools/KieBase.png)

##### 2.1.7 KieSession

KieSession 就是一个跟Drools引擎打交道的会话，其基于 KieBase 创建，***<u>它会包含运行时数据， 包含“Fact”，</u>*** 并对运行时数据 ***<u>实时</u>*** 进行规则运算。通过 KieContainer 创建 KieSession 是一种较为方便的做法，其本质上是从 KieBase 中创建出来的。KieSession 就是 应用程序跟规则引擎进行交互的会话通道。

***<u>创建 KieBase 是一个成本非常高的事情</u>***，KieBase会建立知识(规则、流程)仓库，而***<u>创建 KieSession 则是一个成本非常低的事情</u>***，***<u>所以 KieBase 会建立缓存，而 KieSession 则不必。</u>***

![KieSession](/Users/sherlock/Desktop/notes/allPics/Drools/KieSession.png)

##### 2.1.8 KieRepository

KieRepository 是一个单例对象，它是存放 KieModule 的仓库，KieModule 由 kmodule.xml 文件定义（当然不仅仅只是用它来定义）。

```java
// 通过单例创建 KieServices
KieServices kieServices = KieServices.Factory.get();

// 获取 KieRepository
KieRepository kieRepository = kieServices.getRepository();
```

![KieRepository](/Users/sherlock/Desktop/notes/allPics/Drools/KieRepository.png)

##### 2.1.9 KieProject

KieContainer 通过 KieProject 来初始化、构造 KieModule ， 并将 KieModule 存放到 KieRepository 中，然后 KieContainer 可以通过 KieProject 来查找 KieModule 定义的信息，并 根据这些信息构造 KieBase 和 KieSession。

##### 2.1.10 ClasspathKieProject

ClasspathKieProject实现了KieProject 接口 ， 它提供了根据类路径中的`META-INF/kmodule.xml`文件构造 KieModule 的能力，是基于 Maven 构造 Drools 组件的基本保障之一。意味着只要按照前面提到过的 Maven 工程结构组织我们的规则文件或流程文件，只用很少的代码完成模型的加载和构建。

##### 2.1.11 kmodule.xml

kmodule 的具体的配置。

***<u>kbase的属性:</u>***

| 属性名              | 默认值   | 合法的值                           | 描述                                                         |
| :------------------ | -------- | ---------------------------------- | ------------------------------------------------------------ |
| name                | none     | any                                | KieBase 的名称，这个属性是强制的，  必须设置。               |
| includes            | none     | 逗 号 分 隔 的 KieBase 名 称 列 表 | 意 味 着 本 KieBase 将 会 包 含 所 有 include 的 KieBase 的 rule、process 定 义制品文件。非强制属性。 |
| packages            | all      | 逗号分隔的字符  串列表             | 默认情况下将包含 resources 目录下  面（子目录）的所有规则文件。也可  以指定具体目录下面的规则文件，通  过逗号可以包含多个目录下的文件。 |
| default             | false    | true, false                        | 表示当前 KieBase 是不是默认的，如 果是默认的话，不用名称就可以查找 到该 KieBase，但是每一个 module 最 多只能有一个 KieBase。 |
| equalsBehavior      | identity | identity,equality                  | 顾名思义就是定义“equals”（等于）的  行为，这个 equals 是针对 Fact（事实）  的， 当插入一个 Fact 到 Working  Memory 中的时候，Drools 引擎会检  查该 Fact 是否已经存在，如果存在的  话就使用已有的 FactHandle，否则就  创建新的。而判断 Fact 是否存在的依  据通过该属性定义的方式来进行的：  设置成 identity ，就是判断对象是否  存在，可以理解为用==判断，看是否  是同一个对象； 如果该属性设置成  equality 的话，就是通过 Fact 对象的  equals 方法来判断。 |
| eventProcessingMode | cloud    | cloud, stream                      | 当以云模式编译时，KieBase 将事件视 为正常事实，而在流模式下允许对其 进行时间推理。 |
| declarativeAgenda   | disabled | disabled,enabled                   | 这是一个高级功能开关，打开后规则将可以控制一些规则的执行与否。 |

***<u>ksession 的属性：</u>***

| 属性名       | 默认值   | 合法的值                 | 描述                                                         |
| ------------ | -------- | ------------------------ | ------------------------------------------------------------ |
| name         | none     | any                      | KieSession 的名称，该值必须唯一，也是强  制的，必须设置。    |
| type         | stateful | stateful, stateless      | 定义该 session 到底是有状态（stateful）的还 是无状态（stateless）的，有状态的 session 可 以利用 Working Memory 执行多次，而无状 态的则只能执行一次。 |
| default      | false    | true, false              | 定义该 session 是否是默认的，如果是默认的  话则可以不用通过 session 的 name 来创建  session，在同一个 module 中最多只能有一个  默认的 session。 |
| clockType    | realtime | realtime,pseudo          | 定义时钟类型，用在事件处理上面，在复合事 件处理上会用到，其中 realtime 表示用的是 系统时钟，而 pseudo 则是用在单元测试时模 拟用的。 |
| beliefSystem | simple   | simple,defeasible,  jtms | 定义 KieSession 使用的 belief System 的类型。                |

#### 2.2 规则

在前面的章节中我们已经实现了一个简单规则引擎的使用。规则引擎的优势所在那就是将可变的部分剥离出来放置在 DRL 文件中，规则的变化只用修 改 DRL 文件中的逻辑即可，而其他相关的业务代码则几乎不用改动。

本章节的重点介绍规则文件的构成、语法函数、执行及各种属性等。

##### 2.2.1 规则文件

一个标准的规则文件的格式为已“.drl”结尾的文本文件，因此可以通过记事本工具 进行编辑。规则放置于规则文件当中，一个规则文件可以放置多条规则。在规则文件当 中也可以存放用户自定义的函数、数据对象及自定义查询等相关在规则当中可能会用到 的一些对象。

从架构角度来讲，一般将同一业务的规则放置在同一规则文件，也可根据不同类型处理操作放置在不同规则文件当中。不建议将所有的规则放置与一个规则文件当中。分开放置，当规则变动时不至于影响到不相干的业务。读取构建规则的成本业务会相应 减少。

***<u>标准规则文件的结构如下：</u>***

```drools
package package-name

imports

globals

functions

queries

rules
```

- package：

  在一个规则文件当中 package 是必须的，而且必须放置在文件的第一行。 package 的名字是随意的，不是必须对应物理路径，这里跟 java 的 package 的概念不同， 只是逻辑上的区分，但建议与文件路径一致。***<u>同一的 package 下定义的 function 和 query 等 可以直接使用</u>***。比如，上面实例中 package 的定义: package com.rules

- import：

  导入规则文件需要的外部变量，使用方法跟 java 相同。像 java 的是 import 一 样，***<u>还可以导入类中的某一个可访问的静态方法</u>***。（特别注意的是，某些教程中提示 import 引入静态方法是不同于 java 的一方面，可能是作者没有用过 java 的静态方法引入。）另外， 目前针对 Drools7 版本，static 和 function 关键字的效果是一样的。

  ```Java
  import static com.secbro.drools.utils.DroolsStringUtils.isEmpty;
  import function com.secbro.drools.utils.DroolsStringUtils.isEmpty;
  ```

- rule：

  定义一个条规则。rule “ruleName”。一条规则包含三部分：属性部分、条件部分 和结果部分。rule 规则以 rule 开头，以 end 结尾。

  属性部分：定义当前规则执行的一些属性等，比如是否可被重复执行、过期时间、生效 时间等。

  条件部分，简称 LHS，即 Left Hand Side。定义当前规则的条件，处于 when 和 then 之 间。如 when Message();判断当前 workingMemory 中是否存在 Message 对象。LHS 中，可 包含 0~n 个条件，如果没有条件，默认为 eval(true)，也就是始终返回 true。条件又称之为 pattern（匹配模式），多个 pattern 之间用可以使用 and 或 or 来进行连接，同时还可以使用 小括号来确定 pattern 的优先级。

  结果部分，简称 RHS，即 Right Hand Side，处于 then 和 end 之间，用于处理满足条件 之后的业务逻辑。可以使用 LHS 部分定义的变量名、设置的全局变量、或者是直接编写 Java 代码。

  RHS 部分可以直接编写 Java 代码，但不建议在代码当中有条件判断，如果需要条件判 断，那么需要重新考虑将其放在 LHS 部分，否则就违背了使用规则的初衷。

  RHS 部分，提供了一些对当前 WorkingMemory 实现快速操作的宏函数或对象，比如 insert/insertLogical、update/modify 和 retract 等。利用这些函数可以实现对当前 Working Memory 中的 Fact 对象进行新增、修改或删除操作；如果还要使用 Drools 提供的其它方法， 可以使用另一个宏对象 drools，通过该对象可以使用更多的方法；同时 Drools 还提供了一 个 名 为 kcontext 的 宏 对 象 ， 可以通过该对象直接访问当前Working Memory的 KnowledgeRuntime。

  ***<u>标准规则的结构示例：</u>***

  ```
  rule "name"
  attributes
  when
  	LHS
  then
  	RHS
  end
  ```

#### 2.3 包(package)

在 Drools 中，包的概念与 java 基本相同，区别的地方是 ***<u>Drools 的中包不一定与物理路 径相对应</u>***。包一组规则和相关语法结构，比如引入和全局变量。在同一包下的规则相互之间 有一定的关联。

***<u>一个包代表一个命名空间，一般来讲针对同一组规则要确保包的唯一性。包名本身是命 名空间，并且不以任何方式关联到文件或文件夹。</u>***

![package](/Users/sherlock/Desktop/notes/allPics/Drools/package.png)

##### 2.3.1 import

Drools 的导入语句的作用和在 java 一样。需要为在规则中使用的对象指定完整路径和类 型名字。 Drools 会自动从同名的 java 包中导入类，***<u>默认会引入 java.lang 下面的类</u>***。

##### 2.3.2 global 全局变量

global 用来定义全局变量，***<u>它可以让应用程序的对象在规则文件中能够被访问</u>***。通常， 可以用来为规则文件提供数据或服务。特别是用来操作规则执行结果的处理和从规则返回数据，比如执行结果的日志或值，或者与应用程序进行交互的规则的回调处理。

全局变量并不会被插入到 Working Memory 中，因此，除非作为常量值，否则不应该将 全局变量用于规则约束的判断中。对规则引擎中的 fact 修改，规则引擎根据算法会动态更新决策树，重新激活某些规则的执行，而全局变量不会对规则引擎的决策树有任何影响。在约束条件中错误的使用全局变量会导致意想不到的错误。

***<u>如果多个包中声明具有相同标识符的全局变量，则必须是相同的类型，并且它们都将引 用相同的全局值。</u>***

实例代码如下：

规则文件内容：

```Java
package com.rules

import com.secbro.drools.model.Risk
import com.secbro.drools.model.Message

global com.secbro.drools.EmailService emailService

rule "test-global"
agenda-group "test-global-group"
	when

	then
		Message message = new Message();
		message.setRule(drools.getRule().getName());
		message.setDesc("to send email!");
		emailService.sendEmail(message);
end
```

测试代码:

```Java
@Test
public void testGlobal(){
	KieSession kieSession = this.getKieSession("test-global-group");
	int count = kieSession.fireAllRules();
	kieSession.dispose();
	System.out.println("Fire " + count + "rules!");
}
```

实体类:

```Java
public class Message {
  private String rule;
  private String desc;
  // getter/sette省略
}
```

操作类:

```Java
public class EmailService {
	public static void sendEmail(Message message){
		System.out.println("Send message to email,the fired rule is '" + message.getRule() + "', and description is '" + message.getDesc() + "'");
	}
}
```

#### 2.4 规则属性

规则属性提供了一种声明性的方式来影响规则的行为。有些很简单，而其他的则是复杂 子系统的一部分，比如规则流。如果想充分使用 Drools 提供的便利，应该对每个属性都有 正确的理解。

![规则属性](/Users/sherlock/Desktop/notes/allPics/Drools/规则属性.png)

##### 2.4.1 no-loop

定义当前的规则是否不允许多次循环执行，默认是 false，也就是当前的规则只要满足条 件，可以无限次执行。什么情况下会出现规则被多次重复执行呢？下面看一个实例：

```Java
package com.rules

import com.secbro.drools.model.Product;

rule "updateDistcount"
no-loop false
	when
		productObj:Product(discount > 0);
	then
		productObj.setDiscount(productObj.getDiscount() + 1);
		System.out.println(productObj.getDiscount());
		update(productObj);
end
```

其中 Product 对象的 discount 属性值默认为 1。执行此条规则时就会发现程序进入了死 循环。也就是说对传入当前 workingMemory 中的 FACT 对象的属性进行修改，并调用 update 方法就会重新触发规则。从打印的结果来看，update 之后被修改的数据已经生效，在重新 执行规则时并未被重置。当然对 Fact 对象数据的修改并不是一定需要调用 update 才可以生 效，简单的使用 set 方法设置就可以完成，但仅仅调用 set 方法时并不会重新触发规则。所 以，对 `insert、retract、update` 之类的方法使用时一定要慎重，否则极可能会造成死循环。

可以通过设置 no-loop 为 true 来避免规则的重新触发，同时，如果本身的 RHS 部分有 insert、retract、update 等触发规则重新执行的操作，也不会再次执行当前规则。

上面的设置虽然解决了当前规则的不会被重复执行，但其他规则还是会收到影响，比如 下面的例子：

```Java
package com.rules

import com.secbro.drools.model.Product;

rule "updateDistcount"
no-loop true
	when
		productObj:Product(discount > 0);
	then
		productObj.setDiscount(productObj.getDiscount() + 1);
		System.out.println(productObj.getDiscount());
		update(productObj);
end

rule "otherRule"
	when
		productObj : Product(discount > 1);
	then
		System.out.println("被触发了" + productObj.getDiscount());
end
```

此时执行会发现，当第一个规则执行 update 方法之后，规则 otherRule 也会被触发执 行。如果注释掉 update 方法，规则 otherRule 则不会被触发。那么，这个问题是不是就没办法解决了？当然可以，那就是引入 `lock-on-active true` 属性。

##### 2.4.2 ruleflow-group

在使用规则流的时候要用到 ruleflow-group 属性，该属性的值为一个字符串，作用是将 规则划分为一个个的组，然后在规则流当中通过使用 ruleflow-group 属性的值，从而使用对应的规则。该属性会通过流程的走向确定要执行哪一条规则。在规则流中有具体的说明。

代码实例：

```Java
package com.rules

rule "test-ruleflow-group1"
ruleflow-group "group1"
	when
	
	then
		System.out.println("规则test-ruleflow-group1 被触发");
end
  
rule "test-ruleflow-group2"
ruleflow-group "group1"
  when

  then
  	System.out.println("规则test-ruleflow-group2 被触发");
end
```

##### 2.4.3 lock-on-active

当在规则上使用 ruleflow-group 属性或 agenda-group 属性的时候，将 lock-on-active 属性的值设置为 true，可避免因某些 Fact 对象被修改而使已经执行过的规则再次被激活执行。可以看出该属性与 no-loop 属性有相似之处，no-loop 属性是为了避免 Fact 被修改或 调用了 insert、retract、update 之类的方法而导致规则再次激活执行，这里的 lock-on-active 属性起同样的作用，lock-on-active 是 no-loop 的增强版属性，它主要作用在使用 ruleflowgroup 属性或 agenda-group 属性的时候。lock-on-active 属性默认值为 false。与 no-loop 不同的是 lock-on-active 可以避免其他规则修改 FACT 对象导致规则的重新执行。

通过 lock-on-active 属性来避免被其他规则更新导致自身规则被重复执行示例：

```Java
package com.rules

import com.secbro.drools.model.Product;

rule "rule1"
no-loop true
  when
  	obj : Product(discount > 0);
  then
  	obj.setDiscount(obj.getDiscount() + 1);
  	System.out.println("新折扣为：" + obj.getDiscount());
  	update(obj);
end

rule "rule2"
lock-on-active true
  when
  	productObj : Product(discount > 1);
  then
  	System.out.println("其他规则被触发了" + productObj.getDiscount());
end
```

很明显在 rule2 的属性部分新增了 lock-on-active true。执行结果为：

```
新折扣为：2
第一次执行命中了 1 条规则！
```

标注了 lock-on-active true 的规则不再被触发。

##### 2.4.4 salience

用来设置规则执行的优先级，salience 属性的值是一个数字，数字越大执行优先级越高， 同时它的值可以是一个负数。默认情况下，规则的 salience 默认值为 0。如果不设置规则的 salience 属性，那么执行顺序是随机的。

示例代码：

```Java
package com.rules

rule "salience1"
salience 3
	when
  	
  then
  	System.out.println("salience1 被执行");
end

rule "salience2"
salience 5
  when
  	
  then
  	System.out.println("salience2 被执行");
end
```

执行结果：

```
salience2 被执行
salience1 被执行
```

显然，salience2 的优先级高于 salience1 的优先级，因此被先执行。

Drools 还支持动态 saline，可以使用绑定绑定变量表达式来作为 salience 的值。比如：

```Java
package com.rules

import com.secbro.drools.model.Product

rule "salience1"
salience sal
  when
  	Product(sal:discount);
  then
  	System.out.println("salience1 被执行");
end
```

这样，salience 的值就是传入的 FACT 对象 Product 的 discount 的值了。

##### 2.4.5 agenda-group

规则的调用与执行是通过 `StatelessKieSession` 或 `KieSession` 来实现的，一般的顺序是创建一个 StatelessKieSession 或 KieSession，将各种经过编译的规则添加到 session 当中，然后 将规则当中可能用到的 Global 对象和 Fact 对象插入到 Session 当中，最后调用 fireAllRules 方法来触发、执行规则。

在没有调用 fireAllRules 方法之前，所有的规则及插入的 Fact 对象都存放在一个 Agenda 表的对象当中，这个 Agenda 表中每一个规则及与其匹配相关业务数据叫做 Activation，在 调用 fireAllRules 方法后，这些 Activation 会依次执行，执行顺序在没有设置相关控制顺序属 性时（比如 salience 属性），它的执行顺序是随机的。

Agenda Group 是用来在 Agenda 基础上 对规则进行再次分组， 可 通过 为规则添加 agenda-group 属性来实现。agenda-group 属性的值是一个字符串，通过这个字符串，可以 将规则分为若干个 Agenda Group。引擎在调用设置了 agenda-group 属性的规则时需要显 示的指定某个 Agenda Group 得到 Focus（焦点），否则将不执行该 Agenda Group 当中的规则。

规则代码:

```Java
package com.rules

rule "test agenda-group"
agenda-group "abc"
  when

  then
  	System.out.println("规则 test agenda-group 被触发");
end

rule "otherRule"
  when
  
  then
  	System.out.println("其他规则被触发");
end
```

调用代码：

```Java
KieServices kieServices = KieServices.Factory.get();
KieContainer kieContainer = kieServices.getKieClasspathContainer();
KieSession kSession = kieContainer.newKieSession("ksession-rule");
kSession.getAgenda().getAgendaGroup("abc").setFocus();
kSession.fireAllRules();
kSession.dispose();
```

执行以上代码，打印结果为：

```
规则 test agenda-group 被触发
```

其他规则被触发

如果将代码 `kSession.getAgenda().getAgendaGroup("abc").setFocus()`注释掉，则只会打 印出：

```
其他规则被触发
```

很显然，***<u>如果不设置指定 AgendaGroup 获得焦点，则该 AgendaGroup 下的规则将不会 被执行。</u>***

##### 2.4.6 activation-group

该属性将若干个规则划分成一个组，统一命名。在执行的时候，具有相同 activation-group 属性的规则中只要有一个被执行，其它的规则都不再执行。可以用类似 salience 之类属性来 实现规则的执行优先级。该属性以前也被称为异或（Xor）组，但技术上并不是这样实现的， 当提到此概念，知道是该属性即可。

实例代码：

```Java
package com.rules

rule "test-activation-group1"
activation-group "foo"
  when

  then
  	System.out.println("规则test-activation-group1 被触发");
end

rule "test-activation-group2"
activation-group "foo"
salience 1
  when

  then
  	System.out.println("规则test-activation-group2 被触发");
end
```

执行规则之后，打印结果：

```
规则test-activation-group2 被触发
```

以上实例证明，同一 activation-group 优先级高的被执行，其他规则不会再被执行。

##### 2.4.7 enabled

设置规则是否可用。true：表示该规则可用；false：表示该规则不可用。

#### 2.5 LHS语法

##### 2.5.1 LHS语法简介

LHS 是规则条件部分的 统称，由 0 个或多个条件元素组成。前面我们已经提到，如果没有条件元素那么默认就是 true。

没有条件元素，官方示例：

```Java
rule "no CEs"
  when
  // empty
  then
  ... // actions (executed once)
end

// The above rule is internally rewritten as:
rule "eval(true)"
  when
  	eval( true )
  then
  ... // actions (executed once)
end
```

如果有多条规则元素，默认它们之间是“和”的关系，也就是说必须同时满足所有的条件 元素才会触发规则。官方示例：

```Java
rule "2 unconnected patterns"
  when
    Pattern1()
    Pattern2()
  then
  ... // actions
end

// The above rule is internally rewritten as:
rule "2 and connected patterns"
  when
    Pattern1() and Pattern2()
  then
  ... // actions
end
```

和“or”不一样，“and”不具有优先绑定的功能。因为申明一次只能绑定一个 FACT 对象， 而当使用 and 时就无法确定声明的变量绑定到哪个对象上了。以下代码会编译出错。

```
$person : (Person( name == "Romeo" ) and Person( name == "Juliet"))
```

##### 2.5.2 Pattern(条件元素)

Pattern 元素是最重要的一个条件元素，它可以匹配到插入 working memory 中的每个 FACT 对象。一个 Pattern 包含0到多个约束条件，同时可以选择性的进行绑定。

![Pattern](/Users/sherlock/Desktop/notes/allPics/Drools/Pattern.png)

通过上图可以明确的知道 Pattern 的使用方式，左边变量定义，然后用冒号分割。右边 pattern 对象的类型也就是 FACT 对象，后面可以在括号内添加多个约束条件。最简单的一 种形式就是，只有 FACT 对象，没有约束条件，这样一个 pattern 配到指定的 patternType 类 即可。

比如，下面的 pattern 定义表示匹配 Working Memory 中所有的 Person 对象。

```
Person()
```

pattemType 并不需要使用实际存在的 FACT 类，比如下面的定义表示匹配 Working Memory 中所有的对象。很明显，Object 是所有类的父类。

```
Object() // 匹配 working memory 中的所有对象
```

如下面的示例，括号内的表达式决定了当前条件是否会被匹配到，这也是实际应用中最 常见的使用方法。

```
Person( age == 100 )
```

Pattern 绑定：当匹配到对象时，可以将 FACT 对象绑定到指定的变量上。这里的用法类 似于 java 的变量定义。绑定之后，在后面就可以直接使用此变量。

```Java
rule ...
  when
  	$p : Person()
  then
  	System.out.println( "Person " + $p );
end
```

其中前缀$只是一个约定标识，有助于在复杂的规则中轻松区分变量和字段，但并不强制要求必须添加此前缀。

##### 4.5.3 约束(Pattern的一部分)

前面我们已经介绍了条件约束在 Pattern 中位置了，那么什么是条件约束呢？简单来说就 是一个返回 true 或者 false 的表达式，比如下面的 5 小于 6，就是一个约束条件。

```
Person(5 < 6)
```

从本质上来讲，它是 JAVA 表达式的一种增强版本（比如属性访问），同时它又有一些 小的区别，比如 equals 方法和==的语言区别。下面我们就深入了解一下。

###### 4.5.3.1 访问JavaBean中的属性

任何一个 JavaBean 中的属性都可以访问， 不过对应的属性要提供 getter 方法或 isProperty 方法。比如：

```Java
Person(age == 50)

// 与上面拥有同样的效果
Person(getAge() == 50)
```

Drools 使用 java 标准的类检查，因此遵循 java 标准即可。同时，嵌套属性也是支持的， 比如：

```Java
Person(address.houseNumber == 50)

// 与上面写法相同
Person(getAddress().getHouseNumber() == 50)
```

在使用有状态 session 的情况下使用嵌套属性需要注意属性的值可能被其他地方修改。 要么认为它们是不可变的，当任何一个父引用被插入到 working memory 中。或者，如果要 修改嵌套属性值，则应将所有外部 fact 标记更新。在上面的例子中，当 houseNumber 属性 值改变时，任何一个包含 Address 的 Person 需要被标记更新。

###### 4.5.3.2 Java表达式

在 pattern 的约束条件中，可以返回任何结果为布尔类型的 java 表达式。当然，java 表达式也可以和增强的表达式进行结合使用，比如属性访问。可以通过使用括号来更改计算优 先级，如在任一逻辑或数学表达式中。

```javascript
Person(age > 100 && (age % 10 == 0 ))
```

也可以直接使用 java 提供的工具方法来进行操作计算：

```Java
Person(Math.round(weight / (height * height)) < 25.0)
```

在使用的过程中需要注意，在 LHS 中执行的方法只能是只读的，不能在执行方法过程 中改变 FACT 对象的值，否则会影响规则的正确执行。

```Java
//不要像这样在比较的过程中更新 Fact 对象
Person(incrementAndGetAge() == 10) 
```

另外，FACT 对象的相关状态除了在 working memory 中可以进行更新操作，不应该在每次调用时使状态发生变化。

```Java
// 不要这样实现
Person(System.currentTimeMillis() % 1000 == 0) 
```

标准 Java 运算符优先级也适用于此处，详情参考下面的运算符优先级列表。所有的操 作符都有标准的 Java 语义，除了==和!=。它们是 null 安全的，就相当于 java 中比较两个 字符串时把常量字符串放前面调用 equals 方法的效果一样。

约束条件的比较过程中是会进行强制类型转换的，比如在数据计算中传入字符串“10”， 则能成功转换成数字 10 进行计算。如果传入的值无法进行转换，比如传了“ten”，会抛出异 常。

###### 4.5.3.3 逗号分隔符

逗号可以对约束条件进行分组，它的作用相当于“AND”。

```Java
// Person 的年龄要超过 50，并且重量 超过 80 kg
Person(age > 50, weight > 80)
```

虽然`&&`和`,`拥有相同的功能，但是它们有不同的优先级。`&&`优先于`||`，`&&`和`||`又优先于`,`。建议优先使用“,”分隔符，因为它更利于阅读理解和引擎的操作优化。同时，逗号分隔符不能和其他操作符混合使用，比如：

```Java
Person((age > 50, weight > 80)||height > 2) // 会编译错误

// 使用此种方法替代
Person((age > 50 && weight > 80)||height > 2)
```

###### 4.5.3.4 绑定变量

一个属性也可以绑定到一个变量：

```Java
// 2 person 的 age 属性值相同
Person($firstAge : age) // 绑定
Person(age == $firstAge) // 约束表达式
```

前缀$只是个通用惯例，在复杂规则中可以通过它来区分变量和属性。为了向后兼容， 允许（但不推荐）混合使用约束绑定和约束表达式。

```Java
// 不建议这样写
Person($age : age * 2 < 100)

// 推荐（分离绑定和约束表达式）
Person(age * 2 < 100, $age : age)
```

使用操作符“==”来绑定变量，Drools 会使用散列索引来提高执行性能。

