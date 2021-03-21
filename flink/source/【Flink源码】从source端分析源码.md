# 【Flink源码】从source端分析源码

Flink 程序开始时的标准 code 是:

```scala
object Main {
	
	def main(args: Array[String]): Unit = {
		
		val sEnv: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
	
		val source: DataStreamSource[String] = sEnv.socketTextStream("localhost", 9999)
		
    // 暂时省略
		
	}
	
}
```

OK, 今天就从这个`socketTextStream`处来一步步跟进分析.

首先一层层跟进`socketTextStream`

```scala
 StreamExecutionEnvironment.socketTextStream(String hostname, int port)

--> 
StreamExecutionEnvironment.socketTextStream(String hostname, int port, String delimiter)

--> 
StreamExecutionEnvironment.socketTextStream(String hostname, int port, String delimiter, long maxRetry)

--> 
StreamExecutionEnvironment.addSource(SourceFunction<OUT> function, String sourceName)
-->
public <OUT> DataStreamSource<OUT> addSource(
  			SourceFunction<OUT> function, 
  			String sourceName, 
  			TypeInformation<OUT> typeInfo) {

	TypeInformation<OUT> resolvedTypeInfo = getTypeInfo(
      	function, 
      	sourceName, 
      	SourceFunction.class, 
      	typeInfo);

	boolean isParallel = function instanceof ParallelSourceFunction;

	clean(function);

	final StreamSource<OUT, ?> sourceOperator = 
  			new StreamSource<>(function);
		
  return new DataStreamSource<>(
    		this, 
    		resolvedTypeInfo, 
    		sourceOperator, 
    		isParallel, 
    		sourceName);
}
```

用户传递的 hostname, port参数被构造成了 

`SocketTextStreamFunction(hostname, port, delimiter, maxRetry),"Socket Stream")`它是`SourceFunction`的实现类, 读取源头数据底层都是使用的`addSource`方法

