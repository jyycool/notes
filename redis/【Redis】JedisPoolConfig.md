# 【Redis】JedisPoolConfig

`JedisPoolConfig`中可以能够配置的参数有很多，连接池实现依赖 apache 的`commons-pool2`。上面源码也大致列举了一些配置参数，下面在详细说明一下。

**把池理解为工厂，池中的实例理解为工人**，如下图，这样池中的很多参数理解起来就比较容易了。

![null](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy95emlhYk43cEsyWWp3OWlhVU4wYnJpYWxOZGNPWll0Y1c5NVBkQjlUbFJ3R2ZjTUZET2tCUVE1VVl5NHRiQjJraG1GcjRNeW9LZEZOUThackFnTGljaWNHeldBLzY0MA?x-oss-process=image/format,png)

Jedis连接就是连接池中JedisPool管理的资源，JedisPool保证资源在一个可控范围内，并且保障线程安全。使用合理的**GenericObjectPoolConfig**配置能够提升Redis的服务性能，降低资源开销。下列两表将对一些重要参数进行说明，并提供设置建议。

| 参数               | 说明                                                         | 默认值             | 建议                                                |
| ------------------ | ------------------------------------------------------------ | ------------------ | --------------------------------------------------- |
| maxTotal           | 资源池中的最大连接数                                         | 8                  | 参见**关键参数设置建议**                            |
| maxIdle            | 资源池允许的最大空闲连接数                                   | 8                  | 参见**关键参数设置建议**                            |
| minIdle            | 资源池确保的最少空闲连接数                                   | 0                  | 参见**关键参数设置建议**                            |
| blockWhenExhausted | 当资源池用尽后，调用者是否要等待。只有当值为true时，下面的**maxWaitMillis**才会生效。 | true               | 建议使用默认值。                                    |
| maxWaitMillis      | 当资源池连接用尽后，调用者的最大等待时间（单位为毫秒）。     | -1（表示永不超时） | 不建议使用默认值。                                  |
| testOnBorrow       | 向资源池借用连接时是否做连接有效性检测（ping）。检测到的无效连接将会被移除。 | false              | 业务量很大时候建议设置为false，减少一次ping的开销。 |
| testOnReturn       | 向资源池归还连接时是否做连接有效性检测（ping）。检测到无效连接将会被移除。 | false              | 业务量很大时候建议设置为false，减少一次ping的开销。 |
| jmxEnabled         | 是否开启JMX监控                                              | true               | 建议开启，请注意应用本身也需要开启。                |

空闲Jedis对象检测由下列四个参数组合完成，**testWhileIdle**是该功能的开关。

| 名称                          | 说明                                                         | 默认值             | 建议                                                         |
| ----------------------------- | ------------------------------------------------------------ | ------------------ | ------------------------------------------------------------ |
| testWhileIdle                 | 是否开启空闲资源检测。                                       | false              | true                                                         |
| timeBetweenEvictionRunsMillis | 空闲资源的检测周期（单位为毫秒）                             | -1（不检测）       | 建议设置，周期自行选择，也可以默认也可以使用下方**JedisPoolConfig** 中的配置。 |
| minEvictableIdleTimeMillis    | 资源池中资源的最小空闲时间（单位为毫秒），达到此值后空闲资源将被移除。 | 180000（即30分钟） | 可根据自身业务决定，一般默认值即可，也可以考虑使用下方**JeidsPoolConfig**中的配置。 |
| numTestsPerEvictionRun        | 做空闲资源检测时，每次检测资源的个数。                       | 3                  | 可根据自身应用连接数进行微调，如果设置为 -1，就是对所有连接做空闲监测。 |

**说明** 可以在**org.apache.commons.pool2.impl.BaseObjectPoolConfig**中查看全部默认值。

## 一、关键参数设置建议

### 1.1 maxTotal(最大连接数)

想合理设置**maxTotal**（最大连接数）需要考虑的因素较多，如：

- 业务希望的Redis并发量；
- 客户端执行命令时间；
- Redis资源，例如 nodes(如应用个数等) * maxTotal不能超过 Redis 的最大连接数；
- 资源开销，例如虽然希望控制空闲连接，但又不希望因为连接池中频繁地释放和创建连接造成不必要的开销。

假设一次命令时间，即borrow|return resource加上Jedis执行命令 (含网络耗时) 的平均耗时约为1ms，一个连接的QPS大约是1000，业务期望的QPS是50000，那么理论上需要的资源池大小是50000 / 1000 = 50。

但事实上这只是个理论值，除此之外还要预留一些资源，所以**maxTotal**可以比理论值大一些。这个值不是越大越好，一方面连接太多会占用客户端和服务端资源，另一方面对于Redis这种高QPS的服务器，如果出现大命令的阻塞，即使设置再大的资源池也无济于事。

### 1.2 maxIdle与minIdle

**maxIdle** 实际上才是业务需要的最大连接数，**maxTotal** 是为了给出余量，所以 **maxIdle** 不要设置得过小，否则会有`new Jedis`（新连接）开销，而**minIdle**是为了控制空闲资源检测。

连接池的最佳性能是**maxTotal**=**maxIdle**，这样就避免了连接池伸缩带来的性能干扰。但如果并发量不大或者**maxTotal**设置过高，则会导致不必要的连接资源浪费。

您可以根据实际总QPS和调用Redis的客户端规模整体评估每个节点所使用的连接池大小。

1.3 使用监控获取合理值

在实际环境中，比较可靠的方法是通过监控来尝试获取参数的最佳值。可以考虑通过JMX等方式实现监控，从而找到合理值。

