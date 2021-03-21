## 【Zookeeper】Curator

## 一、简介

Curator 是 Netflix 公司开源的一套 zookeeper 客户端框架，了解过Zookeeper原生API都会清楚其复杂度。Curator帮助我们在其基础上进行封装、实现一些开发细节，包括接连重连、反复注册Watcher 和 NodeExistsException 等。

目前 Curator 已经作为 Apache 的顶级项目出现，是最流行的 Zookeeper 客户端之一。从编码风格上来讲，它提供了基于Fluent的编程风格支持。此外，Curator 还提供了 Zookeeper 的各种应用场景：Recipe、共享锁服务、Master 选举机制和分布式计数器等。

Patrixck Hunt（Zookeeper）以一句 “Guava is to Java that Curator to Zookeeper” 给Curator予高度评价。

目前(截止2021.1) Curator 有 2.x.x、3.x.x、4.x.x 和 5.x.x 这几个系列的版本，以支持不同版本的 Zookeeper。其中 Curator 2.x.x 兼容 Zookeeper 的 3.4.x 和 3.5.x。而Curator 3.x.x 只兼容 Zookeeper 3.5.x，并且提供了一些诸如动态重新配置、watch 删除等新特性。

> Tips: Curator 5.x.x 正式申明不兼容 Zookeeper-3.4.x 这个版本, 如果要使用这个版本的 Zookeeper, 则需要使用 Curator-4.2.x, 并且要 exclude ZooKeeper。各个管理工具的具体依赖如下:
>
> ```ini
> // maven
> <dependency>
>     <groupId>org.apache.curator</groupId>
>     <artifactId>curator-recipes</artifactId>
>     <version>4.2.0</version>
>     <exclusions>
>         <exclusion>
>             <groupId>org.apache.zookeeper</groupId>
>             <artifactId>zookeeper</artifactId>
>         </exclusion>
>     </exclusions>
> </dependency>
> <dependency>
>     <groupId>org.apache.zookeeper</groupId>
>     <artifactId>zookeeper</artifactId>
>     <version>3.4.x</version>
> </dependency>
> 
> // gradle
> compile('org.apache.curator:curator-recipes:4.2.0') {
>   exclude group: 'org.apache.zookeeper', module: 'zookeeper'
> }
> compile('org.apache.zookeeper:zookeeper:3.4.x')
> 
> // sbt
> libraryDependencies += "org.apache.curator" % "curator-recipes" % "4.2.0" exclude("org.apache.zookeeper", "zookeeper")
> libraryDependencies += "org.apache.zookeeper" % "zookeeper" % "3.4.x"
> ```

### 1.1 Curator Modules

- curator-framework

  对 zookeeper 的底层 api 的一些封装。

- curator-client

  提供一些客户端的操作，例如重试策略等。

- curator-recipes

  封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式计数器、分布式Barrier等。

- curator-test

  包含TestingServer、TestingCluster和一些测试工具。

- curator-examples 

  各种使用Curator特性的案例。

- curator-x-discovery 

  在framework上构建的服务发现实现。

- curator-x-discoveryserver 

  可以和 Curator Discovery 一起使用的 RESTful服务器。

- curator-x-rpc 

  Curator framework和recipes非jvm环境的桥接。

常用的 module 有 `curator-framework` 和 `curator-recipes` 这两个, 下面主要介绍这两个模块的一些功能。

## 二、Curator的基本API

### 2.1 使用静态工程方法创建客户端

一个例子如下：

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client =
CuratorFrameworkFactory.newClient(
						connectionInfo,
						5000,
						3000,
						retryPolicy);
```

newClient静态工厂方法包含四个主要参数：

| 参数名              | 说明                                                      |
| ------------------- | --------------------------------------------------------- |
| connectionString    | 服务器列表，格式host1:port1,host2:port2,…                 |
| retryPolicy         | 重试策略,内建有四种重试策略,也可以自行实现RetryPolicy接口 |
| sessionTimeoutMs    | 会话超时时间，单位毫秒，默认60s                           |
| connectionTimeoutMs | 连接创建超时时间，单位毫秒，默认60s                       |

### 2.2 使用Fluent风格的Api创建会话

核心参数变为流式设置，一个列子如下：

```java
COPYRetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client = CuratorFrameworkFactory.builder()
  .connectString(connectionInfo)
  .sessionTimeoutMs(5000)
  .connectionTimeoutMs(5000)
  .retryPolicy(retryPolicy)
  .build();
