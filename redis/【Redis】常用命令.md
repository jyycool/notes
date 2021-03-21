# 【Redis】常用命令

#redis

Redis支持5种数据类型, 分别是***String, Hash, List, Set, ZSet, Hyperloglog*** 

### keys

1. **keys **

   获取所有的key

2. ***select <n>*** 

   选择第n个库

3. ***move <key> <n>*** 

   将当前的数据库的key移动到第n个数据库, 目标库没有则不能移动

4. ***flushdb***      

   清楚所有数据

5. ***randomkey***     

   随机key

6. ***type <key>***      

   返回key的类型

7. ***set <key> <value>***

   设置key, value

8. ***get <key>***    

   获取key对应的值

9. ***mset <key1> <value1> <key2> <value2> <key3> <value3>***

   批量设置键值

10. ***mget <key1> <key2> <key3>***

    批量获取键的值

11. ***del <key>***   

    删除key

12. ***exists <key>***     

    判断是否存在key

13. ***expire <key> <seconds>***   

    设置key的过期时长(秒)

14. ***pexpire <key> <millionseconds>*** 

    重新设置key的过期时长(毫秒)

15. ***persist <key>***  

    若key设置了过期时长且未过期, 则将key改为永不过期

16. ***rename <key1> <key2>***

    key1重命名为key2

    

### String

1. ***set <key> <value>***

   设置key, value键值对

2. ***get <key>***

   获取key对应值

3. ***getrange <key> <start> <end>***        

   截取key的值的5~8位(包含8)

4. ***getset <key> <newValue>***       

   获取key的旧value, 并设置新值newValue,  该指令返回旧值

5. ***mset <key1> <value1> <key2> <value2>***           

   批量设置键值对

6. ***mget <key1> <key2>***           

   批量获取键值

7. ***setnx <key> <value>***           

   不存在就插入（not exists）

8. ***setex <key> <seconds> <value>***      

   设置键值对, 并指定key过期时间

9. ***setrange <key> <index> <value>***  

   从index处开始用value替换

10. ***incr <key>***        

    递增key的值

11. ***incrby <key> <n>***   

    对key的值从n开始递增

12. ***decr <key>***        

    递减key的值

13. ***decrby <key> <n>***   

    对key的值从n开始递减

14. ***incrbyfloat***     

    增减浮点数

15. ***append <key> value***          

    key对应值的末尾追加value, key不存在则创建<key>, <value>

16. ***strlen <key>***          

    返回对应键值的长度

17. ***getbit / setbit / bitcount / bitop***    

    位操作



### hash

Redis hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象。

1. ***HDEL key field1 [field2]***  

   删除一个或多个哈希表字段

2. ***HEXISTS key field*** 

   查看哈希表 key 中，指定的字段是否存在。

3. ***HGET key field***  

   获取存储在哈希表中指定字段的值。

4. ***HGETALL key***

   获取在哈希表中指定 key 的所有字段和值

5. ***HINCRBY key field increment***

   为哈希表 key 中的指定字段的整数值加上增量 increment 。

6. ***HINCRBYFLOAT key field increment***

   为哈希表 key 中的指定字段的浮点数值加上增量 increment 。

7. ***HKEYS key***

   获取所有哈希表中的字段

8. ***HLEN key***

   获取哈希表中字段的数量

9. ***HMGET key field1 [field2]***

   获取所有给定字段的值

10. ***HMSET key field1 value1 [field2 value2]***

    同时将多个 field-value (域-值)对设置到哈希表 key 中。

11. ***HSET key field value***

    将哈希表 key 中的字段 field 的值设为 value 。

12. ***HSETNX key field value***

    只有在字段 field 不存在时，设置哈希表字段的值。

13. ***HVALS key***

    获取哈希表中所有值。

14. ***HSCAN key cursor [MATCH pattern\] [COUNT count]***

    迭代哈希表中的键值对。



### List

Redis列表是简单的字符串列表，按照插入顺序排序。

1. ***BLPOP key1 [key2] timeout***  

   移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。

2. ***BRPOP key1 [key2] timeout***  

   移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。

3. ***BRPOPLPUSH source destination timeout*** 

   从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。

4. ***LINDEX key index*** 

   通过索引获取列表中的元素

5. ***LINSERT key BEFORE|AFTER pivot value***

   在列表的元素前或者后插入元素

6. ***LLEN key***  

   获取列表长度

7. ***LPOP key***  

   移出并获取列表的第一个元素

8. ***LPUSH key value1 [value2]***  

   将一个或多个值插入到列表头部  

9. ***LPUSHX key value***  

   将一个值插入到已存在的列表头部

10. ***LRANGE key start stop***  

    获取列表指定范围内的元素  

11. ***LREM key count value*** 

    移除列表元素  

12. ***LSET key index value*** 

    通过索引设置列表元素的值  

13. ***LTRIM key start stop***  

    对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。  

14. ***RPOP key***  

    移除列表的最后一个元素，返回值为移除的元素。  

15. ***RPOPLPUSH source destination*** 

    移除列表的最后一个元素，并将该元素添加到另一个列表并返回

16. ***RPUSH key value1 [value2]*** 

    在列表中添加一个或多个值    

17. ***RPUSHX key value***  

    为已存在的列表添加值

### Set

Redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。

1. ***SADD key member1 [member2]***

   向集合添加一个或多个成员

2. ***SCARD key*** 

   获取集合的成员数  

3. ***SDIFF key1 [key2]*** 

   返回给定所有集合的差集  

4. ***SDIFFSTORE destination key1 [key2]***  

   返回给定所有集合的差集并存储在 destination 中  

5. ***SINTER key1 [key2]***  

   返回给定所有集合的交集  

6. ***SINTERSTORE destination key1 [key2]***  

   返回给定所有集合的交集并存储在 destination 中  

7. ***SISMEMBER key member*** 

   判断 member 元素是否是集合 key 的成员  

8. ***SMEMBERS key***  

   返回集合中的所有成员  

9. ***SMOVE source destination member*** 

   将 member 元素从 source 集合移动到 destination 集合  

10. ***SPOP key*** 

    移除并返回集合中的一个随机元素  

11. ***SRANDMEMBER key [count]*** 

    返回集合中一个或多个随机数  

12. ***SREM key member1 [member2]***

    移除集合中一个或多个成员  

13. ***SUNION key1 [key2]***  

    返回所有给定集合的并集  

14. ***SUNIONSTORE destination key1 [key2]*** 

    所有给定集合的并集存储在 destination 集合中  

15. ***SSCAN key cursor [MATCH pattern\] [COUNT count]***

    迭代集合中的元素