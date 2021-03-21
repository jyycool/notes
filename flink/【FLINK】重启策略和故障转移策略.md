## 【FLINK】重启策略和故障转移策略

### Overview

在fink中提交任务后，就不可避免的会有失败的情况，当一个task失败，我们需要重启失败的task和受到影响的task，来恢复job的运行到一个正常的state。

重新启动策略和故障转移策略用于控制任务重新启动。 

- 重启策略决定是否可以重启以及重启的间隔
- 故障恢复策略决定哪些 Task 需要重启

### restart-strategy

flink中提供了3中重启策略，这3种重启策略都有2种方式可以进行配置，一种是在配置文件中进行全局的配置，另外一种是在程序中，通过代码的形式进行配置。

通过 Flink 配置文件 flink-conf.yaml 来设置重启策略。配置参数 ***restart-strategy*** 定义了采取何种策略。 若没有启用 checkpoint，就采用“不重启( none )”策略。如果启用了 checkpoint 且没有配置重启策略，那么默认采用固定延时重启( fixed-delay )策略， 此时最大尝试重启次数由 ***Integer.MAX_VALUE*** 参数设置。下表列出了可用的重启策略和与其对应的配置值。

每个重启策略都有自己的一组配置参数来控制其行为。 这些参数也在配置文件中设置。 后文的描述中会详细介绍每种重启策略的配置项。

| 重启策略         | restart-strategy 配置值 |
| ---------------- | ----------------------- |
| 固定延时重启策略 | fixed-delay             |
| 故障率重启策略   | failure-rate            |
| 不重启策略       | none                    |

除了定义默认的重启策略以外，还可以为每个 Flink 作业单独定义重启策略。 这个重启策略通过在程序中的 ExecutionEnvironment 对象上调用 setRestartStrategy 方法来设置。 当然，对于 StreamExecutionEnvironment 也同样适用。

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // 尝试重启的次数
  Time.of(10, TimeUnit.SECONDS) // 延时
));
```



#### fixed-delay

***固定延时重启策略***按照给定的次数尝试重启作业。 如果尝试超过了给定的最大次数，作业将最终失败。 在连续的两次重启尝试之间，重启策略等待一段固定长度的时间。

通过在 `flink-conf.yaml` 中设置如下配置参数，默认启用此策略。

```yaml
restart-strategy: fixed-delay
```

| 配置参数                              | 描述                                                         | 默认配置值                                                   |
| ------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| restart-strategy.fixed-delay.attempts | 作业宣告失败之前 Flink 重试执行的最大次数                    | 启用 checkpoint 的话是 `Integer.MAX_VALUE`，否则是 1         |
| restart-strategy.fixed-delay.delay    | 延时重试意味着执行遭遇故障后，并不立即重新启动，而是延后一段时间。当程序与外部系统有交互时延时重试可能会有所帮助，比如程序里有连接或者挂起的事务的话，在尝试重新执行之前应该等待连接或者挂起的事务超时。 | 启用 checkpoint 的话是 10 秒，否则使用 `akka.ask.timeout` 的值 |

固定延迟重启策略也可以在程序中设置：

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // 尝试重启的次数
  Time.of(10, TimeUnit.SECONDS) // 延时
));
```

#### failure-rate

故障率重启策略在故障发生之后重启作业，但是当**故障率**（每个时间间隔发生故障的次数）超过设定的限制时，作业会最终失败。 在连续的两次重启尝试之间，重启策略等待一段固定长度的时间。

通过在 `flink-conf.yaml` 中设置如下配置参数，默认启用此策略。

```yaml
restart-strategy: failure-rate
 	 	
```

| 配置参数                                                | 描述                             | 配置默认值       |
| ------------------------------------------------------- | -------------------------------- | ---------------- |
| restart-strategy.failure-rate.max-failures-per-interval | 单个时间间隔内允许的最大重启次数 | 1                |
| restart-strategy.failure-rate.failure-rate-interval     | 测量故障率的时间间隔             | 1 分钟           |
| restart-strategy.failure-rate.delay                     | 连续两次重启尝试之间的延时       | akka.ask.timeout |