```

### 2.3 创建包含隔离命名空间的会话

为了实现不同的Zookeeper业务之间的隔离，需要为每个业务分配一个独立的命名空间（**NameSpace** 也可以理解为一个项目一个根目录），即指定一个Zookeeper的根路径（官方术语：***为 Zookeeper 添加 “Chroot” 特性***）。例如（下面的例子）当客户端指定了独立命名空间为“/base”，那么该客户端对Zookeeper上的数据节点的操作都是基于该目录进行的。通过设置Chroot可以将客户端应用与Zookeeper服务端的一课子树相对应，在多个应用共用一个Zookeeper集群的场景下，这对于实现不同应用之间的相互隔离十分有意义。

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
		CuratorFramework client =
		CuratorFrameworkFactory.builder()
				.connectString(connectionInfo)
				.sessionTimeoutMs(5000)
				.connectionTimeoutMs(5000)
				.retryPolicy(retryPolicy)
				.namespace("base")
				.build();
```

### 2.4 启动客户端

当创建会话成功，得到client的实例然后可以直接调用其start( )方法：

```java
client.start();
```

> 这里看了下源码里面 start() 方法的具体逻辑:
>
> ```java
> public void start()
> {
>     log.info("Starting");
>   	// 这里使用了 Java 的 AtomicReference 类来装载 CuratorFrameworkState
>   	// 保证并发状态下 CuratorFrameworkState 的一致性, 状态从 LATENT 切换到 STARTED 的一致性, 如果客户端之前状态不是 LATENT 则报错
>     if ( !state.compareAndSet(CuratorFrameworkState.LATENT, CuratorFrameworkState.STARTED) )
>     {
>         throw new IllegalStateException("Cannot be started more than once");
>     }
> 
>     try
>     {
>       	// 在 ConnectionStateManager 中也管理了客户端的状态 State, 它本质上和 CuratorFrameworkState没有任何区别 (在 Curator 中还有ZK连接状态这个概念 ConnectionState, 这里要区分清楚).
>       	// 接着在这个 ConnectionStateManager的start()方法中调用了 newSingleThreadExecutor 这个线程池去执行 processEvents()这个方法, 这里代码会继续执行, 但是后台有个线程池在执行 processEvents()
>         connectionStateManager.start(); // ordering dependency - must be called before client.start()
> 				
>       	// 这里就是调用了一个回调接口将当前ZK连接状态状态返给 ConnectionStateManager 中的 eventQueue 阻塞队列, 所以在processEvents()方法中可以获取到新的 newState = "CONNECTED", 这里设计太精妙了
>         final ConnectionStateListener listener = new ConnectionStateListener()
>         {
>             @Override
>             public void stateChanged(CuratorFramework client, ConnectionState newState)
>             {
>                 if ( ConnectionState.CONNECTED == newState || ConnectionState.RECONNECTED == newState )
>                 {
>                     logAsErrorConnectionErrors.set(true);
>                 }
>             }
>         };
> 
>         this.getConnectionStateListenable().addListener(listener);
> 
>         client.start();
> 
>         executorService = Executors.newSingleThreadScheduledExecutor(threadFactory);
>         executorService.submit(new Callable<Object>()
>         {
>             @Override
>             public Object call() throws Exception
>             {
>                 backgroundOperationsLoop();
>                 return null;
>             }
>         });
>     }
>     catch ( Exception e )
>     {
>         ThreadUtils.checkInterrupted(e);
>         handleBackgroundOperationException(null, e);
>     }
> }
> ```

## 三、数据节点操作

### 3.1 创建数据节点

1. ***Zookeeper的节点创建模式***
   - PERSISTENT：持久化节点
   - PERSISTENT_SEQUENTIAL：持久化并且带序列号的节点
   - EPHEMERAL：临时节点
   - EPHEMERAL_SEQUENTIAL：临时并且带序列号的节点

2. ***创建一个节点，初始内容为空***

   ```java
   client.create().forPath("path");
   ```

   注意：如果没有设置节点属性，节点创建模式默认为持久化节点，内容默认为空

3. ***创建一个节点，附带初始化内容***

   ```java
   client.create().forPath("path","init".getBytes());
   ```

4. ***创建一个节点，指定创建模式（临时节点），内容为空***

   ```JAVA
   client.create().withMode(CreateMode.EPHEMERAL).forPath("path");
   ```

5. ***创建一个节点，指定创建模式（临时节点），附带初始化内容***

   ```java
   client.create()
     .withMode(CreateMode.EPHEMERAL)
     .forPath("path","init".getBytes());
   ```

