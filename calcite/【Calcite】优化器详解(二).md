# 【Calcite】优化器详解(二)

这里是 Calcite 系列文章的第二篇，后面还会有文章讲述 Calcite 的实践（包括：如何开发用于 SQL 优化的 Rule）。本篇文章主要介绍 Apache Calcite 优化器部分的内容，会先简单介绍一下 RBO 和 CBO 模型，之后详细讲述 Calcite 关于这两个优化器的实现 —— HepPlanner 和 VolcanoPlanner，文章内容都是个人的一些理解，由于也是刚接触这块，理解有偏差的地方，欢迎指正。

## 一、查询优化器

查询优化器是传统数据库的核心模块，也是大数据计算引擎的核心模块，开源大数据引擎如 Impala、Presto、Drill、HAWQ、 Spark、Hive 等都有自己的查询优化器。Calcite 就是从 Hive 的优化器演化而来的。

优化器的作用：将解析器生成的关系代数表达式转换成执行计划，供执行引擎执行，在这个过程中，会应用一些规则优化，以帮助生成更高效的执行计划。

> 关于 Volcano 模型和 Cascades 模型的内容，建议看下相关的论文，这个是 Calcite 优化器的理论基础，代码只是把这个模型落地实现而已。 

### 1.1 基于规则优化（RBO）

基于规则的优化器（Rule-Based Optimizer，RBO）：根据优化规则对关系表达式进行转换，这里的转换是说一个关系表达式经过优化规则后会变成另外一个关系表达式，同时原有表达式会被裁剪掉，经过一系列转换后生成最终的执行计划。

RBO 中包含了一套有着严格顺序的优化规则，同样一条 SQL，无论读取的表中数据是怎么样的，最后生成的执行计划都是一样的。同时，在 RBO 中 SQL 写法的不同很有可能影响最终的执行计划，从而影响执行计划的性能。

### 1.2 基于成本优化（CBO）

基于代价的优化器(Cost-Based Optimizer，CBO)：根据优化规则对关系表达式进行转换，这里的转换是说一个关系表达式经过优化规则后会生成另外一个关系表达式，同时原有表达式也会保留，经过一系列转换后会生成多个执行计划，然后 CBO 会根据统计信息和代价模型 (Cost Model) 计算每个执行计划的 Cost，从中挑选 Cost 最小的执行计划。

由上可知，CBO 中有两个依赖：统计信息和代价模型。统计信息的准确与否、代价模型的合理与否都会影响 CBO 选择最优计划。 从上述描述可知，CBO 是优于 RBO 的，原因是 RBO 是一种只认规则，对数据不敏感的呆板的优化器，而在实际过程中，数据往往是有变化的，通过 RBO 生成的执行计划很有可能不是最优的。事实上目前各大数据库和大数据计算引擎都倾向于使用 CBO，但是对于流式计算引擎来说，使用 CBO 还是有很大难度的，因为并不能提前预知数据量等信息，这会极大地影响优化效果，CBO 主要还是应用在离线的场景。

## 优化规则

无论是 RBO，还是 CBO 都包含了一系列优化规则，这些优化规则可以对关系表达式进行等价转换，常见的优化规则包含：

1. 谓词下推 Predicate Pushdown
2. 常量折叠 Constant Folding
3. 列裁剪 Column Pruning
4. 其他

在 Calcite 的代码里，有一个测试类（`org.apache.calcite.test.RelOptRulesTest`）汇集了对目前内置所有 Rules 的测试 case，这个测试类可以方便我们了解各个 Rule 的作用。在这里有下面一条 SQL，通过这条语句来说明一下上面介绍的这三种规则。

```
select 10 + 30, users.name, users.age
from users join jobs on users.id= user.id
where users.age > 30 and jobs.id>10
```

### 谓词下推(Predicate Pushdown)

关于谓词下推，它主要还是从关系型数据库借鉴而来，关系型数据中将谓词下推到外部数据库用以减少数据传输；属于逻辑优化，优化器将谓词过滤下推到数据源，使物理执行跳过无关数据。

最常见的例子就是 join 与 filter 操作一起出现时，提前执行 filter 操作以减少处理的数据量，将 filter 操作下推，以上面例子为例，示意图如下（对应 Calcite 中的 `FilterJoinRule.FilterIntoJoinRule.FILTER_ON_JOIN` Rule）：