例如：

```yaml
restart-strategy.failure-rate.max-failures-per-interval: 3
restart-strategy.failure-rate.failure-rate-interval: 5 min
restart-strategy.failure-rate.delay: 10 s
```

故障率重启策略也可以在程序中设置：

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // 每个时间间隔的最大故障次数
  Time.of(5, TimeUnit.MINUTES), // 测量故障率的时间间隔
  Time.of(10, TimeUnit.SECONDS) // 延时
));
```

#### none

作业直接失败，不尝试重启。

```yaml
restart-strategy: none
```

不重启策略也可以在程序中设置：

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
```

#### 备用重启策略

使用群集定义的重启策略。 这对于启用了 checkpoint 的流处理程序很有帮助。 如果没有定义其他重启策略，默认选择固定延时重启策略。

### failover-strategy

Flink支持不同的故障转移策略，可以通过Flink的配置文件flink-conf.yaml中的配置参数 ***jobmanager.execution.failover-strategy*** 对其进行配置。

| 故障恢复策略           | jobmanager.execution.failover-strategy 配置值 |
| ---------------------- | --------------------------------------------- |
| 全图重启               | full                                          |
| 基于 Region 的局部重启 | region                                        |

#### 重启所有（full）

这种策略重新启动作业中的所有任务以从任务失败中恢复

#### 重启流水线区域故障转移策略（region）

这种战略把任务分成互不相连的区域。当检测到任务失败时，此策略重新计算最小的region集以从故障中恢复，对于某些作业，与Restart All Failover策略相比，这会重新启动的任务更少

一个region是一个task的集合，这些task的通信是通过pipeline方式进行数据交换

- 在Datastream作业，流式的表或SQL作业中的所有数据交换都是流水线的
- 默认情况下，批处理表/SQL作业中的所有数据交换都是批处理的。
- DataSet作业中的数据交换类型由可通过ExecutionConfig设置的ExecutionMode确定

定义重启的region：

- 包含失败任务的区域将重新启动
- 如果结果的分区不可用，而将被重新启动的区域需要它，则产生结果分区的区域也将重新启动
- 如果要重新启动某个区域，则该区域的所有消费者区域也将重新启动。这是为了保证数据一致性，因为不确定性处理或分区可能导致不同的分区

### 源码

#### **RestartStrategies**

这个类是重启策略的基础类，这个类里面提供了各种重启策略的静态方法，用于返回相关的RestartStrategyConfiguration ，也就是返回重启策略配置类。

```java
// 不重启
public static RestartStrategyConfiguration noRestart() {
  return new NoRestartStrategyConfiguration();
}
// 根据集群的重启策略
public static RestartStrategyConfiguration fallBackRestart() {
  return new FallbackRestartStrategyConfiguration();
}
//固定延迟重启
public static RestartStrategyConfiguration fixedDelayRestart(int restartAttempts, long delayBetweenAttempts) {
  return fixedDelayRestart(restartAttempts,
  Time.of(delayBetweenAttempts, TimeUnit.MILLISECONDS));
}
// 失败率重启策略
public static FailureRateRestartStrategyConfiguration failureRateRestart(
  int failureRate, Time failureInterval, Time delayInterval) {
  return new FailureRateRestartStrategyConfiguration(failureRate, failureInterval, delayInterval);
}
```

**RestartStrategyConfiguration**这个类是重启策略配置的基础抽象类，这个类中只有一个方法，就是获取描述的方法

```java
public abstract static class RestartStrategyConfiguration implements Serializable {
	private static final long serialVersionUID = 6285853591578313960L;

	private RestartStrategyConfiguration() {}

  /**
	 * Returns a description which is shown in the web interface.
	 *
	 * @return Description of the restart strategy
	 */
  public abstract String getDescription();
}
```

