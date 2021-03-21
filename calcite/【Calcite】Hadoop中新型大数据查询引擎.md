# 【Calcite】Hadoop中新型大数据查询引擎

[Apache Calcite](https://calcite.incubator.apache.org/) 是面向 Hadoop 新的查询引擎，它提供了标准的 SQL 语言、多种查询优化和连接各种数据源的能力，除此之外，Calcite 还提供了 OLAP 和流处理的查询引擎。正是有了这些诸多特性，Calcite 项目在 Hadoop 中越来越引入注目，并被众多项目集成。

Calcite 之前的名称叫做 [optiq](https://wiki.apache.org/incubator/optiqproposal)，optiq 起初在 Hive 项目中，为 Hive 提供基于成本模型的优化，即[CBO（Cost Based Optimizatio）](https://cwiki.apache.org/confluence/download/attachments/27362075/CBO-2.pdf)。2014 年 5 月 optiq 独立出来，成为 Apache 社区的孵化项目，2014 年 9 月正式更名为 Calcite。Calcite 项目的创建者是 [Julian Hyde](https://www.linkedin.com/in/julianhyde)，他在数据平台上有非常多的工作经历，曾经是 Oracle、 Broadbase 公司 SQL 引擎的主要开发者、SQLStream 公司的创始人和主架构师、Pentaho BI 套件中 OLAP 部分的架构师和主要开发者。现在他在 Hortonworks 公司负责 Calcite 项目，其工作经历对 Calcite 项目有很大的帮助。除了 Hortonworks，该项目的代码提交者还有 MapR 、Salesforce 等公司，并且还在不断壮大。

Calcite 的目标是“[one size fits all](http://www.slideshare.net/julianhyde/apache-calcite-one-planner-fits-all)（一种方案适应所有需求场景）”，希望能为不同计算平台和数据源提供统一的查询引擎，并以类似传统数据库的访问方式（SQL 和高级查询优化）来访问 Hadoop 上的数据。

Apache Calcite 具有以下几个[技术特性](http://www.slideshare.net/julianhyde/apache-calcite-overview) :

- 支持标准[SQL语言](http://calcite.incubator.apache.org/docs/reference.html)；
- 独立于编程语言和数据源，可以支持不同的前端和后端；
- 支持关系代数、可定制的逻辑规划规则和基于成本模型优化的查询引擎；
- 支持物化视图（ materialized view）的管理（创建、丢弃、持久化和自动识别）；
- 基于物化视图的 Lattice 和 Tile 机制，以应用于 OLAP 分析；
- 支持对流数据的查询。

下面对其中的一些特性更详细的介绍。

## 基于关系代数的查询引擎

我们知道，关系代数是关系型数据库操作的理论基础，关系代数支持并、差、笛卡尔积、投影和选择等基本运算。[关系代数](http://calcite.incubator.apache.org/docs/algebra.html#adding-a-filter-and-aggregate)是 Calcite 的核心，任何一个查询都可以表示成由关系运算符组成的树。 你可以将 SQL 转换成关系代数，或者通过 Calcite 提供的 API 直接创建它。比如下面这段 SQL 查询：

```sql
SELECT deptno, COUNT(*) AS c, SUM(sal) AS s
FROM emp
GROUP BY deptno
HAVING count(*) > 10
```

可以表达成如下的关系表达式语法树(AST)：

```SQL
LogicalFilter(condition=[>($1, 10)])
  LogicalAggregate(group=[{7}], C=[COUNT()], S=[SUM($5)])
    LogicalTableScan(table=[[scott, EMP]])
```

当上层编程语言，如 SQL 转换为关系表达式后，就会被送到 Calcite 的逻辑规划器进行规则匹配。在这个过程中，Calcite 查询引擎会循环使用规划规则对关系表达式语法树的节点和子图进行优化。这种优化过程会以一个成本模型作为参考，每次优化都在保证语义的情况下利用规则来降低成本，成本主要以查询时间最快、资源消耗最少这些维度去度量。

使用逻辑规划规则等同于数学恒等式变换，比如将一个过滤器推到内连接（inner join）输入的内部执行，当然使用这个规则的前提是过滤器不会引用内连接输入之外的数据列。图1 就是一个将 Filter 操作下推到 Join 下面的示例，这样做的好处是减少 Join 操作记录的数量。

![](https://static001.infoq.cn/resource/image/a0/b6/a0011186ec607810d3a87577acc79bb6.png)

>图1：一个逻辑规划的规则匹配（Filter 操作下沉)

非常好的一点是 Calcite 中的查询引擎是可以定制和扩展的，你可以自定义关系运算符、规划规则、成本模型和相关的统计，从而应用到不同需求的场景。

## 动态的数据管理系统

Calcite 的设计目标是成为[动态的数据管理系统](http://calcite.incubator.apache.org/docs/index.html)，所以在具有很多特性的同时，它也舍弃了一些功能，比如数据存储、处理数据的算法和元数据仓库。由于舍弃了这些功能，Calcite 可以在应用和数据存储、数据处理引擎之间很好地扮演中介的角色。用 Calcite 创建数据库非常灵活，你只需要动态地添加数据即可。

同时，前面提到过，Calcite 使用了基于关系代数的查询引擎，聚焦在关系代数的语法分析和查询逻辑的规划制定上。它不受上层编程语言的限制，前端可以使用SQL、Pig、Cascading 或者Scalding，只要通过 Calcite 提供的 API 将它们转化成关系代数的抽象语法树即可。

同时，Calcite 也不涉及物理规划层，它通过扩展适配器来连接多种后端的数据源和处理引擎，如 Spark、Splunk、HBase、Cassandra 或者 MangoDB。简单的说，这种架构就是“一种查询引擎，[连接多种前端和后端](http://calcite.incubator.apache.org/docs/adapter.html)”。

## 物化视图的应用

[Calcite 的物化视图](http://zh.hortonworks.com/blog/dmmq/)是从传统的关系型数据库系统（Oracle/DB2/Teradata/SQL server）借鉴而来，传统概念上，一个物化视图包含一个 SQL 查询和这个查询所生成的数据表。

下面是在 Hive 中创建物化视图的一个例子，它按部门、性别统计出相应的员工数量和工资总额:

```sql
CREATE MATERIALIZED VIEW emp_summary AS
SELECT deptno, gender, COUNT(*) AS c, SUM(salary) AS s
FROM emp
GROUP BY deptno, gender;
;
```

因为物化视图本质上也是一个数据表，所以你可以直接查询它，比如下面这个例子查询男员工人数大于 20 的部门：

```sql
SELECT deptno FROM emp_summary
WHERE gender = ‘M’ AND c > 20;
```

更重要的是，你还可以通过物化视图的查询取代对相关数据表的查询，可参见图2。由于物化视图一般存储在内存中，且其数据更接近于最终结果，所以查询速度会大大加快。

![img](https://static001.infoq.cn/resource/image/67/98/679d2cacf5b496aafc42447979731398.png)

> 图2：查询、物化视图和表的关系

比如下面这个对员工表（emp）的查询（女性的平均工资）：

```
SELECT deptno, AVG(salary) AS average_sal
FROM emp WHERE gender = 'F'
GROUP BY deptno;
```

可以被 Calcite 规划器改写成对物化视图（emp_summary）的查询：

```
SELECT deptno, s / c AS average_sal
FROM emp_summary WHERE gender = 'F'
GROUP BY deptno;
```

我们可以看到，多数值的平均运算，即先累加再除法转化成了单个除法。

为了让物化视图可以被所有编程语言访问，需要将其转化为与语言无关的关系代数并将其元数据保存在 Hive 的 HCatalog 中。HCatalog 可以独立于 Hive，被其它查询引擎使用，它负责 Hadoop 元数据和表的管理。

物化视图可以进一步扩展为[DIMMQ（Discardable, In-Memory, Materialized Query](http://zh.hortonworks.com/blog/dmmq/)。简单地说，DIMMQ 就是内存中可丢弃的物化视图，它是高级别的缓存。相对原始数据，它离查询结果更近，所占空间更小，并可以被多个应用共享，并且应用不必感知物化视图存在，查询引擎会自动匹配它。物化视图可以和异构存储结合起来，即它可以存储在 Disk、SSD 或者内存中，并根据数据的热度进行动态调整。

除了上面例子中的归纳表（员工工资、员工数量），物化视图还可以应用在其它地方，比如 b-tree 索引 (使用基础的排序投影运算)、分区表和远端快照。总之，通过使用物化视图，应用程序可以设计自己的派生数据结构，并使其被系统自动识别和使用。

## 在线分析处理(OLAP)

为了加速在线分析处理，除了物化视图，Calcite 还引入[Lattice（格子）和 Tile（瓷片](http://www.slideshare.net/julianhyde/calcite-stratany2014)的概念。Lattice 可以看做是在 [星模式（star schema](https://en.wikipedia.org/wiki/Star_schema) 数据模型下对物化视图的推荐、创建和识别的机制。这种推荐可以根据查询的频次统计，也可以基于某些分析维度的重要等级。Tile 则是Lattice 中的一个逻辑的物化视图，它可以通过三种方法来实体化：

1. 在lattice 中声明
2. 通过推荐算法实现
3. 在响应查询时创建

下图是 Lattice 和 Tile 的一个图例，这个 OLAP 分析涉及五个维度的数据：邮政编码、州、性别、年和月。每个椭圆代表一个 Tile，黑色椭圆是实体化后物化视图，椭圆中的数字代表该物化视图对应的记录数。

![img](https://static001.infoq.cn/resource/image/74/a8/74f4c9ebfe63386a0afd51f97e0ea9a8.png)

> 图3：Lattice 和 Tile 的示例图

由于 Calcite 可以很好地支持物化视图和星模式这些 OLAP 分析的关键特性，所以 Apache 基金会的 [Kylin 项目](http://kylin.incubator.apache.org/)（Hadoop 上 OLAP 系统）在选用查询引擎时就直接集成了 Calcite。

## 支持流查询

Calcite 对其 SQL 和关系代数进行了扩展以支持[流查询](https://calcite.incubator.apache.org/docs/stream.html)。Calcite 的 SQL 语言是标准 SQL 的扩展，而不是类 SQL（SQL-like），这个差别非常重要，因为：

- 如果你懂标准 SQL，那么流的 SQL 也会非常容易学；
- 因为在流和表上使用相同的机制，语义会很清楚；
- 你可以写同时对流和表结合的查询语句；
- 很多工具可以直接生成标准的 SQL。

Calcite 的流查询除了支持排序、聚合、过滤等常用操作和子查询外，也支持各种窗口操作，比如翻滚窗口 (Tumbling window)、[跳跃窗口 (Hopping window)](https://calcite.incubator.apache.org/docs/stream.html#hopping-windows)、滑动窗口 (Sliding windows)、级联窗口 (Cascading window)。其中级联窗口可以看作是滑动窗口和翻滚窗口的结合。

## 总结

Calcite 是一种动态数据管理系统，它具有标准 SQL、连接不同前端和后端、可定制的逻辑规划器、物化视图、多维数据分析和流查询等诸多能力，使其成为大数据领域中非常有吸引力的查询引擎。

目前它已经或被规划集成到 Hadoop 的诸多项目中，比如 Lingual (Cascading 项目的 SQL 接口)、Apache Drill、Apache Hive、Apache Kylin、Apache Phoenix、Apache Samza 和 Apache Flink。