[![Filter操作下推前后的对比](http://matt33.com/images/calcite/6-filter-pushdown.png)](http://matt33.com/images/calcite/6-filter-pushdown.png)

在进行 join 前进行相应的过滤操作，可以极大地减少参加 join 的数据量。

### 常量折叠(Constant Folding)

常量折叠也是常见的优化策略，这个比较简单、也很好理解，可以看下 [编译器优化 – 常量折叠](http://blog.caoxudong.info/blog/2013/10/23/compiler_optimizations_constant_folding) 这篇文章，基本不用动脑筋就能理解，对于我们这里的示例，有一个常量表达式 `10 + 30`，如果不进行常量折叠，那么每行数据都需要进行计算，进行常量折叠后的结果如下图所示（对应 Calcite 中的 `ReduceExpressionsRule.PROJECT_INSTANCE` Rule）：

[![常量折叠前后的对比](http://matt33.com/images/calcite/7-constant.png)](http://matt33.com/images/calcite/7-constant.png)常量折叠前后的对比

### 列裁剪(Column Pruning)

列裁剪也是一个经典的优化规则，在本示例中对于jobs 表来说，并不需要扫描它的所有列值，而只需要列值 id，所以在扫描 jobs 之后需要将其他列进行裁剪，只留下列 id。这个优化带来的好处很明显，大幅度减少了网络 IO、内存数据量的消耗。裁剪前后的示意图如下（不过并没有找到 Calcite 对应的 Rule）：

[![列裁剪前后的对比](http://matt33.com/images/calcite/8-pruning.png)](http://matt33.com/images/calcite/8-pruning.png)

## 二、Calcite中的优化器实现

有了前面的基础后，这里来看下 Calcite 中优化器的实现，RelOptPlanner 是 Calcite 中优化器的基类，其子类实现如下图所示：

![](http://matt33.com/images/calcite/9-RelOptPlanner.png)

Calcite 中关于优化器提供了两种实现：

1. HepPlanner：就是前面 RBO 的实现，它是一个启发式的优化器，按照规则进行匹配，直到达到次数限制（match 次数限制）或者遍历一遍后不再出现 rule match 的情况才算完成；
2. VolcanoPlanner：就是前面 CBO 的实现，它会一直迭代 rules，直到找到 cost 最小的 paln。

>前面提到过像 calcite 这类查询优化器最核心的两个问题之一是: 怎么把优化规则应用到关系代数相关的 RelNode Tree 上。所以在阅读 calicite 的代码时就得带着这个问题去看看它的实现过程，然后才能判断它的代码实现得是否优雅。
>
>calcite 的每种规则实现类 (RelOptRule 的子类) 都会声明自己应用在哪种 RelNode 子类上，每个 RelNode 子类其实都可以看成是一种 operator(中文常翻译成算子)。
>
>VolcanoPlanner 就是优化器，用的是动态规划算法，在创建 VolcanoPlanner 的实例后，通过 calcite 的标准 jdbc 接口执行 sql 时，默认会给这个 VolcanoPlanner 的实例注册将近90条优化规则(还不算常量折叠这种最常见的优化)，所以看代码时，知道什么时候注册可用的优化规则是第一步(调用 VolcanoPlanner.addRule 实现)，这一步比较简单。
>
>接下来就是如何筛选规则了，当把语法树转成 RelNode Tree 后是没有必要把前面注册的90条优化规则都用上的，所以需要有个筛选的过程，因为每种规则是有应用范围的，按 RelNode Tree 的不同节点类型就可以筛选出实际需要用到的优化规则了。这一步说起来很简单，但在 calcite 的代码实现里是相当复杂的，也是非常关键的一步，是从调用 VolcanoPlanner.setRoot 方法开始间接触发的，如果只是静态的看代码不跑起来跟踪调试多半摸不清它的核心流程的。筛选出来的优化规则会封装成 VolcanoRuleMatch，然后扔到 RuleQueue 里，而这个 RuleQueue 正是接下来执行动态规划算法要用到的核心类。筛选规则这一步的代码实现很晦涩。
>
>第三步才到 VolcanoPlanner.findBestExp，本质上就是一个动态规划算法的实现，但是最值得关注的还是怎么用第二步筛选出来的规则对 RelNode Tree 进行变换，变换后的形式还是一棵RelNode Tree，最常见的是把 LogicalXXX 开头的 RelNode 子类换成了 EnumerableXXX或 BindableXXX。
>
>总而言之，看看具体优化规则的实现就对了，都是繁琐的体力活。一个优化器，理解了上面所说的三步基本上就抓住重点了。



下面详细讲述一下这两种 planner 在 Calcite 内部的具体实现。

### 2.1 HepPlanner

使用 HepPlanner 实现的完整代码见 [SqlHepTest](https://github.com/wangzzu/program-example/blob/master/calcite-example/src/main/java/com/matt/test/calcite/sql/SqlHepTest.java)。

#### 2.1.1 HepPlanner 中的基本概念

这里先看下 HepPlanner 的一些基本概念，对于后面的理解很有帮助。

##### HepRelVertex

HepRelVertex 是对 RelNode 进行了简单封装。HepPlanner 中的所有节点都是 HepRelVertex，每个 HepRelVertex 都指向了一个真正的 RelNode 节点。

```java
// org.apache.calcite.plan.hep.HepRelVertex
/**
 * HepRelVertex wraps a real {@link RelNode} as a vertex in a DAG representing
 * the entire query expression.
 * note：HepRelVertex 将一个 RelNode 封装为一个 DAG 中的 vertex（DAG 代表整个 query expression）
 */
public class HepRelVertex extends AbstractRelNode {
  //~ Instance fields ------------------------

  /**
   * Wrapped rel currently chosen for implementation of expression.
   */
  private RelNode currentRel;
}
```

##### HepInstruction

HepInstruction 是 HepPlanner 对一些内容的封装，具体的子类实现比较多，其中 RuleInstance 是 HepPlanner 中对 Rule 的一个封装，注册的 Rule 最后都会转换为这种形式。

> HepInstruction represents one instruction in a HepProgram. 

```java
//org.apache.calcite.plan.hep.HepInstruction
/** Instruction that executes a given rule. */
//note: 执行指定 rule 的 Instruction
static class RuleInstance extends HepInstruction {
  /**
   * Description to look for, or null if rule specified explicitly.
   */
  String ruleDescription;

  /**
   * Explicitly specified rule, or rule looked up by planner from
   * description.
   * note：设置其 Rule
   */
  RelOptRule rule;

  void initialize(boolean clearCache) {
    if (!clearCache) {
      return;
    }

    if (ruleDescription != null) {
      // Look up anew each run.
      rule = null;
    }
  }

  void execute(HepPlanner planner) {
    planner.executeInstruction(this);
  }
}
```

#### 2.1.2 HepPlanner 处理流程

下面这个示例是上篇文章[【Calcite】处理流程详解(一)](/Users/sherlock/Desktop/notes/calcite/[Calcite]处理流程详解(一).md) 的示例，通过这段代码来看下 HepPlanner 的内部实现机制。

```java
HepProgramBuilder builder = new HepProgramBuilder();
//note: 添加 rule
builder.addRuleInstance(
  	FilterJoinRule.FilterIntoJoinRule.FILTER_ON_JOIN); 
HepPlanner hepPlanner = new HepPlanner(builder.build());
hepPlanner.setRoot(relNode);
relNode = hepPlanner.findBestExp();
```

上面的代码总共分为三步：

1. 初始化 HepProgram 对象；
2. 初始化 HepPlanner 对象，并通过 `setRoot()` 方法将 RelNode 树转换成 HepPlanner 内部使用的 Graph；
3. 通过 `findBestExp()` 找到最优的 plan，规则的匹配都是在这里进行。

##### 1.初始化 HepProgram

这几步代码实现没有太多需要介绍的地方，先初始化 HepProgramBuilder 也是为了后面初始化 HepProgram 做准备，HepProgramBuilder 主要也就是提供了一些配置设置和添加规则的方法等，常用的方法如下：

1. `addRuleInstance()`

   注册相应的规则；

2. `addRuleCollection()`

   这里是注册一个规则集合，先把规则放在一个集合里，再注册整个集合，如果规则多的话，一般是这种方式；

3. `addMatchLimit()`

   设置 MatchLimit，这个 rule match 次数的最大限制；

HepProgram 这个类对于后面 HepPlanner 的优化很重要，它定义 Rule 匹配的顺序，默认按【深度优先】顺序，它可以提供以下几种（见 HepMatchOrder 类）：

1. **ARBITRARY**：按任意顺序匹配（因为它是有效的，而且大部分的 Rule 并不关心匹配顺序）；
2. **BOTTOM_UP**：自下而上，先从子节点开始匹配；
3. **TOP_DOWN**：自上而下，先从父节点开始匹配；
4. **DEPTH_FIRST**：深度优先匹配，某些情况下比 ARBITRARY 高效（为了避免新的 vertex 产生后又从 root 节点开始匹配）。

这个匹配顺序到底是什么呢？对于规则集合 rules，HepPlanner 的算法是：从一个节点开始，跟 rules 的所有 Rule 进行匹配，匹配上就进行转换操作，这个节点操作完，再进行下一个节点，这里的匹配顺序就是指的**节点遍历顺序**（这种方式的优劣，我们下面再说）。

##### 2.HepPlanner.setRoot（RelNode –> Graph）

先看下 `setRoot()` 方法的实现：

```java
// org.apache.calcite.plan.hep.HepPlanner
public void setRoot(RelNode rel) {
  //note: 将 RelNode 转换为 DAG 表示
  root = addRelToGraph(rel);
  //note: 仅仅是在 trace 日志中输出 Graph 信息
  dumpGraph();
}
```

HepPlanner 会先将所有 relNode tree 转化为 HepRelVertex，这时就构建了一个 Graph：将所有的 elNode 节点使用 Vertex 表示，Gragh 会记录每个 HepRelVertex 的 input 信息，这样就是构成了一张 graph。

在真正的实现时，递归逐渐将每个 relNode 转换为 HepRelVertex，并在 `graph` 中记录相关的信息，实现如下：

```java
//org.apache.calcite.plan.hep.HepPlanner
//note: 根据 RelNode 构建一个 Graph
private HepRelVertex addRelToGraph(
  RelNode rel) {
  // Check if a transformation already produced a reference
  // to an existing vertex.
  //note: 检查这个 rel 是否在 graph 中转换了
  if (graph.vertexSet().contains(rel)) {
    return (HepRelVertex) rel;
  }

  // Recursively add children, replacing this rel's inputs
  // with corresponding child vertices.
  //note: 递归地增加子节点，使用子节点相关的 vertices 代替 rel 的 input
  final List<RelNode> inputs = rel.getInputs();
  final List<RelNode> newInputs = new ArrayList<>();
  for (RelNode input1 : inputs) {
    HepRelVertex childVertex = addRelToGraph(input1); //note: 递归进行转换
    newInputs.add(childVertex); //note: 每个 HepRelVertex 只记录其 Input
  }

  if (!Util.equalShallow(inputs, newInputs)) { //note: 不相等的情况下
    RelNode oldRel = rel;
    rel = rel.copy(rel.getTraitSet(), newInputs);
    onCopy(oldRel, rel);
  }
  // Compute digest first time we add to DAG,
  // otherwise can't get equivVertex for common sub-expression
  //note: 计算 relNode 的 digest
  //note: Digest 的意思是：
  //note: A short description of this relational expression's type, inputs, and
  //note: other properties. The string uniquely identifies the node; another node
  //note: is equivalent if and only if it has the same value.
  rel.recomputeDigest();

  // try to find equivalent rel only if DAG is allowed
  //note: 如果允许 DAG 的话，检查是否有一个等价的 HepRelVertex，有的话直接返回
  if (!noDag) {
    // Now, check if an equivalent vertex already exists in graph.
    String digest = rel.getDigest();
    HepRelVertex equivVertex = mapDigestToVertex.get(digest);
    if (equivVertex != null) { //note: 已经存在
      // Use existing vertex.
      return equivVertex;
    }
  }

  // No equivalence:  create a new vertex to represent this rel.
  //note: 创建一个 vertex 代替 rel
  HepRelVertex newVertex = new HepRelVertex(rel);
  graph.addVertex(newVertex); //note: 记录 Vertex
  updateVertex(newVertex, rel);//note: 更新相关的缓存，比如 mapDigestToVertex map

  for (RelNode input : rel.getInputs()) { //note: 设置 Edge
    graph.addEdge(newVertex, (HepRelVertex) input);//note: 记录与整个 Vertex 先关的 input
  }

  nTransformations++;
  return newVertex;
}
```

到这里 HepPlanner 需要的 gragh 已经构建完成，通过 DEBUG 方式也能看到此时 HepPlanner root 变量的内容：

[![Root 转换之后的内容](http://matt33.com/images/calcite/10-calcite.png)](http://matt33.com/images/calcite/10-calcite.png)Root 转换之后的内容

##### 3.HepPlanner findBestExp 规则优化

```
//org.apache.calcite.plan.hep.HepPlanner
// implement RelOptPlanner
//note: 优化器的核心，匹配规则进行优化
public RelNode findBestExp() {
  assert root != null;

  //note: 运行 HepProgram 算法(按 HepProgram 中的 instructions 进行相应的优化)
  executeProgram(mainProgram);

  // Get rid of everything except what's in the final plan.
  //note: 垃圾收集
  collectGarbage();

  return buildFinalPlan(root); //note: 返回最后的结果，还是以 RelNode 表示
}
```

主要的实现是在 `executeProgram()` 方法中，如下：

```java
//org.apache.calcite.plan.hep.HepPlanner
private void executeProgram(HepProgram program) {
  HepProgram savedProgram = currentProgram; //note: 保留当前的 Program
  currentProgram = program;
  currentProgram.initialize(program == mainProgram);//note: 如果是在同一个 Program 的话，保留上次 cache
  for (HepInstruction instruction : currentProgram.instructions) {
    instruction.execute(this); //note: 按 Rule 进行优化(会调用 executeInstruction 方法)
    int delta = nTransformations - nTransformationsLastGC;
    if (delta > graphSizeLastGC) {
      // The number of transformations performed since the last
      // garbage collection is greater than the number of vertices in
      // the graph at that time.  That means there should be a
      // reasonable amount of garbage to collect now.  We do it this
      // way to amortize garbage collection cost over multiple
      // instructions, while keeping the highwater memory usage
      // proportional to the graph size.
      //note: 进行转换的次数已经大于 DAG Graph 中的顶点数，这就意味着已经产生大量垃圾需要进行清理
      collectGarbage();
    }
  }
  currentProgram = savedProgram;
}
```

这里会遍历 HepProgram 中 instructions（记录注册的所有 HepInstruction），然后根据 instruction 的类型执行相应的 `executeInstruction()` 方法，如果instruction 是 `HepInstruction.MatchLimit` 类型，会执行 `executeInstruction(HepInstruction.MatchLimit instruction)` 方法，这个方法就是初始化 matchLimit 变量。

对于 `HepInstruction.RuleInstance` 类型的 instruction 会执行下面的方法（前面的示例注册规则使用的是 `addRuleInstance()` 方法，所以返回的 rules 只有一个规则，如果注册规则的时候使用的是 `addRuleCollection()` 方法注册一个规则集合的话，这里会返回的 rules 就是那个规则集合）：

```java
//org.apache.calcite.plan.hep.HepPlanner
//note: 执行相应的 RuleInstance
void executeInstruction(
    HepInstruction.RuleInstance instruction) {
  if (skippingGroup()) {
    return;
  }
  if (instruction.rule == null) {//note: 如果 rule 为 null，那么就按照 description 查找具体的 rule
    assert instruction.ruleDescription != null;
    instruction.rule =
        getRuleByDescription(instruction.ruleDescription);
    LOGGER.trace("Looking up rule with description {}, found {}",
        instruction.ruleDescription, instruction.rule);
  }
  //note: 执行相应的 rule
  if (instruction.rule != null) {
    applyRules(
        Collections.singleton(instruction.rule),
        true);
  }
}
```

接下来看 `applyRules()` 的实现：

```java
//org.apache.calcite.plan.hep.HepPlanner
//note: 执行 rule（forceConversions 默认 true）
private void applyRules(
    Collection<RelOptRule> rules,
    boolean forceConversions) {
  if (currentProgram.group != null) {
    assert currentProgram.group.collecting;
    currentProgram.group.ruleSet.addAll(rules);
    return;
  }

  LOGGER.trace("Applying rule set {}", rules);

  //note: 当遍历规则是 ARBITRARY 或 DEPTH_FIRST 时，设置为 false，此时不会从 root 节点开始，否则每次 restart 都从 root 节点开始
  boolean fullRestartAfterTransformation =
      currentProgram.matchOrder != HepMatchOrder.ARBITRARY
      && currentProgram.matchOrder != HepMatchOrder.DEPTH_FIRST;

  int nMatches = 0;

  boolean fixedPoint;
  //note: 两种情况会跳出循环，一种是达到 matchLimit 限制，一种是遍历一遍不会再有新的 transform 产生
  do {
    //note: 按照遍历规则获取迭代器
    Iterator<HepRelVertex> iter = getGraphIterator(root);
    fixedPoint = true;
    while (iter.hasNext()) {
      HepRelVertex vertex = iter.next();//note: 遍历每个 HepRelVertex
      for (RelOptRule rule : rules) {//note: 遍历每个 rules
        //note: 进行规制匹配，也是真正进行相关操作的地方
        HepRelVertex newVertex =
            applyRule(rule, vertex, forceConversions);
        if (newVertex == null || newVertex == vertex) {
          continue;
        }
        ++nMatches;
        //note: 超过 MatchLimit 的限制
        if (nMatches >= currentProgram.matchLimit) {
          return;
        }
        if (fullRestartAfterTransformation) {
          //note: 发生 transformation 后，从 root 节点再次开始
          iter = getGraphIterator(root);
        } else {
          // To the extent possible, pick up where we left
          // off; have to create a new iterator because old
          // one was invalidated by transformation.
          //note: 尽可能从上次进行后的节点开始
          iter = getGraphIterator(newVertex);
          if (currentProgram.matchOrder == HepMatchOrder.DEPTH_FIRST) {
            //note: 这样做的原因就是为了防止有些 HepRelVertex 遗漏了 rule 的匹配（每次从 root 开始是最简单的算法），因为可能出现下推
            nMatches =
                depthFirstApply(iter, rules, forceConversions, nMatches);
            if (nMatches >= currentProgram.matchLimit) {
              return;
            }
          }
          // Remember to go around again since we're
          // skipping some stuff.
          //note: 再来一遍，因为前面有跳过一些节点
          fixedPoint = false;
        }
        break;
      }
    }
  } while (!fixedPoint);
}
```

在这里会调用 `getGraphIterator()` 方法获取 HepRelVertex 的迭代器，迭代的策略（遍历的策略）跟前面说的顺序有关，默认使用的是【深度优先】，这段代码比较简单，就是遍历规则+遍历节点进行匹配转换，直到满足条件再退出，从这里也能看到 HepPlanner 的实现效率不是很高，它也无法保证能找出最优的结果。

总结一下，HepPlanner 在优化过程中，是先遍历规则，然后再对每个节点进行匹配转换，直到满足条件（超过限制次数或者规则遍历完一遍不会再有新的变化），其方法调用流程如下：

[![HepPlanner 处理流程](http://matt33.com/images/calcite/11-hep.png)](http://matt33.com/images/calcite/11-hep.png)

#### 2.1.3 思考

1. 为什么要把 RelNode 转换 HepRelVertex 进行优化？带来的收益在哪里？

   关于这个，能想到的就是：RelNode 是底层提供的抽象、偏底层一些，在优化器这一层，需要记录更多的信息，所以又做了一层封装。

### 2.2 VolcanoPlanner

介绍完 HepPlanner 之后，接下来再来看下基于成本优化（CBO）模型在 Calcite 中是如何实现、如何落地的，关于 Volcano 理论内容建议先看下相关理论知识，否则直接看实现的话可能会有一些头大。

从 Volcano 模型的理论落地到实践是有很大区别的，这里先看一张 VolcanoPlanner 整体实现图，如下所示（图片来自 [Cost-based Query Optimization in Apache Phoenix using Apache Calcite](https://www.slideshare.net/julianhyde/costbased-query-optimization-in-apache-phoenix-using-apache-calcite?qid=b7a1ca0f-e7bf-49ad-bc51-0615ec8a4971&v=&b=&from_search=4)）：

[![Calcite VolcanoPlanner Process](http://matt33.com/images/calcite/12-VolcanoPlanner.png)](http://matt33.com/images/calcite/12-VolcanoPlanner.png)

上面基本展现了 VolcanoPlanner 内部实现的流程，也简单介绍了 VolcanoPlanner 在实现中的一些关键点（有些概念暂时不了解也不要紧，后面会介绍）：

1. Add Rule matches to Queue：向 Rule Match Queue 中添加相应的 Rule Match；
2. Apply Rule match transformations to plan gragh：应用 Rule Match 对 plan graph 做 transformation 优化（Rule specifies an Operator sub-graph to match and logic to generate equivalent better sub-graph）；
3. Iterate for fixed iterations or until cost doesn’t change：进行相应的迭代，直到 cost 不再变化或者 Rule Match Queue 中 rule match 已经全部应用完成；
4. Match importance based on cost of RelNode and height：Rule Match 的 importance 依赖于 RelNode 的 cost 和深度。

使用 VolcanoPlanner 实现的完整代码见 [SqlVolcanoTest](https://github.com/wangzzu/program-example/blob/master/calcite-example/src/main/java/com/matt/test/calcite/sql/SqlVolcanoTest.java)。

下面来看下 VolcanoPlanner 实现具体的细节。

#### 2.2.1 VolcanoPlanner 中的基本概念

VolcanoPlanner 在实现中引入了一些基本概念，先明白这些概念对于理解 VolcanoPlanner 的实现非常有帮助。

##### RelSet

关于 RelSet，源码中介绍如下：

> RelSet is an equivalence-set of expressions that is, a set of expressions which have **identical semantics**.
> We are generally interested in using the expression which has **the lowest cost**.
> All of the expressions in an RelSet have the **same calling convention**.

它有以下特点：

1. 描述一组等价 Relation Expression，所有的 RelNode 会记录在 `rels` 中；
2. have the same calling convention；
3. 具有相同物理属性的 Relational Expression 会记录在其成员变量 `List<RelSubset> subsets` 中.

RelSet 中比较重要成员变量如下：

```java
class RelSet {
  // 记录属于这个 RelSet 的所有 RelNode
  final List<RelNode> rels = new ArrayList<>();
  /**
   * Relational expressions that have a subset in this set as a child. This
   * is a multi-set. If multiple relational expressions in this set have the
   * same parent, there will be multiple entries.
   */
  final List<RelNode> parents = new ArrayList<>();
  //note: 具体相同物理属性的子集合（本质上 RelSubset 并不记录 RelNode，也是通过 RelSet 按物理属性过滤得到其 RelNode 子集合，见下面的 RelSubset 部分）
  final List<RelSubset> subsets = new ArrayList<>();

  /**
   * List of {@link AbstractConverter} objects which have not yet been
   * satisfied.
   */
  final List<AbstractConverter> abstractConverters = new ArrayList<>();

  /**
   * Set to the superseding set when this is found to be equivalent to another
   * set.
   * note：当发现与另一个 RelSet 有相同的语义时，设置为替代集合
   */
  RelSet equivalentSet;
  RelNode rel;

  /**
   * Variables that are set by relational expressions in this set and available for use by parent and child expressions.
   * note：在这个集合中 relational expression 设置的变量，父类和子类 expression 可用的变量
   */
  final Set<CorrelationId> variablesPropagated;

  /**
   * Variables that are used by relational expressions in this set.
   * note：在这个集合中被 relational expression 使用的变量
   */
  final Set<CorrelationId> variablesUsed;
  final int id;

  /**
   * Reentrancy flag.
   */
  boolean inMetadataQuery;
}
```

##### RelSubset

关于 RelSubset，源码中介绍如下：

> Subset of an equivalence class where all relational expressions have the same physical properties.

它的特点如下：

1. 描述一组物理属性相同的等价 Relation Expression，即它们具有相同的 Physical Properties；
2. 每个 RelSubset 都会记录其所属的 RelSet；
3. RelSubset 继承自 AbstractRelNode，它也是一种 RelNode，物理属性记录在其成员变量 traitSet 中。

RelSubset 一些比较重要的成员变量如下：

```java
public class RelSubset extends AbstractRelNode {
  /**
   * cost of best known plan (it may have improved since)
   * note: 已知最佳 plan 的 cost
   */
  RelOptCost bestCost;

  /**
   * The set this subset belongs to.
   * RelSubset 所属的 RelSet，在 RelSubset 中并不记录具体的 RelNode，直接记录在 RelSet 的 rels 中
   */
  final RelSet set;

  /**
   * best known plan
   * note: 已知的最佳 plan
   */
  RelNode best;

  /**
   * Flag indicating whether this RelSubset's importance was artificially
   * boosted.
   * note: 标志这个 RelSubset 的 importance 是否是人为地提高了
   */
  boolean boosted;

  //~ Constructors -----------------------------------------------------------
  RelSubset(
      RelOptCluster cluster,
      RelSet set,
      RelTraitSet traits) {
    super(cluster, traits); // 继承自 AbstractRelNode，会记录其相应的 traits 信息
    this.set = set;
    this.boosted = false;
    assert traits.allSimple();
    computeBestCost(cluster.getPlanner()); //note: 计算 best
    recomputeDigest(); //note: 计算 digest
  }
}
```

每个 RelSubset 都将会记录其最佳 plan（`best`）和最佳 plan 的 cost（`bestCost`）信息。

##### RuleMatch

RuleMatch 是这里对 Rule 和 RelSubset 关系的一个抽象，它会记录这两者的信息。

> A match of a rule to a particular set of target relational expressions, frozen in time.

##### importance

importance 决定了在进行 Rule 优化时 Rule 应用的顺序，它是一个相对概念，在 VolcanoPlanner 中有两个 importance，分别是 RelSubset 和 RuleMatch 的 importance，这里先提前介绍一下。

###### RelSubset 的 importance

RelSubset importance 计算方法见其 api 定义（**图中的 sum 改成 Math.max{}**这个地方有误）：

[![computeImportance](http://matt33.com/images/calcite/13-compute.png)](http://matt33.com/images/calcite/13-compute.png)

举个例子：假设一个 RelSubset（记为 s0s0） 的 cost 是3，对应的 importance 是0.5，这个 RelNode 有两个输入（inputs），对应的 RelSubset 记为 s1s1、s2s2（假设 s1s1、s2s2 不再有输入 RelNode），其 cost 分别为 2和5，那么 s1s1 的 importance 为

Importance of s1s1 = 23+2+523+2+5 ⋅⋅ 0.5 = 0.1

Importance of s2s2 = 53+2+553+2+5 ⋅⋅ 0.5 = 0.25

其中，2代表的是 s1s1 的 cost，3+2+53+2+5 代表的是 s0s0 的 cost（本节点的 cost 加上其所有 input 的 cost）。下面看下其具体的代码实现（调用 RuleQueue 中的 `recompute()` 计算其 importance）：

```java
//org.apache.calcite.plan.volcano.RuleQueue
/**
 * Recomputes the importance of the given RelSubset.
 * note：重新计算指定的 RelSubset 的 importance
 * note：如果为 true，即使 subset 没有注册，也会强制 importance 更新
 *
 * @param subset RelSubset whose importance is to be recomputed
 * @param force  if true, forces an importance update even if the subset has
 *               not been registered
 */
public void recompute(RelSubset subset, boolean force) {
  Double previousImportance = subsetImportances.get(subset);
  if (previousImportance == null) { //note: subset 还没有注册的情况下
    if (!force) { //note: 如果不是强制，可以直接先返回
      // Subset has not been registered yet. Don't worry about it.
      return;
    }

    previousImportance = Double.NEGATIVE_INFINITY;
  }

  //note: 计算器 importance 值
  double importance = computeImportance(subset);
  if (previousImportance == importance) {
    return;
  }

  //note: 缓存中更新其 importance
  updateImportance(subset, importance);
}


// 计算一个节点的 importance
double computeImportance(RelSubset subset) {
  double importance;
  if (subset == planner.root) {
    // The root always has importance = 1
    //note: root RelSubset 的 importance 为1
    importance = 1.0;
  } else {
    final RelMetadataQuery mq = subset.getCluster().getMetadataQuery();

    // The importance of a subset is the max of its importance to its
    // parents
    //note: 计算其相对于 parent 的最大 importance，多个 parent 的情况下，选择一个最大值
    importance = 0.0;
    for (RelSubset parent : subset.getParentSubsets(planner)) {
      //note: 计算这个 RelSubset 相对于 parent 的 importance
      final double childImportance =
          computeImportanceOfChild(mq, subset, parent);
      //note: 选择最大的 importance
      importance = Math.max(importance, childImportance);
    }
  }
  LOGGER.trace("Importance of [{}] is {}", subset, importance);
  return importance;
}

//note：根据 cost 计算 child 相对于 parent 的 importance（这是个相对值）
private double computeImportanceOfChild(RelMetadataQuery mq, RelSubset child,
    RelSubset parent) {
  //note: 获取 parent 的 importance
  final double parentImportance = getImportance(parent);
  //note: 获取对应的 cost 信息
  final double childCost = toDouble(planner.getCost(child, mq));
  final double parentCost = toDouble(planner.getCost(parent, mq));
  double alpha = childCost / parentCost;
  if (alpha >= 1.0) {
    // child is always less important than parent
    alpha = 0.99;
  }
  //note: 根据 cost 比列计算其 importance
  final double importance = parentImportance * alpha;
  LOGGER.trace("Importance of [{}] to its parent [{}] is {} (parent importance={}, child cost={},"
      + " parent cost={})", child, parent, importance, parentImportance, childCost, parentCost);
  return importance;
}
```

在 `computeImportanceOfChild()` 中计算 RelSubset 相对于 parent RelSubset 的 importance 时，一个比较重要的地方就是如何计算 cost，关于 cost 的计算见：

```java
//org.apache.calcite.plan.volcano.VolcanoPlanner
//note: Computes the cost of a RelNode.
public RelOptCost getCost(RelNode rel, RelMetadataQuery mq) {
  assert rel != null : "pre-condition: rel != null";
  if (rel instanceof RelSubset) { //note: 如果是 RelSubset，证明是已经计算 cost 的 subset
    return ((RelSubset) rel).bestCost;
  }
  if (rel.getTraitSet().getTrait(ConventionTraitDef.INSTANCE)
      == Convention.NONE) {
    return costFactory.makeInfiniteCost(); //note: 这种情况下也会返回 infinite Cost
  }
  //note: 计算其 cost
  RelOptCost cost = mq.getNonCumulativeCost(rel);
  if (!zeroCost.isLt(cost)) { //note: cost 比0还小的情况
    // cost must be positive, so nudge it
    cost = costFactory.makeTinyCost();
  }
  //note: RelNode 的 cost 会把其 input 全部加上
  for (RelNode input : rel.getInputs()) {
    cost = cost.plus(getCost(input, mq));
  }
  return cost;
}
```

上面就是 RelSubset importance 计算的代码实现，从实现中可以发现这个特点：

1. 越靠近 root 的 RelSubset，其 importance 越大，这个带来的好处就是在优化时，会尽量先优化靠近 root 的 RelNode，这样带来的收益也会最大。

###### RuleMatch 的 importance

RuleMatch 的 importance 定义为以下两个中比较大的一个（如果对应的 RelSubset 有 importance 的情况下）：

1. 这个 RuleMatch 对应 RelSubset（这个 rule match 的 RelSubset）的 importance；
2. 输出的 RelSubset（taget RelSubset）的 importance（如果这个 RelSubset 在 VolcanoPlanner 的缓存中存在的话）。

```java
//org.apache.calcite.plan.volcano.VolcanoRuleMatch
/**
 * Computes the importance of this rule match.
 * note：计算 rule match 的 importance
 *
 * @return importance of this rule match
 */
double computeImportance() {
  assert rels[0] != null; //note: rels[0] 这个 Rule Match 对应的 RelSubset
  RelSubset subset = volcanoPlanner.getSubset(rels[0]);
  double importance = 0;
  if (subset != null) {
    //note: 获取 RelSubset 的 importance
    importance = volcanoPlanner.ruleQueue.getImportance(subset);
  }
  //note: Returns a guess as to which subset the result of this rule will belong to.
  final RelSubset targetSubset = guessSubset();
  if ((targetSubset != null) && (targetSubset != subset)) {
    // If this rule will generate a member of an equivalence class
    // which is more important, use that importance.
    //note: 获取 targetSubset 的 importance
    final double targetImportance =
        volcanoPlanner.ruleQueue.getImportance(targetSubset);
    if (targetImportance > importance) {
      importance = targetImportance;

      // If the equivalence class is cheaper than the target, bump up
      // the importance of the rule. A converter is an easy way to
      // make the plan cheaper, so we'd hate to miss this opportunity.
      //
      // REVIEW: jhyde, 2007/12/21: This rule seems to make sense, but
      // is disabled until it has been proven.
      //
      // CHECKSTYLE: IGNORE 3
      if ((subset != null)
          && subset.bestCost.isLt(targetSubset.bestCost)
          && false) { //note: 肯定不会进入
        importance *=
            targetSubset.bestCost.divideBy(subset.bestCost);
        importance = Math.min(importance, 0.99);
      }
    }
  }

  return importance;
}
```

RuleMatch 的 importance 主要是决定了在选择 RuleMatch 时，应该先处理哪一个？它本质上还是直接用的 RelSubset 的 importance。

#### 2.2.2 VolcanoPlanner 处理流程

还是以前面的示例，只不过这里把优化器换成 VolcanoPlanner 来实现，通过这个示例来详细看下 VolcanoPlanner 内部的实现逻辑。

```java
//1. 初始化 VolcanoPlanner 对象，并添加相应的 Rule
VolcanoPlanner planner = new VolcanoPlanner();
planner.addRelTraitDef(ConventionTraitDef.INSTANCE);
planner.addRelTraitDef(RelDistributionTraitDef.INSTANCE);
// 添加相应的 rule
planner.addRule(FilterJoinRule.FilterIntoJoinRule.FILTER_ON_JOIN);
planner.addRule(ReduceExpressionsRule.PROJECT_INSTANCE);
planner.addRule(PruneEmptyRules.PROJECT_INSTANCE);
// 添加相应的 ConverterRule
planner.addRule(EnumerableRules.ENUMERABLE_MERGE_JOIN_RULE);
planner.addRule(EnumerableRules.ENUMERABLE_SORT_RULE);
planner.addRule(EnumerableRules.ENUMERABLE_VALUES_RULE);
planner.addRule(EnumerableRules.ENUMERABLE_PROJECT_RULE);
planner.addRule(EnumerableRules.ENUMERABLE_FILTER_RULE);
//2. Changes a relational expression to an equivalent one with a different set of traits.
RelTraitSet desiredTraits =
    relNode.getCluster().traitSet().replace(EnumerableConvention.INSTANCE);
relNode = planner.changeTraits(relNode, desiredTraits);
//3. 通过 VolcanoPlanner 的 setRoot 方法注册相应的 RelNode，并进行相应的初始化操作
planner.setRoot(relNode);
//4. 通过动态规划算法找到 cost 最小的 plan
relNode = planner.findBestExp();
```

优化后的结果为：

```
EnumerableSort(sort0=[$0], dir0=[ASC])
  EnumerableProject(USER_ID=[$0], USER_NAME=[$1], USER_COMPANY=[$5], USER_AGE=[$2])
    EnumerableMergeJoin(condition=[=($0, $3)], joinType=[inner])
      EnumerableFilter(condition=[>($2, 30)])
        EnumerableTableScan(table=[[USERS]])
      EnumerableFilter(condition=[>($0, 10)])
        EnumerableTableScan(table=[[JOBS]])
```

在应用 VolcanoPlanner 时，整体分为以下四步：

1. 初始化 VolcanoPlanner，并添加相应的 Rule（包括 ConverterRule）；
2. 对 RelNode 做等价转换，这里只是改变其物理属性（`Convention`）；
3. 通过 VolcanoPlanner 的 `setRoot()` 方法注册相应的 RelNode，并进行相应的初始化操作；
4. 通过动态规划算法找到 cost 最小的 plan；

下面来分享一下上面的详细流程。

##### 1.VolcanoPlanner 初始化

在这里总共有三步，分别是 VolcanoPlanner 初始化，`addRelTraitDef()` 添加 RelTraitDef，`addRule()` 添加 rule，先看下 VolcanoPlanner 的初始化：

```java
//org.apache.calcite.plan.volcano.VolcanoPlanner
/**
 * Creates a uninitialized <code>VolcanoPlanner</code>. To fully initialize it, the caller must register the desired set of relations, rules, and calling conventions.
 * note: 创建一个没有初始化的 VolcanoPlanner，如果要进行初始化，调用者必须注册 set of relations、rules、calling conventions.
 */
public VolcanoPlanner() {
  this(null, null);
}

/**
 * Creates a {@code VolcanoPlanner} with a given cost factory.
 * note: 创建 VolcanoPlanner 实例，并制定 costFactory（默认为 VolcanoCost.FACTORY）
 */
public VolcanoPlanner(RelOptCostFactory costFactory, //
    Context externalContext) {
  super(costFactory == null ? VolcanoCost.FACTORY : costFactory, //
      externalContext);
  this.zeroCost = this.costFactory.makeZeroCost();
}
```

这里其实并没有做什么，只是做了一些简单的初始化，如果要想设置相应 RelTraitDef 的话，需要调用 `addRelTraitDef()` 进行添加，其实现如下：

```java
//org.apache.calcite.plan.volcano.VolcanoPlanner
//note: 添加 RelTraitDef
@Override public boolean addRelTraitDef(RelTraitDef relTraitDef) {
  return !traitDefs.contains(relTraitDef) && traitDefs.add(relTraitDef);
}
```

如果要给 VolcanoPlanner 添加 Rule 的话，需要调用 `addRule()` 进行添加，**在这个方法里重点做的一步是将具体的 RelNode 与 RelOptRuleOperand 之间的关系记录下来，记录到 `classOperands` 中**，相当于在优化时，哪个 RelNode 可以应用哪些 Rule 都是记录在这个缓存里的。其实现如下：

```java
//org.apache.calcite.plan.volcano.VolcanoPlanner
//note: 添加 rule
public boolean addRule(RelOptRule rule) {
  if (locked) {
    return false;
  }
  if (ruleSet.contains(rule)) {
    // Rule already exists.
    return false;
  }
  final boolean added = ruleSet.add(rule);
  assert added;

  final String ruleName = rule.toString();
  //note: 这里的 ruleNames 允许重复的 key 值，但是这里还是要求 rule description 保持唯一的，与 rule 一一对应
  if (ruleNames.put(ruleName, rule.getClass())) {
    Set<Class> x = ruleNames.get(ruleName);
    if (x.size() > 1) {
      throw new RuntimeException("Rule description '" + ruleName
          + "' is not unique; classes: " + x);
    }
  }

  //note: 注册一个 rule 的 description（保存在 mapDescToRule 中）
  mapRuleDescription(rule);

  // Each of this rule's operands is an 'entry point' for a rule call. Register each operand against all concrete sub-classes that could match it.
  //note: 记录每个 sub-classes 与 operand 的关系（如果能 match 的话，就记录一次）。一个 RelOptRuleOperand 只会有一个 class 与之对应，这里找的是 subclass
  for (RelOptRuleOperand operand : rule.getOperands()) {
    for (Class<? extends RelNode> subClass
        : subClasses(operand.getMatchedClass())) {
      classOperands.put(subClass, operand);
    }
  }

  // If this is a converter rule, check that it operates on one of the
  // kinds of trait we are interested in, and if so, register the rule
  // with the trait.
  //note: 对于 ConverterRule 的操作，如果其 ruleTraitDef 类型包含在我们初始化的 traitDefs 中，
  //note: 就注册这个 converterRule 到 ruleTraitDef 中
  //note: 如果不包含 ruleTraitDef，这个 ConverterRule 在本次优化的过程中是用不到的
  if (rule instanceof ConverterRule) {
    ConverterRule converterRule = (ConverterRule) rule;

    final RelTrait ruleTrait = converterRule.getInTrait();
    final RelTraitDef ruleTraitDef = ruleTrait.getTraitDef();
    if (traitDefs.contains(ruleTraitDef)) { //note: 这里注册好像也没有用到
      ruleTraitDef.registerConverterRule(this, converterRule);
    }
  }

  return true;
}
```

##### 2.RelNode changeTraits

这里分为两步：

1. 通过 RelTraitSet 的 `replace()` 方法，将 RelTraitSet 中对应的 RelTraitDef 做对应的更新，其他的 RelTrait 不变；
2. 这一步简单来说就是：Changes a relational expression to an equivalent one with a different set of traits，对相应的 RelNode 做 converter 操作，这里实际上也会做很多的内容，这部分会放在第三步讲解，主要是 `registerImpl()` 方法的实现。

##### 3.VolcanoPlanner setRoot

VolcanoPlanner 会调用 `setRoot()` 方法注册相应的 Root RelNode，并进行一系列 Volcano 必须的初始化操作，很多的操作都是在这里实现的，这里来详细看下其实现。

```java
//org.apache.calcite.plan.volcano.VolcanoPlanner
public void setRoot(RelNode rel) {
  // We're registered all the rules, and therefore RelNode classes,
  // we're interested in, and have not yet started calling metadata providers.
  // So now is a good time to tell the metadata layer what to expect.
  registerMetadataRels();

  //note: 注册相应的 RelNode，会做一系列的初始化操作, RelNode 会有对应的 RelSubset
  this.root = registerImpl(rel, null);
  if (this.originalRoot == null) {
    this.originalRoot = rel;
  }

  // Making a node the root changes its importance.
  //note: 重新计算 root subset 的 importance
  this.ruleQueue.recompute(this.root);
  //Ensures that the subset that is the root relational expression contains converters to all other subsets in its equivalence set.
  ensureRootConverters();
}
```

对于 `setRoot()` 方法来说，核心的处理流程是在 `registerImpl()` 方法中，在这个方法会进行相应的初始化操作（包括 RelNode 到 RelSubset 的转换、计算 RelSubset 的 importance 等），其他的方法在上面有相应的备注，这里我们看下 `registerImpl()` 具体做了哪些事情：

```java
//org.apache.calcite.plan.volcano.VolcanoPlanner
/**
 * Registers a new expression <code>exp</code> and queues up rule matches.
 * If <code>set</code> is not null, makes the expression part of that
 * equivalence set. If an identical expression is already registered, we
 * don't need to register this one and nor should we queue up rule matches.
 *
 * note：注册一个新的 expression；对 rule match 进行排队；
 * note：如果 set 不为 null，那么就使 expression 成为等价集合（RelSet）的一部分
 * note：rel：必须是 RelSubset 或者未注册的 RelNode
 * @param rel relational expression to register. Must be either a
 *         {@link RelSubset}, or an unregistered {@link RelNode}
 * @param set set that rel belongs to, or <code>null</code>
 * @return the equivalence-set
 */
private RelSubset registerImpl(
    RelNode rel,
    RelSet set) {
  if (rel instanceof RelSubset) { //note: 如果是 RelSubset 类型，已经注册过了
    return registerSubset(set, (RelSubset) rel); //note: 做相应的 merge
  }

  assert !isRegistered(rel) : "already been registered: " + rel;
  if (rel.getCluster().getPlanner() != this) { //note: cluster 中 planner 与这里不同
    throw new AssertionError("Relational expression " + rel
        + " belongs to a different planner than is currently being used.");
  }

  // Now is a good time to ensure that the relational expression
  // implements the interface required by its calling convention.
  //note: 确保 relational expression 可以实施其 calling convention 所需的接口
  //note: 获取 RelNode 的 RelTraitSet
  final RelTraitSet traits = rel.getTraitSet();
  //note: 获取其 ConventionTraitDef
  final Convention convention = traits.getTrait(ConventionTraitDef.INSTANCE);
  assert convention != null;
  if (!convention.getInterface().isInstance(rel)
      && !(rel instanceof Converter)) {
    throw new AssertionError("Relational expression " + rel
        + " has calling-convention " + convention
        + " but does not implement the required interface '"
        + convention.getInterface() + "' of that convention");
  }
  if (traits.size() != traitDefs.size()) {
    throw new AssertionError("Relational expression " + rel
        + " does not have the correct number of traits: " + traits.size()
        + " != " + traitDefs.size());
  }

  // Ensure that its sub-expressions are registered.
  //note: 其实现在 AbstractRelNode 对应的方法中，实际上调用的还是 ensureRegistered 方法进行注册
  //note: 将 RelNode 的所有 inputs 注册到 planner 中
  //note: 这里会递归调用 registerImpl 注册 relNode 与 RelSet，直到其 inputs 全部注册
  //note: 返回的是一个 RelSubset 类型
  rel = rel.onRegister(this);

  // Record its provenance. (Rule call may be null.)
  //note: 记录 RelNode 的来源
  if (ruleCallStack.isEmpty()) { //note: 不知道来源时
    provenanceMap.put(rel, Provenance.EMPTY);
  } else { //note: 来自 rule 触发的情况
    final VolcanoRuleCall ruleCall = ruleCallStack.peek();
    provenanceMap.put(
        rel,
        new RuleProvenance(
            ruleCall.rule,
            ImmutableList.copyOf(ruleCall.rels),
            ruleCall.id));
  }

  // If it is equivalent to an existing expression, return the set that
  // the equivalent expression belongs to.
  //note: 根据 RelNode 的 digest（摘要，全局唯一）判断其是否已经有对应的 RelSubset，有的话直接放回
  String key = rel.getDigest();
  RelNode equivExp = mapDigestToRel.get(key);
  if (equivExp == null) { //note: 还没注册的情况
    // do nothing
  } else if (equivExp == rel) {//note: 已经有其缓存信息
    return getSubset(rel);
  } else {
    assert RelOptUtil.equal(
        "left", equivExp.getRowType(),
        "right", rel.getRowType(),
        Litmus.THROW);
    RelSet equivSet = getSet(equivExp); //note: 有 RelSubset 但对应的 RelNode 不同时，这里对其 RelSet 做下 merge
    if (equivSet != null) {
      LOGGER.trace(
          "Register: rel#{} is equivalent to {}", rel.getId(), equivExp.getDescription());
      return registerSubset(set, getSubset(equivExp));
    }
  }

  //note： Converters are in the same set as their children.
  if (rel instanceof Converter) {
    final RelNode input = ((Converter) rel).getInput();
    final RelSet childSet = getSet(input);
    if ((set != null)
        && (set != childSet)
        && (set.equivalentSet == null)) {
      LOGGER.trace(
          "Register #{} {} (and merge sets, because it is a conversion)",
          rel.getId(), rel.getDigest());
      merge(set, childSet);
      registerCount++;

      // During the mergers, the child set may have changed, and since
      // we're not registered yet, we won't have been informed. So
      // check whether we are now equivalent to an existing
      // expression.
      if (fixUpInputs(rel)) {
        rel.recomputeDigest();
        key = rel.getDigest();
        RelNode equivRel = mapDigestToRel.get(key);
        if ((equivRel != rel) && (equivRel != null)) {
          assert RelOptUtil.equal(
              "rel rowtype",
              rel.getRowType(),
              "equivRel rowtype",
              equivRel.getRowType(),
              Litmus.THROW);

          // make sure this bad rel didn't get into the
          // set in any way (fixupInputs will do this but it
          // doesn't know if it should so it does it anyway)
          set.obliterateRelNode(rel);

          // There is already an equivalent expression. Use that
          // one, and forget about this one.
          return getSubset(equivRel);
        }
      }
    } else {
      set = childSet;
    }
  }

  // Place the expression in the appropriate equivalence set.
  //note: 把 expression 放到合适的 等价集 中
  //note: 如果 RelSet 不存在，这里会初始化一个 RelSet
  if (set == null) {
    set = new RelSet(
        nextSetId++,
        Util.minus(
            RelOptUtil.getVariablesSet(rel),
            rel.getVariablesSet()),
        RelOptUtil.getVariablesUsed(rel));
    this.allSets.add(set);
  }

  // Chain to find 'live' equivalent set, just in case several sets are
  // merging at the same time.
  //note: 递归查询，一直找到最开始的 语义相等的集合，防止不同集合同时被 merge
  while (set.equivalentSet != null) {
    set = set.equivalentSet;
  }

  // Allow each rel to register its own rules.
  registerClass(rel);

  registerCount++;
  //note: 初始时是 0
  final int subsetBeforeCount = set.subsets.size();
  //note: 向等价集中添加相应的 RelNode，并更新其 best 信息
  RelSubset subset = addRelToSet(rel, set);

  //note: 缓存相关信息，返回的 key 之前对应的 value
  final RelNode xx = mapDigestToRel.put(key, rel);
  assert xx == null || xx == rel : rel.getDigest();

  LOGGER.trace("Register {} in {}", rel.getDescription(), subset.getDescription());

  // This relational expression may have been registered while we
  // recursively registered its children. If this is the case, we're done.
  if (xx != null) {
    return subset;
  }

  // Create back-links from its children, which makes children more
  // important.
  //note: 如果是 root，初始化其 importance 为 1.0
  if (rel == this.root) {
    ruleQueue.subsetImportances.put(
        subset,
        1.0); // todo: remove
  }
  //note: 将 Rel 的 input 对应的 RelSubset 的 parents 设置为当前的 Rel
  //note: 也就是说，一个 RelNode 的 input 为其对应 RelSubset 的 children 节点
  for (RelNode input : rel.getInputs()) {
    RelSubset childSubset = (RelSubset) input;
    childSubset.set.parents.add(rel);

    // Child subset is more important now a new parent uses it.
    //note: 重新计算 RelSubset 的 importance
    ruleQueue.recompute(childSubset);
  }
  if (rel == this.root) {// TODO: 2019-03-11 这里为什么要删除呢？
    ruleQueue.subsetImportances.remove(subset);
  }

  // Remember abstract converters until they're satisfied
  //note: 如果是 AbstractConverter 示例，添加到 abstractConverters 集合中
  if (rel instanceof AbstractConverter) {
    set.abstractConverters.add((AbstractConverter) rel);
  }

  // If this set has any unsatisfied converters, try to satisfy them.
  //note: check set.abstractConverters
  checkForSatisfiedConverters(set, rel);

  // Make sure this rel's subset importance is updated
  //note: 强制更新（重新计算） subset 的 importance
  ruleQueue.recompute(subset, true);

  //note: 触发所有匹配的 rule，这里是添加到对应的 RuleQueue 中
  // Queue up all rules triggered by this relexp's creation.
  fireRules(rel, true);

  // It's a new subset.
  //note: 如果是一个 new subset，再做一次触发
  if (set.subsets.size() > subsetBeforeCount) {
    fireRules(subset, true);
  }

  return subset;
}
```

`registerImpl()` 处理流程比较复杂，其方法实现，可以简单总结为以下几步：

1. 在经过最上面的一些验证之后，会通过 `rel.onRegister(this)` 这步操作，递归地调用 VolcanoPlanner 的 `ensureRegistered()` 方法对其 `inputs` RelNode 进行注册，最后还是调用 `registerImpl()` 方法先注册叶子节点，然后再父节点，最后到根节点；
2. 根据 RelNode 的 digest 信息（一般这个对于 RelNode 来说是全局唯一的），判断其是否已经存在 `mapDigestToRel` 缓存中，如果存在的话，那么判断会 RelNode 是否相同，如果相同的话，证明之前已经注册过，直接通过 `getSubset()` 返回其对应的 RelSubset 信息，否则就对其 RelSubset 做下 merge；
3. 如果 RelNode 对应的 RelSet 为 null，这里会新建一个 RelSet，并通过 `addRelToSet()` 将 RelNode 添加到 RelSet 中，并且更新 VolcanoPlanner 的 `mapRel2Subset` 缓存记录（RelNode 与 RelSubset 的对应关系），在 `addRelToSet()` 的最后还会更新 RelSubset 的 best plan 和 best cost（每当往一个 RelSubset 添加相应的 RelNode 时，都会判断这个 RelNode 是否代表了 best plan，如果是的话，就更新）；
4. 将这个 RelNode 的 inputs 设置为其对应 RelSubset 的 children 节点（实际的操作时，是在 RelSet 的 `parents` 中记录其父节点）；
5. 强制重新计算当前 RelNode 对应 RelSubset 的 importance；
6. 如果这个 RelSubset 是新建的，会再触发一次 `fireRules()` 方法（会先对 RelNode 触发一次），遍历找到所有可以 match 的 Rule，对每个 Rule 都会创建一个 VolcanoRuleMatch 对象（会记录 RelNode、RelOptRuleOperand 等信息，RelOptRuleOperand 中又会记录 Rule 的信息），并将这个 VolcanoRuleMatch 添加到对应的 RuleQueue 中（就是前面图中的那个 RuleQueue）。

这里，来看下 `fireRules()` 方法的实现，它的目的是把配置的 RuleMatch 添加到 RuleQueue 中，其实现如下：

```java
//org.apache.calcite.plan.volcano.VolcanoPlanner
/**
 * Fires all rules matched by a relational expression.
 * note： 触发满足这个 relational expression 的所有 rules
 *
 * @param rel      Relational expression which has just been created (or maybe
 *                 from the queue)
 * @param deferred If true, each time a rule matches, just add an entry to
 *                 the queue.
 */
void fireRules(
    RelNode rel,
    boolean deferred) {
  for (RelOptRuleOperand operand : classOperands.get(rel.getClass())) {
    if (operand.matches(rel)) { //note: rule 匹配的情况
      final VolcanoRuleCall ruleCall;
      if (deferred) { //note: 这里默认都是 true，会把 RuleMatch 添加到 queue 中
        ruleCall = new DeferringRuleCall(this, operand);
      } else {
        ruleCall = new VolcanoRuleCall(this, operand);
      }
      ruleCall.match(rel);
    }
  }
}

/**
 * A rule call which defers its actions. Whereas {@link RelOptRuleCall}
 * invokes the rule when it finds a match, a <code>DeferringRuleCall</code>
 * creates a {@link VolcanoRuleMatch} which can be invoked later.
 */
private static class DeferringRuleCall extends VolcanoRuleCall {
  DeferringRuleCall(
      VolcanoPlanner planner,
      RelOptRuleOperand operand) {
    super(planner, operand);
  }

  /**
   * Rather than invoking the rule (as the base method does), creates a
   * {@link VolcanoRuleMatch} which can be invoked later.
   * note：不是直接触发 rule，而是创建一个后续可以被触发的 VolcanoRuleMatch
   */
  protected void onMatch() {
    final VolcanoRuleMatch match =
        new VolcanoRuleMatch(
            volcanoPlanner,
            getOperand0(), //note: 其实就是 operand
            rels,
            nodeInputs);
    volcanoPlanner.ruleQueue.addMatch(match);
  }
}
```

在上面的方法中，对于匹配的 Rule，将会创建一个 VolcanoRuleMatch 对象，之后再把这个 VolcanoRuleMatch 对象添加到对应的 RuleQueue 中。

```java
//org.apache.calcite.plan.volcano.RuleQueue
/**
 * Adds a rule match. The rule-matches are automatically added to all
 * existing {@link PhaseMatchList per-phase rule-match lists} which allow
 * the rule referenced by the match.
 * note：添加一个 rule match（添加到所有现存的 match phase 中）
 */
void addMatch(VolcanoRuleMatch match) {
  final String matchName = match.toString();
  for (PhaseMatchList matchList : matchListMap.values()) {
    if (!matchList.names.add(matchName)) {
      // Identical match has already been added.
      continue;
    }

    String ruleClassName = match.getRule().getClass().getSimpleName();

    Set<String> phaseRuleSet = phaseRuleMapping.get(matchList.phase);
    //note: 如果 phaseRuleSet 不为 ALL_RULES，并且 phaseRuleSet 不包含这个 ruleClassName 时，就跳过(其他三个阶段都属于这个情况)
    //note: 在添加 rule match 时，phaseRuleSet 可以控制哪些 match 可以添加、哪些不能添加
    //note: 这里的话，默认只有处在 OPTIMIZE 阶段的 PhaseMatchList 可以添加相应的 rule match
    if (phaseRuleSet != ALL_RULES) {
      if (!phaseRuleSet.contains(ruleClassName)) {
        continue;
      }
    }

    LOGGER.trace("{} Rule-match queued: {}", matchList.phase.toString(), matchName);

    matchList.list.add(match);

    matchList.matchMap.put(
        planner.getSubset(match.rels[0]), match);
  }
}
```

到这里 VolcanoPlanner 需要初始化的内容都初始化完成了，下面就到了具体的优化部分。

##### 4.VolcanoPlanner findBestExp

VolcanoPlanner 的 `findBestExp()` 是具体进行优化的地方，先介绍一下这里的优化策略（每进行一次迭代，`cumulativeTicks` 加1，它记录了总的迭代次数）：

1. 第一次找到可执行计划的迭代次数记为 `firstFiniteTick`，其对应的 Cost 暂时记为 BestCost；
2. 制定下一次优化要达到的目标为 `BestCost*0.9`，再根据 `firstFiniteTick` 及当前的迭代次数计算 `giveUpTick`，这个值代表的意思是：如果迭代次数超过这个值还没有达到优化目标，那么将会放弃迭代，认为当前的 plan 就是 best plan；
3. 如果 RuleQueue 中 RuleMatch 为空，那么也会退出迭代，认为当前的 plan 就是 best plan；
4. 在每次迭代时都会从 RuleQueue 中选择一个 RuleMatch，策略是选择一个最高 importance 的 RuleMatch，可以保证在每次规则优化时都是选择当前优化效果最好的 Rule 去优化；
5. 最后根据 best plan，构建其对应的 RelNode。

上面就是 `findBestExp()` 主要设计理念，这里来看其具体的实现：

```java
//org.apache.calcite.plan.volcano.VolcanoPlanner
/**
 * Finds the most efficient expression to implement the query given via
 * {@link org.apache.calcite.plan.RelOptPlanner#setRoot(org.apache.calcite.rel.RelNode)}.
 *
 * note：找到最有效率的 relational expression，这个算法包含一系列阶段，每个阶段被触发的 rules 可能不同
 * <p>The algorithm executes repeatedly in a series of phases. In each phase
 * the exact rules that may be fired varies. The mapping of phases to rule
 * sets is maintained in the {@link #ruleQueue}.
 *
 * note：在每个阶段，planner 都会初始化这个 RelSubset 的 importance，planner 会遍历 rule queue 中 rules 直到：
 * note：1. rule queue 变为空；
 * note：2. 对于 ambitious planner，最近 cost 不再提高时（具体来说，第一次找到一个可执行计划时，需要达到需要迭代总数的10%或更大）；
 * note：3. 对于 non-ambitious planner，当找到一个可执行的计划就行；
 * <p>In each phase, the planner sets the initial importance of the existing
 * RelSubSets ({@link #setInitialImportance()}). The planner then iterates
 * over the rule matches presented by the rule queue until:
 *
 * <ol>
 * <li>The rule queue becomes empty.</li>
 * <li>For ambitious planners: No improvements to the plan have been made
 * recently (specifically within a number of iterations that is 10% of the
 * number of iterations necessary to first reach an implementable plan or 25
 * iterations whichever is larger).</li>
 * <li>For non-ambitious planners: When an implementable plan is found.</li>
 * </ol>
 *
 * note：此外，如果每10次迭代之后，没有一个可实现的计划，包含 logical RelNode 的 RelSubSets 将会通过 injectImportanceBoost 给一个 importance；
 * <p>Furthermore, after every 10 iterations without an implementable plan,
 * RelSubSets that contain only logical RelNodes are given an importance
 * boost via {@link #injectImportanceBoost()}. Once an implementable plan is
 * found, the artificially raised importance values are cleared (see
 * {@link #clearImportanceBoost()}).
 *
 * @return the most efficient RelNode tree found for implementing the given
 * query
 */
public RelNode findBestExp() {
  //note: 确保 root relational expression 的 subset（RelSubset）在它的等价集（RelSet）中包含所有 RelSubset 的 converter
  //note: 来保证 planner 从其他的 subsets 找到的实现方案可以转换为 root，否则可能因为 convention 不同，无法实施
  ensureRootConverters();
  //note: materialized views 相关，这里可以先忽略~
  registerMaterializations();
  int cumulativeTicks = 0; //note: 四个阶段通用的变量
  //note: 不同的阶段，总共四个阶段，实际上只有 OPTIMIZE 这个阶段有效，因为其他阶段不会有 RuleMatch
  for (VolcanoPlannerPhase phase : VolcanoPlannerPhase.values()) {
    //note: 在不同的阶段，初始化 RelSubSets 相应的 importance
    //note: root 节点往下子节点的 importance 都会被初始化
    setInitialImportance();

    //note: 默认是 VolcanoCost
    RelOptCost targetCost = costFactory.makeHugeCost();
    int tick = 0;
    int firstFiniteTick = -1;
    int splitCount = 0;
    int giveUpTick = Integer.MAX_VALUE;

    while (true) {
      ++tick;
      ++cumulativeTicks;
      //note: 第一次运行是 false，两个不是一个对象，一个是 costFactory.makeHugeCost， 一个是 costFactory.makeInfiniteCost
      //note: 如果低于目标 cost，这里再重新设置一个新目标、新的 giveUpTick
      if (root.bestCost.isLe(targetCost)) {
        //note: 本阶段第一次运行，目的是为了调用 clearImportanceBoost 方法，清除相应的 importance 信息
        if (firstFiniteTick < 0) {
          firstFiniteTick = cumulativeTicks;

          //note: 对于那些手动提高 importance 的 RelSubset 进行重新计算
          clearImportanceBoost();
        }
        if (ambitious) {
          // Choose a slightly more ambitious target cost, and
          // try again. If it took us 1000 iterations to find our
          // first finite plan, give ourselves another 100
          // iterations to reduce the cost by 10%.
          //note: 设置 target 为当前 best cost 的 0.9，调整相应的目标，再进行优化
          targetCost = root.bestCost.multiplyBy(0.9);
          ++splitCount;
          if (impatient) {
            if (firstFiniteTick < 10) {
              // It's possible pre-processing can create
              // an implementable plan -- give us some time
              // to actually optimize it.
              //note: 有可能在 pre-processing 阶段就实现一个 implementable plan，所以先设置一个值，后面再去优化
              giveUpTick = cumulativeTicks + 25;
            } else {
              giveUpTick =
                  cumulativeTicks
                      + Math.max(firstFiniteTick / 10, 25);
            }
          }
        } else {
          break;
        }
      //note: 最近没有任何进步（超过 giveUpTick 限制，还没达到目标值），直接采用当前的 best plan
      } else if (cumulativeTicks > giveUpTick) {
        // We haven't made progress recently. Take the current best.
        break;
      } else if (root.bestCost.isInfinite() && ((tick % 10) == 0)) {
        injectImportanceBoost();
      }

      LOGGER.debug("PLANNER = {}; TICK = {}/{}; PHASE = {}; COST = {}",
          this, cumulativeTicks, tick, phase.toString(), root.bestCost);

      VolcanoRuleMatch match = ruleQueue.popMatch(phase);
      //note: 如果没有规则，会直接退出当前的阶段
      if (match == null) {
        break;
      }

      assert match.getRule().matches(match);
      //note: 做相应的规则匹配
      match.onMatch();

      // The root may have been merged with another
      // subset. Find the new root subset.
      root = canonize(root);
    }

    //note: 当期阶段完成，移除 ruleQueue 中记录的 rule-match list
    ruleQueue.phaseCompleted(phase);
  }
  if (LOGGER.isTraceEnabled()) {
    StringWriter sw = new StringWriter();
    final PrintWriter pw = new PrintWriter(sw);
    dump(pw);
    pw.flush();
    LOGGER.trace(sw.toString());
  }
  //note: 根据 plan 构建其 RelNode 树
  RelNode cheapest = root.buildCheapestPlan(this);
  if (LOGGER.isDebugEnabled()) {
    LOGGER.debug(
        "Cheapest plan:\n{}", RelOptUtil.toString(cheapest, SqlExplainLevel.ALL_ATTRIBUTES));

    LOGGER.debug("Provenance:\n{}", provenance(cheapest));
  }
  return cheapest;
}
```

整体的流程正如前面所述，这里来看下 RuleQueue 中 `popMatch()` 方法的实现，它的目的是选择 the highest importance 的 RuleMatch，这个方法的实现如下：

```java
//org.apache.calcite.plan.volcano.RuleQueue
/**
 * Removes the rule match with the highest importance, and returns it.
 *
 * note：返回最高 importance 的 rule，并从 Rule Match 中移除（处理过后的就移除）
 * note：如果集合为空，就返回 null
 * <p>Returns {@code null} if there are no more matches.</p>
 *
 * <p>Note that the VolcanoPlanner may still decide to reject rule matches
 * which have become invalid, say if one of their operands belongs to an
 * obsolete set or has importance=0.
 *
 * @throws java.lang.AssertionError if this method is called with a phase
 *                              previously marked as completed via
 *                              {@link #phaseCompleted(VolcanoPlannerPhase)}.
 */
VolcanoRuleMatch popMatch(VolcanoPlannerPhase phase) {
  dump();

  //note: 选择当前阶段对应的 PhaseMatchList
  PhaseMatchList phaseMatchList = matchListMap.get(phase);
  if (phaseMatchList == null) {
    throw new AssertionError("Used match list for phase " + phase
        + " after phase complete");
  }

  final List<VolcanoRuleMatch> matchList = phaseMatchList.list;
  VolcanoRuleMatch match;
  for (;;) {
    //note: 按照前面的逻辑只有在 OPTIMIZE 阶段，PhaseMatchList 才不为空，其他阶段都是空
    // 参考 addMatch 方法
    if (matchList.isEmpty()) {
      return null;
    }
    if (LOGGER.isTraceEnabled()) {
      matchList.sort(MATCH_COMPARATOR);
      match = matchList.remove(0);

      StringBuilder b = new StringBuilder();
      b.append("Sorted rule queue:");
      for (VolcanoRuleMatch match2 : matchList) {
        final double importance = match2.computeImportance();
        b.append("\n");
        b.append(match2);
        b.append(" importance ");
        b.append(importance);
      }

      LOGGER.trace(b.toString());
    } else { //note: 直接遍历找到 importance 最大的 match（上面先做排序，是为了输出日志）
      // If we're not tracing, it's not worth the effort of sorting the
      // list to find the minimum.
      match = null;
      int bestPos = -1;
      int i = -1;
      for (VolcanoRuleMatch match2 : matchList) {
        ++i;
        if (match == null
            || MATCH_COMPARATOR.compare(match2, match) < 0) {
          bestPos = i;
          match = match2;
        }
      }
      match = matchList.remove(bestPos);
    }

    if (skipMatch(match)) {
      LOGGER.debug("Skip match: {}", match);
    } else {
      break;
    }
  }

  // A rule match's digest is composed of the operand RelNodes' digests,
  // which may have changed if sets have merged since the rule match was
  // enqueued.
  //note: 重新计算一下这个 RuleMatch 的 digest
  match.recomputeDigest();

  //note: 从 phaseMatchList 移除这个 RuleMatch
  phaseMatchList.matchMap.remove(
      planner.getSubset(match.rels[0]), match);

  LOGGER.debug("Pop match: {}", match);
  return match;
}
```

到这里，我们就把 VolcanoPlanner 的优化讲述完了，当然并没有面面俱到所有的细节，VolcanoPlanner 的整体处理图如下：

![](http://matt33.com/images/calcite/14-volcano.png)

#### 2.2.3 思考

1. 初始化 RuleQueue 时，添加的 one useless rule name 有什么用？

   在初始化 RuleQueue 时，会给 VolcanoPlanner 的四个阶段 `PRE_PROCESS_MDR, PRE_PROCESS, OPTIMIZE, CLEANUP` 都初始化一个 PhaseMatchList 对象（记录这个阶段对应的 RuleMatch），这时候会给其中的三个阶段添加一个 useless rule，如下所示：

   ```java
   protected VolcanoPlannerPhaseRuleMappingInitializer
       getPhaseRuleMappingInitializer() {
     return phaseRuleMap -> {
       // Disable all phases except OPTIMIZE by adding one useless rule name.
       //note: 通过添加一个无用的 rule name 来 disable 优化器的其他三个阶段
       phaseRuleMap.get(VolcanoPlannerPhase.PRE_PROCESS_MDR).add("xxx");
       phaseRuleMap.get(VolcanoPlannerPhase.PRE_PROCESS).add("xxx");
       phaseRuleMap.get(VolcanoPlannerPhase.CLEANUP).add("xxx");
     };
   }
   ```

   开始时还困惑这个什么用？后来看到下面的代码基本就明白了

   ```java
   for (VolcanoPlannerPhase phase : VolcanoPlannerPhase.values()) {
     // empty phases get converted to "all rules"
     //note: 如果阶段对应的 rule set 为空，那么就给这个阶段对应的 rule set 添加一个 【ALL_RULES】
     //也就是只有 OPTIMIZE 这个阶段对应的会添加 ALL_RULES
     if (phaseRuleMapping.get(phase).isEmpty()) {
       phaseRuleMapping.put(phase, ALL_RULES);
     }
   }
   ```

   

   后面在调用 RuleQueue 的 `addMatch()` 方法会做相应的判断，如果 phaseRuleSet 不为 ALL_RULES，并且 phaseRuleSet 不包含这个 ruleClassName 时，那么就跳过这个 RuleMatch，也就是说实际上只有 **OPTIMIZE** 这个阶段是发挥作用的，其他阶段没有添加任何 RuleMatch。

2. 四个 phase 实际上只用了 1个阶段，为什么要设置4个阶段？

   VolcanoPlanner 的四个阶段 `PRE_PROCESS_MDR, PRE_PROCESS, OPTIMIZE, CLEANUP`，实际只有 `OPTIMIZE` 进行真正的优化操作，其他阶段并没有，这里自己是有一些困惑的：

3. 为什么要分为4个阶段，在添加 RuleMatch 时，是向四个阶段同时添加，这个设计有什么好处？为什么要优化四次？

4. 设计了4个阶段，为什么默认只用了1个？

这两个问题，暂时也没有头绪，有想法的，欢迎交流。

