# 【Kafka】数据的清理 LogDeletion

Kafka 将消息存储在磁盘中，为了控制磁盘占用空间的不断增加就需要对消息做一定的清理操作。

Kafka 中每一个分区 partition 都对应一个日志文件，而日志文件又可以分为多个日志分段文件，这样也便于日志的清理操作。Kafka 提供了两种日志清理策略：

1. 日志删除（Log Deletion）

   按照一定的保留策略来直接删除不符合条件的日志分段。

2. 日志压缩（Log Compaction）

   针对每个消息的 key 进行整合，对于有相同 key 的的不同 value 值，只保留最后一个版本。



我们可以通过 broker 端参数 log.cleanup.policy 来设置日志清理策略，此参数默认值为“delete”，即采用日志删除的清理策略。如果要采用日志压缩的清理策略的话，就需要将log.cleanup.policy 设置为 “compact”，并且还需要将 log.cleaner.enable（默认值为true）设定为true。通过将 log.cleanup.policy 参数设置为“delete,compact”还可以同时支持日志删除和日志压缩两种策略。日志清理的粒度可以控制到 topic 级别，比如与log.cleanup.policy 对应的主题级别的参数为 cleanup.policy，为了简化说明，本文只采用broker端参数做陈述，如若需要 topic 级别的参数可以查看[官方文档](http://kafka.apache.org/documentation/#topicconfigs)。

## 日志删除

Kafka 日志管理器中会有一个专门的日志删除任务来周期性检测和删除不符合保留条件的日志分段文件，这个周期可以通过 broker 端参数 log.retention.check.interval.ms 来配置，默认值为300,000，即5分钟。当前日志分段的保留策略有3种：基于时间的保留策略、基于日志大小的保留策略以及基于日志起始偏移量的保留策略。

### 1. 基于时间

日志删除任务会检查当前日志文件中是否有保留时间超过设定的阈值retentionMs来寻找可删除的的日志分段文件集合deletableSegments，参考下图所示。retentionMs可以通过broker端参数log.retention.hours、log.retention.minutes 以及 log.retention.ms 来配置，其中log.retention.ms 的优先级最高，log.retention.minutes 次之，log.retention.hours最低。默认情况下只配置了 log.retention.hours 参数，其值为168，故默认情况下日志分段文件的保留时间为7天。
[![img](http://image.honeypps.com/images/papers/2018/123.png)](http://image.honeypps.com/images/papers/2018/123.png)

查找过期的日志分段文件，并不是简单地根据日志分段的最近修改时间 lastModifiedTime 来计算，而是根据日志分段中最大的时间戳 largestTimeStamp 来计算。因为日志分段的lastModifiedTime 可以被有意或者无意的修改，比如执行了 touch 操作，或者分区副本进行了重新分配，lastModifiedTime 并不能真实地反映出日志分段在磁盘的保留时间。要获取日志分段中的最大时间戳 largestTimeStamp 的值，首先要查询该日志分段所对应的时间戳索引文件，查找时间戳索引文件中最后一条索引项，若最后一条索引项的时间戳字段值大于0，则取其值，否则才设置为最近修改时间 lastModifiedTime。

若待删除的日志分段的总数等于该日志文件中所有的日志分段的数量，那么说明所有的日志分段都已过期，但是该日志文件中还要有一个日志分段来用于接收消息的写入，即必须要保证有一个活跃的日志分段activeSegment，在此种情况下，会先切分出一个新的日志分段作为activeSegment，然后再执行删除操作。

删除日志分段时，首先会从日志文件对象中所维护日志分段的跳跃表中移除待删除的日志分段，以保证没有线程对这些日志分段进行读取操作。然后将日志分段文件添加上“.deleted”的后缀，当然也包括日志分段对应的索引文件。最后交由一个以“delete-file”命名的延迟任务来删除这些“.deleted”为后缀的文件，这个任务的延迟执行时间可以通过file.delete.delay.ms参数来设置，默认值为60000，即1分钟。

### 2. 基于日志大小

日志删除任务会检查当前日志的大小是否超过设定的阈值retentionSize来寻找可删除的日志分段的文件集合deletableSegments，参考下图所示。retentionSize可以通过broker端参数log.retention.bytes来配置，默认值为-1，表示无穷大。注意log.retention.bytes配置的是日志文件的总大小，而不是单个的日志分段的大小，一个日志文件包含多个日志分段。
[![img](http://image.honeypps.com/images/papers/2018/124.png)](http://image.honeypps.com/images/papers/2018/124.png)

基于日志大小的保留策略与基于时间的保留策略类似，其首先计算日志文件的总大小 size 和retentionSize 的差值 diff，即计算需要删除的日志总大小，然后从日志文件中的第一个日志分段开始进行查找可删除的日志分段的文件集合deletableSegments。查找出 deletableSegments之后就执行删除操作，这个删除操作和基于时间的保留策略的删除操作相同，这里不再赘述。

### 3. 基于日志起始偏移量

一般情况下日志文件的起始偏移量logStartOffset等于第一个日志分段的baseOffset，但是这并不是绝对的，logStartOffset的值可以通过DeleteRecordsRequest请求、日志的清理和截断等操作修改。
[![img](http://image.honeypps.com/images/papers/2018/125.png)](http://image.honeypps.com/images/papers/2018/125.png)

基于日志起始偏移量的删除策略的判断依据是某日志分段的下一个日志分段的起始偏移量baseOffset是否小于等于logStartOffset，若是则可以删除此日志分段。参考上图，假设logStartOffset等于25，日志分段1的起始偏移量为0，日志分段2的起始偏移量为11，日志分段3的起始偏移为23，那么我们通过如下动作收集可删除的日志分段的文件集合deletableSegments：

1. 从头开始遍历每个日志分段，日志分段1的下一个日志分段的起始偏移量为11，小于logStartOffset的大小，将日志分段1加入到deletableSegments中；
2. 日志分段2的下一个日志偏移量的起始偏移量为23，也小于logStartOffset的大小，将日志分段2页加入到deletableSegments中；
3. 日志分段3的下一个日志偏移量在logStartOffset的右侧，故从日志分段3开始的所有日志分段都不会被加入到deletableSegments中。

收集完可删除的日志分段的文件集合之后的删除操作同基于日志大小的保留策略和基于时间的保留策略相同，这里不再赘述。