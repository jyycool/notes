# 【Java8】异步编程 CompletableFuture



Java 为并发编程提供了众多的工具，本文将重点介绍 Java8 中 CompletableFuture。

笔者在自己搜索资料及实践之后，避开已经存在的优秀文章的写作内容与思路，将以更加浅显的示例和语言，介绍 CompleatableFuture， 同时提供自己的思考。

本文最后会附上其他优秀的文章链接供读者进行更详细学习与理解。

## 1. 理解 Future

当处理一个任务时，总会遇到以下几个阶段：

1. 提交任务
2. 执行任务
3. 任务完成的后置处理

以下我们简单定义，构造及提交任务的线程为生产者线程； 执行任务的线程为消费者线程； 任务的后置处理线程为后置消费者线程。

根据任务的特性，会衍生各种各样的线程模型。其中包括 Future 模式。

接下来我们先用最简单的例子迅速对 Future 有个直观理解，然后再对其展开讨论。

```java
ExecutorService executor = Executors.newFixedThreadPool(3);
Future future = executor.submit(new Callable<String>() {
	@Override
	public String call() throws Exception {
 		//do some thing
 		Thread.sleep(100);
 		return "i am ok";
  }
});

println(future.isDone());
println(future.get());
```

在本例中首先创建一个线程池，然后向线程池中提交了一个任务， submit 提交任务后会被立即返回，而不会等到任务实际处理完成才会返回，  而任务提交后返回值便是 Future， 通过 Future 我们可以调用 get() 方法阻塞式的获取返回结果， 也可以使用 isDone  获取任务是否完成

生产者线程在提交完任务后，有两个选择：关注处理结果和不关注处理结果。处理结果包括任务的返回值，也包含任务是否正确完成，中途是否抛出异常等等。

Future 模式提供一种机制，在消费者异步处理生产者提交的任务的情况下，生产者线程也可以拿到消费者线程的处理结果，同时通过 Future 也可以取消掉处理中的任务。

在实际的开发中，我们经常会遇到这种类似需求。任务需要异步处理，同时又关心任务的处理结果，此时使用 Future 是再合适不过了。

### 1.2 Future 如何被构建的

Future 是如何被创建的呢?  生产者线程提交给消费者线程池任务时，线程池会构造一个实现了 Future 接口的对象 FutureTask  。该对象相当于是消费者和生产者的桥梁，消费者通过 FutureTask  存储任务的处理结果，更新任务的状态：未开始、正在处理、已完成等。而生产者拿到的 FutureTask 被转型为 Future  接口，可以阻塞式获取任务的处理结果，非阻塞式获取任务处理状态。

更细节的实现机制，读者可以参考 JDK 中的 FutureTask 类。