6. ***创建一个节点，指定创建模式（临时节点），附带初始化内容，并且自动递归创建父节点***

   ```java
   client.create()
         .creatingParentContainersIfNeeded()
         .withMode(CreateMode.EPHEMERAL)
         .forPath("path","init".getBytes());
   ```

   这个 creatingParentContainersIfNeeded() 接口非常有用，因为一般情况开发人员在创建一个子节点必须判断它的父节点是否存在，如果不存在直接创建会抛出NoNodeException，使用creatingParentContainersIfNeeded()之后Curator能够自动递归创建所有所需的父节点。

### 3.2 删除数据节点

1. **删除一个节点**

   ```java
   client.delete().forPath("path");
   ```

   注意，此方法只能删除**叶子节点**，否则会抛出异常。

2. **删除一个节点，并且递归删除其所有的子节点**

   ```java
   client.delete().deletingChildrenIfNeeded().forPath("path");
   ```

3. **删除一个节点，强制指定版本进行删除**

   ```java
   client.delete().withVersion(10086).forPath("path");
   ```

4. **删除一个节点，强制保证删除**

   ```java
   client.delete().guaranteed().forPath("path");
   ```

   guaranteed()接口是一个保障措施，只要客户端会话有效，那么Curator会在后台持续进行删除操作，直到删除节点成功。

**注意：**上面的多个流式接口是可以自由组合的，例如：

```
COPYclient.delete().guaranteed().deletingChildrenIfNeeded().withVersion(10086).forPath("path");
```

### 3.3 读取数据节点数据

1. **读取一个节点的数据内容**

   ```java
   client.getData().forPath("path");
   ```

   注意，此方法返的返回值是byte[ ];

2. **读取一个节点的数据内容，同时获取到该节点的stat**

   ```java
   Stat stat = new Stat();
   client.getData().storingStatIn(stat).forPath("path");
   ```

#### 3.4 更新数据节点数据

1. **更新一个节点的数据内容**

   ```java
   client.setData().forPath("path","data".getBytes());
   ```

   注意：该接口会返回一个Stat实例

2. **更新一个节点的数据内容，强制指定版本进行更新**

   ```java
   client.setData()
     .withVersion(10086)
     .forPath("path","data".getBytes());
   ```

#### 3.4 检查节点是否存在

```
client.checkExists().forPath("path");
```

注意：该方法返回一个Stat实例，用于检查ZNode是否存在的操作. 可以调用额外的方法(监控或者后台处理)并在最后调用`forPath()`指定要操作的ZNode

#### 3.5 获取某个节点的所有子节点路径

```
client.getChildren().forPath("path");
```

注意：该方法的返回值为List,获得ZNode的子节点Path列表。 可以调用额外的方法(监控、后台处理或者获取状态watch, background or get stat) 并在最后调用forPath()指定要操作的父ZNode

#### 3.6 事务

CuratorFramework 的实例包含 inTransaction( ) 接口方法，调用此方法开启一个ZooKeeper事务. 可以复合 create, setData, check, and/or delete 等操作然后调用commit()作为一个原子操作提交。一个例子如下：

```java
client.inTransaction().check().forPath("path")
    .and()
    .create()
  	.withMode(CreateMode.EPHEMERAL).forPath("path","data".getBytes())
    .and()
    .setData().withVersion(10086).forPath("path","data2".getBytes())
    .and()
    .commit();
```

#### 3.7 异步接口

上面提到的创建、删除、更新、读取等方法都是同步的，Curator提供异步接口，引入了**BackgroundCallback** 接口用于处理异步接口调用之后服务端返回的结果信息。**BackgroundCallback** 接口中一个重要的回调值为CuratorEvent，里面包含事件类型、响应吗和节点的详细信息。

**CuratorEventType**

| 事件类型 | 对应CuratorFramework实例的方法 |
| -------- | ------------------------------ |
| CREATE   | #create()                      |
| DELETE   | #delete()                      |
| EXISTS   | #checkExists()                 |
| GET_DATA | #getData()                     |
| SET_DATA | #setData()                     |
| CHILDREN | #getChildren()                 |
| SYNC     | #sync(String,Object)           |
| GET_ACL  | #getACL()                      |
| SET_ACL  | #setACL()                      |
| WATCHED  | #Watcher(Watcher)              |
| CLOSING  | #close()                       |