上面参数配置：JedisPool资源池优化[1]

**创建`JedisPool`代码**

```java
// volatile 修饰
private static volatile JedisPool jedisPool = null;

private JedisPoolUtils(){}

public static JedisPool getJedisPoolInstance() {
  // 使用双重检查创建单例
  if(null == jedisPool) {
    synchronized (JedisPoolUtils.class) {
      if(null == jedisPool) {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(10);
        poolConfig.setMaxIdle(10);
        poolConfig.setMinIdle(2);
        poolConfig.setMaxWaitMillis(30*1000);
        poolConfig.setTestOnBorrow(true);
        poolConfig.setTestOnReturn(true);
        poolConfig.setTimeBetweenEvictionRunsMillis(10*1000);
        poolConfig.setMinEvictableIdleTimeMillis(30*1000);
        poolConfig.setNumTestsPerEvictionRun(-1);
        jedisPool = new JedisPool(poolConfig,"localhost",6379);
      }
    }
  }
  return jedisPool;
}
```

### 实例创建和释放大致流程解析

![null](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy95emlhYk43cEsyWWp3OWlhVU4wYnJpYWxOZGNPWll0Y1c5NWFOdjZnODhJUmFsaWJpYVFncmRpYUY5OWlhTHNDTEJlc09sYVJBQWlhbktzV2JPbDBpYjN1QUxtd2pQZy82NDA?x-oss-process=image/format,png)

### 根据流程进行源码解析

### 创建过程

使用`pool.getResource()`进行Jedis实例的创建。

```java
//org.apache.commons.pool2.impl.GenericObjectPool#borrowObject(long)



public T borrowObject(final long borrowMaxWaitMillis) throws Exception {



 



 



    final boolean blockWhenExhausted = getBlockWhenExhausted();



 



 



    PooledObject<T> p = null;



    boolean create;



    final long waitTime = System.currentTimeMillis();



 



 



    while (p == null) {



        create = false;



        // 从空闲队列中获取



        p = idleObjects.pollFirst();



        if (p == null) {



            // 创建实例



            p = create();



            if (p != null) {



                create = true;



            }



        }



        // 吃资源是否耗尽



        if (blockWhenExhausted) {



            if (p == null) {



                // 等待时间小于0



                if (borrowMaxWaitMillis < 0) {



                    p = idleObjects.takeFirst();



                } else {



                    p = idleObjects.pollFirst(borrowMaxWaitMillis,



                                              TimeUnit.MILLISECONDS);



                }



            }



            if (p == null) {



                throw new NoSuchElementException(



                    "Timeout waiting for idle object");



            }



        } else {



            if (p == null) {



                throw new NoSuchElementException("Pool exhausted");



            }



        }



        if (!p.allocate()) {



            p = null;



        }



 



 



        if (p != null) {



            try {



                // 重新初始化要由池返回的实例。



                factory.activateObject(p);



            } catch (final Exception e) {



 



 



            }



        }



    }



    updateStatsBorrow(p, System.currentTimeMillis() - waitTime);



 



 



    return p.getObject();



}



 



 
```

### 释放过程

从Jedis3.0版本后`pool.returnResource()`遭弃用,官方重写了Jedis的close方法用以代替；官方建议应用redis.clients.jedis#Jedis的close方法进行资源回收，官方代码如下：

```java
 @Override



  public void close() {



    if (dataSource != null) {



      JedisPoolAbstract pool = this.dataSource;



      this.dataSource = null;



      if (client.isBroken()) {



        pool.returnBrokenResource(this);



      } else {



        pool.returnResource(this);



      }



    } else {



      super.close();



    }



  }
```

这里主要看：`pool.returnResource(this);`

```java
//org.apache.commons.pool2.impl.GenericObjectPool#returnObject



public void returnObject(final T obj) {



    // 获取要释放的实例对象



    final PooledObject<T> p = allObjects.get(new IdentityWrapper<>(obj));



 



 



    if (p == null) {



        if (!isAbandonedConfig()) {



            throw new IllegalStateException(



                "Returned object not currently part of this pool");



        }



        return; // Object was abandoned and removed



    }



    // 将对象标记为返回池的状态。



    markReturningState(p);



 



 



    final long activeTime = p.getActiveTimeMillis();



 



 



    // 这里就和上面配置的参数有关系，释放的时候是否做连接有效性检测（ping）



    if (getTestOnReturn() && !factory.validateObject(p)) {



        try {



            destroy(p);



        } catch (final Exception e) {



            swallowException(e);



        }



        try {



            ensureIdle(1, false);



        } catch (final Exception e) {



            swallowException(e);



        }



        updateStatsReturn(activeTime);



        return;



    }



 



 



    // 检查空闲对象，如果最大空闲对象数小于当前idleObjects大小，则销毁



    final int maxIdleSave = getMaxIdle();



    if (isClosed() || maxIdleSave > -1 && maxIdleSave <= idleObjects.size()) {



        try {



            destroy(p);



        } catch (final Exception e) {



            swallowException(e);



        }



    } else {



        // 否则加入到空闲队列中，空闲队列是一个双端队列



        // getLifo 也和配置的参数有关，默认True



        if (getLifo()) {



            // last in first out，加到队头



            idleObjects.addFirst(p);



        } else {



            // first in first out ，加到队尾



            idleObjects.addLast(p);



        }



    }



    updateStatsReturn(activeTime);



}
```

上面创建和释放删除了一些代码，具体完整代码都是在`GenericObjectPool`类中。