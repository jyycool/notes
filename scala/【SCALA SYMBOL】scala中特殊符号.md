## 【SCALA SYMBOL】scala中特殊符号



### 连接符 @

scala中 `@` 除了是 ***注解*** 以及 ***抽取XML的属性*** 功能以外 , 还有一个功能就是模式匹配中 ***匹配类似 some的值***

#### 示例

```scala
package cgs.scala.symbol

/**
 * @Project: scala_lecture
 * @Description: TODO
 * @author: sherlock
 * @date Date:
 *      @ 可以在模式匹配中 匹配类似 some的值
 */
object Connection extends App {

	val d @ (c @Some(a), Some(b)) = (Some(1), Some(2))
	println(s"d = ${d} \nc = ${c} \na = ${a} \nb = ${b}")

	println("-" * 30)

	(Some(1), Some(2)) match {
		case d@(c@Some(a), Some(b)) =>
			println(s"d = ${d} \nc = ${c} \na = ${a} \nb = ${b}")
	}

	println("-" * 30)

	for (x @ Some(y) <- Seq(None, Some(1)))
		println(x, y)

	println("-" * 30)

	val List(x, xs@_*) = List(1, 2, 3)
	println(x)
	println(xs)
}
/*
  执行结果:
  d = (Some(1),Some(2)) 
  c = Some(1) 
  a = 1 
  b = 2
  ------------------------------
  d = (Some(1),Some(2)) 
  c = Some(1) 
  a = 1 
  b = 2
  ------------------------------
  (Some(1),1)
  ------------------------------
  1
  List(2, 3)
*/
```

