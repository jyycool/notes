## 【FLINK】解析Window机制

### 1. 了解 Window

带着 4 个问题去剖析 Flink 中的 Window 机制

1. window 是为了解决流式计算中的什么问题 ?
2. 怎么划分 window ? 有哪几种 window ? window 和时间属性的关系 ?
3. window 中的数据何时被计算 ?
4. window 何时被清除 ?

> - Q1: window 是为了解决流式计算中的什么问题 ?
>
>   熟悉 Google Dataflow 模型的同学都知道, 流计算被抽象成四个问题, what ? where ? when ? how ?
>
>   所以 window 其实就是为了解决 where, 也就是将无界流数据(unbounded dataflow) 划分成有界流数据(bounded flow)
>
> 
>
> - Q2: 怎么划分 window ? 有哪几种 window ? window 和时间属性的关系 ?
>
>   1. 时间属性
>
>      - EventTime: 事件发生时间, 它和我们生活中当前的时钟时间无任何关系, 它只是标注了一个事件发生/一条数据产生时的时间, 这个时间可以是当下也可以是过去
>      - IngestionTime: 一条数据产生并进入 SourceFunction 的时间, 这个时间就是发生前面动作时的机器时间, 这个时间一定是当下的
>      - ProcessingTime: 处理时间, 一条数据进入 Window Operator 的时间, 这个时间属性和上面的 IngestionTime 一样, 一定是当下的机器时间
>
>   2. window 的划分
>
>      当一条数据到达 window operator 的时候, 会根据当前不同时间属性调用不同的 WindowAssingner 对象方法将该条数据分配到一个或多个窗口
>
>      当前 Flink 中的窗口有三种, 每种又可以基于 EventTime 和 ProcessingTime 再分为 2 种
>
>      - 滚动窗口(可以基于 EventTime/ProcessiongTime): TumblingWindow
>      - 滑动窗口(可以基于 EventTime/ProcessiongTime): SlidingWindow
>      - 会话窗口: SessionWindow
>
> 
>
> - Q3: window 中的数据何时被计算 ?
>
>   结局这个问题的方法时 WaterMark 和 Trigger
>
>   WaterMark 是用来标记窗口的完整性, 它是数据本身的一个隐藏属性
>
>   Trigger 的实现是当 WaterMark 处于某种时间属性下或者窗口内数据达到一定条件, 窗口开始计算.
>
>   在 Flink 中 Trigger 最常见的实现方式是: 当 WaterMark 越界时(window结束时间), 开始触发窗口内数据的计算, 而 WaterMark 和数据本身一样作为正常的消息在流动
>
> 
>
> - Q4: window 何时被清除 ?
>
>   如果 window 一直存在,肯定会造成不必要的内存和磁盘空间的浪费, 那么 window 是何时被清除的呢 ? 
>
>   每一个 window 都有一个 Trigger, Trigger 上会有一个定时器 cleanTime, cleanTime代表了这个 window 的存活时间, `cleanTime = windowEndTime + 窗口允许的延迟(allowLateness)`
>
>   所以当 WaterMark > cleanTime 的时候, 该窗口会被清除, 对应的状态也会被清除

### 2. 剖析 Window API

得益于 Flink window API 的松耦合的设计, 我们可以非常灵活的定义符合特定业务的窗口. Flink 中定义一个窗口需要以下三个组件

- WindowAssigner: 用来决定数据被分配到哪个/哪些窗口中去

  下图是 Flink-1.10 内置实现的 WindowAssigner

  ![WindowAssigner](/Users/sherlock/Desktop/notes/allPics/Flink/WindowAssigner.png)



- Trigger: 触发器, 决定了一个窗口何时能被计算或清除, 每个窗口都拥有自己的 Trigger

  下图是 Flink-1.10 内置实现的 Trigger

  ![Trigger](/Users/sherlock/Desktop/notes/allPics/Flink/Trigger.png)



- Evictor: 可以译为"驱逐者", 在 Trigger 触发后, 在窗口被处理之前, Evictor (如果有的话) 会用来剔除窗口中不需要的元素, 相当于一个 Filter.

  下图是 Flink-1.10 内置实现的 Evictor

  ![Evictor](/Users/sherlock/Desktop/notes/allPics/Flink/Evictor.png)



以上三个组件的不同实现, 可以组合定义出非常复杂的窗口. Flink 中内置的窗口也都是基于这三种组件构成的, 当然内置窗口无法解决用户特殊需求的时候, 用户可以基于 Flink 提供的这些窗口机制的内部接口来实现自定义的窗口. 下面我们将基于这三者来探讨窗口的实现机制



### 3. Window 的实现

下图描述了 Flink 的窗口机制以及各组件之间是如何相互工作的

![window实现](/Users/sherlock/Desktop/notes/allPics/Flink/window的实现.jpeg)

首先上图中的组件都位于一个算子（window operator）中，数据流源源不断地进入算子，每一个到达的元素都会被交给 WindowAssigner。WindowAssigner 会决定元素被放到哪个或哪些窗口（window），可能会创建新窗口。因为一个元素可以被放入多个窗口中，所以同时存在多个窗口是可能的。注意，`Window`本身只是一个ID标识符，其内部可能存储了一些元数据，如`TimeWindow`中有开始和结束时间，但是并不会存储窗口中的元素。窗口中的元素实际存储在 Key/Value State 中，key为`数据 key + Window`，`value为元素集合（或聚合值）`。为了保证窗口的容错性，该实现依赖了 Flink 的 State 机制（参见 [state 文档](https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/state.html)）。而在 StateBackend 中存放的数据格式为<Key, value>, 



每一个窗口都拥有一个属于自己的 Trigger，Trigger上会有定时器，用来决定一个窗口何时能够被计算或清除。每当有元素加入到该窗口，或者之前注册的定时器超时了，那么Trigger都会被调用。Trigger的返回结果可以是 continue（不做任何操作），fire（处理窗口数据），purge（移除窗口和窗口中的数据），或者 fire + purge。一个Trigger的调用结果只是fire的话，那么会计算窗口并保留窗口原样，也就是说窗口中的数据仍然保留不变，等待下次Trigger fire的时候再次执行计算。一个窗口可以被重复计算多次知道它被 purge 了。在purge之前，窗口会一直占用着内存。

当Trigger fire了，窗口中的元素集合就会交给`Evictor`（如果指定了的话）。Evictor 主要用来遍历窗口中的元素列表，并决定最先进入窗口的多少个元素需要被移除。剩余的元素会交给用户指定的函数进行窗口的计算。如果没有 Evictor 的话，窗口中的所有元素会一起交给函数进行计算。

计算函数收到了窗口的元素（可能经过了 Evictor 的过滤），并计算出窗口的结果值，并发送给下游。窗口的结果值可以是一个也可以是多个。DataStream API 上可以接收不同类型的计算函数，包括预定义的`sum()`,`min()`,`max()`，还有 `ReduceFunction`，`FoldFunction`，还有`WindowFunction`。WindowFunction 是最通用的计算函数，其他的预定义的函数基本都是基于该函数实现的。

Flink 对于一些聚合类的窗口计算（如sum,min）做了优化，因为聚合类的计算不需要将窗口中的所有数据都保存下来，只需要保存一个result值就可以了。每个进入窗口的元素都会执行一次聚合函数并修改result值。这样可以大大降低内存的消耗并提升性能。但是如果用户定义了 Evictor，则不会启用对聚合窗口的优化，因为 Evictor 需要遍历窗口中的所有元素，必须要将窗口中所有元素都存下来。