**响应码(#getResultCode())**

| 响应码 | 意义                                     |
| ------ | ---------------------------------------- |
| 0      | OK，即调用成功                           |
| -4     | ConnectionLoss，即客户端与服务端断开连接 |
| -110   | NodeExists，即节点已经存在               |
| -112   | SessionExpired，即会话过期               |

一个异步创建节点的例子如下：

```java
COPYExecutor executor = Executors.newFixedThreadPool(2);
client.create()
      .creatingParentsIfNeeded()
      .withMode(CreateMode.EPHEMERAL)
      .inBackground((curatorFramework, curatorEvent) -> {                	     		System.out.println(String.format(
        		"eventType:%s,resultCode:%s",                                                                         						curatorEvent.getType(),
        		curatorEvent.getResultCode()));
      },executor)
      .forPath("path");
```

注意：如果#inBackground()方法不指定executor，那么会默认使用Curator的EventThread去进行异步处理。

## 四、Curator食谱(高级特性)

**提醒：首先你必须添加curator-recipes依赖，下文仅仅对recipes一些特性的使用进行解释和举例，不打算进行源码级别的探讨**

```xml
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-recipes</artifactId>
  <version>2.12.0</version>
</dependency>
```

**重要提醒：强烈推荐使用ConnectionStateListener监控连接的状态，当连接状态为LOST，curator-recipes下的所有Api将会失效或者过期，尽管后面所有的例子都没有使用到ConnectionStateListener。**

### 4.1 缓存

Zookeeper原生支持通过注册Watcher来进行事件监听，但是开发者需要反复注册(Watcher只能单次注册单次使用)。Cache是Curator中对事件监听的包装，可以看作是对事件监听的本地缓存视图，能够自动为开发者处理反复注册监听。Curator提供了三种Watcher(Cache)来监听结点的变化。

#### 4.1.1 Path Cache

Path Cache用来监控一个ZNode的子节点. 当一个子节点增加， 更新，删除时， Path Cache会改变它的状态， 会包含最新的子节点， 子节点的数据和状态，而状态的更变将通过PathChildrenCacheListener通知。

实际使用时会涉及到四个类：

- PathChildrenCache
- PathChildrenCacheEvent
- PathChildrenCacheListener
- ChildData

通过下面的构造函数创建Path Cache:

```java
public PathChildrenCache(CuratorFramework client, String path, boolean cacheData)
```

想使用cache，必须调用它的`start`方法，使用完后调用`close`方法。 可以设置StartMode来实现启动的模式，

StartMode有下面几种：

1. NORMAL：正常初始化。
2. BUILD_INITIAL_CACHE：在调用`start()`之前会调用`rebuild()`。
3. POST_INITIALIZED_EVENT： 当Cache初始化数据后发送一个PathChildrenCacheEvent.Type#INITIALIZED事件

`public void addListener(PathChildrenCacheListener listener)`可以增加listener监听缓存的变化。

`getCurrentData()`方法返回一个`List<ChildData>`对象，可以遍历所有的子节点。

**设置/更新、移除其实是使用client (CuratorFramework)来操作, 不通过PathChildrenCache操作：**

```java
public class PathCacheDemo {

	private static final String PATH = "/example/pathCache";

	public static void main(String[] args) throws Exception {
		TestingServer server = new TestingServer();
		CuratorFramework client = CuratorFrameworkFactory.
      newClient(server.getConnectString(), 
      new ExponentialBackoffRetry(1000, 3));
		client.start();
		PathChildrenCache cache = new PathChildrenCache(client, PATH, true);
		cache.start();
		PathChildrenCacheListener cacheListener = (client1, event) -> {
			System.out.println("事件类型：" + event.getType());
			if (null != event.getData()) {
				System.out.println("节点数据：" + event.getData().getPath() + " = " + new String(event.getData().getData()));
			}
		};
		cache.getListenable().addListener(cacheListener);
		client.create().creatingParentsIfNeeded().forPath("/example/pathCache/test01", "01".getBytes());
		Thread.sleep(10);
		client.create().creatingParentsIfNeeded().forPath("/example/pathCache/test02", "02".getBytes());
		Thread.sleep(10);
		client.setData().forPath("/example/pathCache/test01", "01_V2".getBytes());
		Thread.sleep(10);
		for (ChildData data : cache.getCurrentData()) {
			System.out.println("getCurrentData:" + data.getPath() + " = " + new String(data.getData()));
		}
		client.delete().forPath("/example/pathCache/test01");
		Thread.sleep(10);
		client.delete().forPath("/example/pathCache/test02");
		Thread.sleep(1000 * 5);
		cache.close();
		client.close();
		System.out.println("OK!");
	}
}
```

**注意：**如果new PathChildrenCache(client, PATH, true)中的参数cacheData值设置为false，则示例中的event.getData().getData()、data.getData()将返回null，cache将不会缓存节点数据。

**注意：**示例中的Thread.sleep(10)可以注释掉，但是注释后事件监听的触发次数会不全，这可能与PathCache的实现原理有关，不能太过频繁的触发事件！

