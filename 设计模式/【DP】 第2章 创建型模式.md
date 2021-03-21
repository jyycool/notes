## 【DP】 第2章 创建型模式

***创建型模式 (Creational Pattern)*** 主要用于处理对象的创建问题, 本章主要介绍以下内容:

- 单例模式
- 工厂模式
- 建造者模式
- 原型模式
- 对象池模式



### 1. 单例模式



### 2. 工厂模式



### 3.建造者模式

当需要实例化一个复杂的类, 以得到不同结构和不同内部状态的对象时, 我们可以使用不同的类对他们实例化的实例化操作逻辑进行封装, 这个类就称为建造者. 每当需要来自同一个类但具有不同结构的对象时, 就可以通过构造另一个建造者来进行实例化

> 这个模式多用于构建服务的客户端类中, 比如 Apache Curator.
>
> 上文中的意思是, 可以先构建一个 Builder 接口, 然后在各个类内部使用各自的嵌套类(静态内部类)来实现 Builder 接口, 这样每个类就都拥有了自己的建造者 (Builder 的实现类).
>
> 比如, 现有一个 Builder 接口, 然后 Bike 类内使用嵌套类 BikeBuilder 实现 Builder 接口, Car 类内部使用嵌套类 CarBuilder 实现 Builder 接口, 这样 Vehicle 和 Car 都拥有各自的建造者.

构造建造类来封装实例化复杂对象的逻辑, 符合单一职责原则和开闭原则, 当需要有不同结构的对象时,我们可以添加新的建造者类, 从而实现对修改关闭和对扩展开放

> 上文中的意思是, 可以在一个类内部添加新的建造者, 来扩展类的不同结构;
>
> 但个人认为, 在大部分开源源码设计中, 都未曾在同一个类中出现多个 Builder, 大都是一个类只用一个 Builder嵌套类来实例化类.

建造者模式中的角色:

- Produce (产品类): 需要为其构建对象的类, 使具有不同表现形式的复杂或复合对象
- Builder (抽象建造者类): 用于声明构建产品类的组成部分的抽象类或接口. ***它的作用是仅公开构建产品类的功能, 隐藏产品类的其他功能 ;*** 将产品类与构建产品类的更高级的类分离开
- ConcreateBuilder (具体的建造者类): 用于实现抽象建造者类或接口中声明的方法. 除此之外, 它还通过 getResult 方法返回构建好的产品类.



#### 3.1 实例

如果现在我们需要构建一个 Zookeeper 客户端, 需要的所有参数是 Zk 服务端地址(zkServers), 客户端会话超时时长(sessionTimeout, 默认10s), 连接超时时长 (connectionTimeout, 默认10s)这三个参数, 我们使用 Builder 模式来构建.

```java
/**
 * 这个属于真正的 zk客户端对象
 */
public class ZkCli{
  
  private final String zkServers;
	private final int sessionTimeout;
	private final int connectionTimeout;
  
  private ZkCli(ZkClientFactory.Builder builder) {
    this.zkServers = builder.getConnectString();
    this.sessionTimeout = builder.getSessionTimeoutMs();
    this.connectionTimeout = builder.getConnectionTimeoutMs();
  }
}
```



```java
/**
 * 构建客户端的工厂类
 */
public class ZkClientFactory {
	
  	private static final String LOCAL_ZK_SERVERS = "127.0.0.1:2181";
    private static final int DEFAULT_SESSION_TIMEOUT = 10_000;
    private static final int DEFAULT_CONNECTION_TIMEOUT = 10_000;


    public static ZkClientFactory.Builder builder(){
        return new Builder();
    }

    public static class Builder{
        private String zkServers = LOCAL_ZK_SERVERS;
        private int sessionTimeout = DEFAULT_SESSION_TIMEOUT;
        private int connectionTimeout = DEFAULT_CONNECTION_TIMEOUT;

        public ZkCli build(){
            return new ZkCli(this);
        }

        public Builder connectString(String zkServers){
            this.zkServers = zkServers;
            return this;
        }

        public Builder connectionTimeoutMs(int connectionTimeout){
            this.connectionTimeout = connectionTimeout;
            return this;
        }

        public Builder sessionTimeoutMs(int sessionTimeout){
            this.sessionTimeout = sessionTimeout;
            return this;
        }

        public String getConnectString(){
            return this.zkServers;
        }
        public int getConnectionTimeoutMs(){
            return this.connectionTimeout;
        }
        public int getSessionTimeoutMs(){
            return this.sessionTimeout;
        }

    }
}
```

接着我们使用 Fluent 风格来构建客户端:

```
ZkCli cli = ZkClientFactory.builder()
							.connectString("localhost:2181")
							.connectionTimeoutMs(20_000)
							.sessionTimeoutMs(60_000)
							.build;
```



### 4. 原型模式

原型模式看似复杂, 实际上它只是一种克隆对象的方法. 现在实例化对象操作并不特别好费性能, 那么为什么还要克隆对象呢? 在以下几种情况, 确实需要克隆那些已经实例化过的对象.

- 依赖于外部资源或硬件密集型操作进行对象创建的情况
- 获取相同对象在相同状态的拷贝而无需进行重复获取状态的操作的情况
- 在不确定所属具体类时需要对象的实例的情况