他有4个实现类,每个实现类描述具体的重启策略相关信息

**RestartStrategyConfiguration**

- FallbackRestartStrategyConfiguration in RestartStrategies
- FailureRateRestartStrategyConfiguration in RestartStrategies
- NoRestartStrategyConfiguration in RestartStrategies
- FixedDelayRestartStrategyConfiguration in RestartStrategies

##### RestartStrategyResolving

供了一个静态方法resolve，用于解析RestartStrategies.RestartStrategyConfiguration，然后使用RestartStrategyFactory创建RestartStrategy.

```java
public static RestartStrategy resolve(RestartStrategies.RestartStrategyConfiguration 					clientConfiguration, RestartStrategyFactory serverStrategyFactory,
boolean isCheckpointingEnabled) {
  final RestartStrategy clientSideRestartStrategy =
		RestartStrategyFactory.createRestartStrategy(clientConfiguration);
...}
```

##### RestartStrategy

这是一个接口，里面有两个方法

```java
// 是否能够重启
boolean canRestart();
// 重启
void restart(RestartCallback restarter, ScheduledExecutor executor);
```

定义了canRestart及restart两个方法，它有NoRestartStrategy、FixedDelayRestartStrategy、FailureRateRestartStrategy这几个子类

***RestartStrategy***

- FailureRateRestartStartegy
- FixedDelayRestartStartegy
- NoRestartStartegy

这三个实现类正好本文开头的三种重启策略相对应。

###### NoRestartStrategy

```java
public class NoRestartStrategy implements RestartStrategy {

	@Override
	public boolean canRestart() {
		return false;
	}

	@Override
	public void restart(RestartCallback restarter, ScheduledExecutor executor) {
		throw new UnsupportedOperationException("NoRestartStrategy does not support restart.");
	}
...
}
```

这个实现类实现了RestartStrategy 的两个抽象方法

###### FixedDelayRestartStrategy

这个是固定延时重启策略的实现类，

```java
@Override
	public boolean canRestart() {
		return currentRestartAttempt < maxNumberRestartAttempts;
	}

	@Override
	public void restart(final RestartCallback restarter, ScheduledExecutor executor) {
		currentRestartAttempt++;
		executor.schedule(restarter::triggerFullRecovery, delayBetweenRestartAttempts, TimeUnit.MILLISECONDS);
	}
```

实现了接口的方法，restart方法则直接调用ScheduledExecutor.schedule方法，延时delayBetweenRestartAttempts毫秒执行RestartCallback.triggerFullRecovery()

###### FailureRateRestartStrategy

失败率重启策略

```java
@Override
	public boolean canRestart() {
		if (isRestartTimestampsQueueFull()) {
			Long now = System.currentTimeMillis();
			Long earliestFailure = restartTimestampsDeque.peek();

			return (now - earliestFailure) > failuresInterval.toMilliseconds();
		} else {
			return true;
		}
	}

	@Override
	public void restart(final RestartCallback restarter, ScheduledExecutor executor) {
		if (isRestartTimestampsQueueFull()) {
			restartTimestampsDeque.remove();
		}
		restartTimestampsDeque.add(System.currentTimeMillis());

		executor.schedule(new Runnable() {
			@Override
			public void run() {
				restarter.triggerFullRecovery();
			}
		}, delayInterval.getSize(), delayInterval.getUnit());
	}
```

canRestart方法在restartTimestampsDeque队列大小小于maxFailuresPerInterval时返回true，大于等于maxFailuresPerInterval时则判断当前时间距离earliestFailure是否大于failuresInterval；

restart方法则往restartTimestampsDeque添加当前时间，然后调用ScheduledExecutor.schedule方法，延时delayInterval执行RestartCallback.triggerFullRecovery()