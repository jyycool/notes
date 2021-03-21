## Flink源码阅读--ClosureCleaner

查看flink源码时,发现api中有clean()方法, 并且在streamExecutionEnvironment.getConfig().disableClosureCleaner()也有这个方法

```java
public <K> KeyedStream<T, K> keyBy(KeySelector<T, K> key) {
		Preconditions.checkNotNull(key);
		return new KeyedStream<>(this, clean(key));
}
```

```java
/**
	* Disables the ClosureCleaner.
  *
	* @see #enableClosureCleaner()
	*/
public ExecutionConfig disableClosureCleaner() {
  this.closureCleanerLevel = ClosureCleanerLevel.NONE;
  return this;
}
```

通过一番百度后[FLINK-1325](https://issues.apache.org/jira/browse/FLINK-1325)
在上面的网址找到了答案

> I suggest to add a closure cleaner that uses an ASM visitor over the function’s code to see if there is any access to the this$0 field. In case there is non, the field should be set to null.

主要是针对内部类的，默认内部类会持有一个外部对象的引用，如果外部对象不实现序列化接口,内部类的序列化会失败,需要将内部类指向外部类的引用置为null



为了验证这一点，写了一段测试代码

```JAVA
//Inner是内部类,实现序列化接口,但Outer没有实现
public class Outer {
	public class Inner implements Serializable{
	}
  
  //main方法中尝试序列化inner对象
  public static void main(String[] args){
  	Outer outer=new Outer();
    Outer.Inner inner=outer.new Inner();
    //下面这行,注释掉会报错,取消注释不会报错
    //ClosureCleaner.clean(inner,false);
    ClosureCleaner.ensureSerializable(inner);
  }
}
```

ClosureCleaner.clean()方法源码，关键代码是

```java
for (Field f: cls.getDeclaredFields()) {
		if (f.getName().startsWith("this$")) {
			// found a closure referencing field - now try to clean
			closureAccessed |= cleanThis0(func, cls, f.getName());
		}
}
```

> this$0: 本类对象
>
> this$1: 本类的内部类对象
>
> this$2: 本类的内部类的内部类对象
>
> this$n: 本类的第n个嵌套的内部类对象

结论很清晰, ClosureCleaner主要解决的就是内部类的序列化问题, 将**外部对象**的引用置为Null，如果外部对象可以被序列化，也就不需要这个类了。