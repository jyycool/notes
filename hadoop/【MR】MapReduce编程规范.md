## 【MR】MapReduce编程规范

### Hadoop 中的数据类型

Hadoop数据类型对应Java数据类型

| Java数据类型 | Hadoop数据类型  | 序列化大小 (字节) |
| ------------ | --------------- | ----------------- |
| boolean      | BooleanWritable | 1                 |
| 整型         | ByteWritable    | 1                 |
| 整型         | IntWritable     | 4                 |
|              | VintWritable    | 1~5               |
| long         | LongWritable    | 8                 |
|              | VlongWritable   | 1~9               |
| float        | FloatWritable   | 4                 |
| double       | DoubleWritable  | 8                 |
| String       | Text            |                   |
| Array        | ArrayWritable   |                   |


hadoop类型与Java数据类型之间的转换有两种方式

1. 通过set方式
2. 通过new的方式



### Mapper阶段

1. 用户自定义的 Mapper 需要继承 ***org.apache.hadoop.mapreduce.Mapper***
2. Mapper 输入的数据是<K, V>的形式, <K, V>类型用户可以自己定义
3. Mapper 中的业务逻辑写在 map() 方法中
4. Mapper 输出的数据是<K, V>的形式, <K, V>类型用户可以自己定义
5. map() 方法( MapTask进程 )对每个<K, V> 调用一次



### Reducer阶段

1. 用户自定义的 Reducer 需要继承 ***org.apache.hadoop.mapreduce.Reducer***
2. Reducer 的输入数据类型对应 Mapper 输出的数据类型<K, V>, <K, V>类型用户可以自己定义
3. Reducer 中的业务逻辑写在 reduce() 方法中
4. reduce() 方法( ReduceTask进程 )对每一组相同K的<K, V> 调用一次

