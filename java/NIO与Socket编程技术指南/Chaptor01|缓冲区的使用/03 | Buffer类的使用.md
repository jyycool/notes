# 03 | Buffer类的使用

Buffer 类的核心 API 列表如下所示:

```
Buffer:
	array(): Object
	arrayOffset(): int
	capacity(): int
	clear(): Buffer
	flip(): Buffer
	hasArray(): boolean
	hasRemaining(): boolean
	isDirect(): boolean
	limit(): int
	limit(int): Buffer
	mark(): Buffer
	position(): int
	position(int): Buffer
	remaining(): int
	reset(): Buffer
	rewind(): Buffer
```

它的 7 个子类的声明信息如下:

```java
public abstract class ByteBuffer extends Buffer
public abstract class CharBuffer extends Buffer
public abstract class DoubleBuffer extends Buffer
public abstract class FloatBuffer extends Buffer
public abstract class ShortBuffer extends Buffer
public abstract class IntBuffer extends Buffer
public abstract class LongBuffer extends Buffer
```

抽象类 Buffer 的 7 个子类也是抽象类, 这就意味着这 7 个子类不能直接 new 实例化, 所以使用方式是将上面 7 种数据类型的数组包装(wrap)进缓冲区, 此时就需要借助静态方法 wrap() 进行实现。wrap() 方法的作用是将数组放入缓冲区中, 来构建存储不同数据类型的缓冲区。

> Tips: 缓冲区是非线程安全的

### 3.1 包装数据与获得容量

在 NIO 技术的缓冲区中, 存在 4 个核心技术点, 分别是:

- capacity(容量)
- limit(限制)
- position(位置)
- mark(标记)

这 4 个技术点之间的值的大小关系如下:

```
0 <= mark <= position <= limit <= capacity
```

首先介绍一下 capacity, 它代表元素的数量。缓冲区的 capacity不能为负数, 并且 capacity 也不能更改。

int capacity() 方法的作用是: 返回此缓冲区的容量。

capacity 代表着缓冲区的大小, 其本质就是内部数组的 length。

### 3.2 限制获取与设置

方法 limit() 的作用: 返回此缓冲区的限制。

方法 Buffer limit(int newLimit) 的作用: 设置此缓冲区的限制。

什么是限制? 它代表了缓冲区中第一个不应该读取或写入的元素的 index。此限制(limit)不能为负, 并且 limit 不能大于 capacity。如果 position 大于新的 limit, 则将 position设置为新的 limit。如果 mark 已定义且大于新的 limit, 则丢弃该 mark。