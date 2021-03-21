# 【Json】Json解析工具--Json4s

在 Java 中对象与json对象之间转换通常都会使用阿里巴巴的 fastjson .

而在 Scala 中其实有一款轻量级小众的很优秀的 Json 工具 Json4s.

### Json4s介绍

Json4s下面有两个类型的包

1. ***Lift Json***

   ```scala
   import org.json4s._
   import org.json4s.native.JsonMethods._
   ```

   ***native package具有lift-json的所有功能。***

   

2. ***Jackson***

   ```scala
   import org.json4s._
   import org.josn4s.jackson.JsonMethods._
   ```

   ***与native package不同的是，jackson包含了大部分jackson-module-scala的功能，也可以使用lift-json下的所有功能。在这个项目中我们通常使用jackson。***



### Json AST

lift-json library 的核心就是 ***Json AST***，它将Json对象转换成了语法树。

```tsx
sealed abstract class JValue
case object JNothing extends JValue // 'zero' for JValue
case object JNull extends JValue
case class JString(s: String) extends JValue
case class JDouble(num: Double) extends JValue
case class JDecimal(num: BigDecimal) extends JValue
case class JInt(num: BigInt) extends JValue
case class JBool(value: Boolean) extends JValue
case class JObject(obj: List[JField]) extends JValue
case class JArray(arr: List[JValue]) extends JValue

type JField = (String, JValue)
```

转换的常见方式有：

- ***Json DSL(implicit)***
- ***String parse***
- ***case class***

下面一一介绍。

### Json4s安装

1. ***SBT***

   - ***native support & jackson support***

     ```kotlin
     val json4sNative = "org.json4s" %% "json4s-native" % "{latestVersion}"
     val json4sJackson = "org.json4s" %% "json4s-jackson" % "{latestVersion}"
     ```

2. ***Maven***

   - ***native support & jackson support***

     ```xml
     <dependency>
       <groupId>org.json4s</groupId>
       <artifactId>json4s-native_${scala.binary}</artifactId>
       <version>{latestVersion}</version>
     </dependency>
     
     <dependency>
       <groupId>org.json4s</groupId>
       <artifactId>json4s-jackson_${scala.binary}</artifactId>
       <version>{latestVersion}</version>
     </dependency>
     ```

### Json转换

这里我们都以 jackson 为例，native support 请参考 [官网](http://json4s.org/)
 一个有效的 json 对象可以转换成AST格式

#### String parse

```python
import org.json4s._
import org.json4s.jackson.JsonMethods._
parse("""{ "numbers" : [1, 2, 3, 4] }""")
parse("""{"name":"Toy","price":35.35}""", useBigDecimalForDouble = true)
```

#### DSL Parse

DSL规则; 

- 基本数据类型映射到`JSON`的基本数据类型
- 任何`Seq`类型映射到`JSON`的数组类型

```dart
# List to json
scala> val json = List(1, 2, 3)
scala> compact(render(json))
res0: String = [1,2,3]
  
#Map to json
scala> val json = ("name" -> "joe")
scala> compact(render(json))
res1: String = {"name":"joe"}

#当有多行数据时，用~符号连接
scala> val json = ("name" -> "joe") ~ ("age" -> 35)
scala> compact(render(json))
res2: String = {"name":"joe","age":35}
```

- 自定义DSL必须实现下面的隐式转换：

```kotlin
type DslConversion = T => JValue
```



#### case class

当你有自定义的数据类型想进行转换时，可以使用下面的方法

```tsx
type DslConversion = T => JValue
```

```kotlin
import org.json4s._
import org.json4s.JsonDSL._
import org.json4s.jackson.JsonMethods._

object JsonExample extends App {
  
  #定义类 Winner, Lotto
  case class Winner(id: Long, numbers: List[Int])
  case class Lotto(id: Long, winningNumbers: List[Int], winners: List[Winner], drawDate: Option[java.util.Date])

  val winners = List(Winner(23, List(2, 45, 34, 23, 3, 5)), Winner(54, List(52, 3, 12, 11, 18, 22)))
  val lotto = Lotto(5, List(2, 45, 34, 23, 7, 5, 3), winners, None)
  
  #转换json
  val json =
    ("lotto" ->
      ("lotto-id" -> lotto.id) ~
      ("winning-numbers" -> lotto.winningNumbers) ~
      ("draw-date" -> lotto.drawDate.map(_.toString)) ~
      ("winners" ->
        lotto.winners.map { w =>
          (("winner-id" -> w.id) ~
           ("numbers" -> w.numbers))}))

  println(compact(render(json)))
}
```

```csharp
scala> JsonExample
{"lotto":{"lotto-id":5,"winning-numbers":[2,45,34,23,7,5,3],"winners":
[{"winner-id":23,"numbers":[2,45,34,23,3,5]},{"winner-id":54,"numbers":[52,3,12,11,18,22]}]}}
```

如果想将结果打印出来，可以使用

```bash
scala> pretty(render(JsonExample.json))

{
  "lotto":{
    "lotto-id":5,
    "winning-numbers":[2,45,34,23,7,5,3],
    "winners":[{
      "winner-id":23,
      "numbers":[2,45,34,23,3,5]
    },{
      "winner-id":54,
      "numbers":[52,3,12,11,18,22]
    }]
  }
}
```

### Json 合并与比较

两个JSON对象之间还可以进行合并与比较：

```scala
scala> import org.json4s._

scala> import org.json4s.jackson.JsonMethods._

scala> val lotto1 = parse("""{
         "lotto":{
           "lotto-id":5,
           "winning-numbers":[2,45,34,23,7,5,3]
           "winners":[{
             "winner-id":23,
             "numbers":[2,45,34,23,3,5]
           }]
         }
       }""")

scala> val lotto2 = parse("""{
         "lotto":{
           "winners":[{
             "winner-id":54,
             "numbers":[52,3,12,11,18,22]
           }]
         }
       }""")

scala> val mergedLotto = lotto1 merge lotto2
scala> pretty(render(mergedLotto))
res0: String =
{
  "lotto":{
    "lotto-id":5,
    "winning-numbers":[2,45,34,23,7,5,3],
    "winners":[{
      "winner-id":23,
      "numbers":[2,45,34,23,3,5]
    },{
      "winner-id":54,
      "numbers":[52,3,12,11,18,22]
    }]
  }
}

scala> val Diff(changed, added, deleted) = mergedLotto diff lotto1
changed: org.json4s.JsonAST.JValue = JNothing
added: org.json4s.JsonAST.JValue = JNothing
deleted: org.json4s.JsonAST.JValue = JObject(List((lotto,JObject(List(JField(winners,
JArray(List(JObject(List((winner-id,JInt(54)), (numbers,JArray(
List(JInt(52), JInt(3), JInt(12), JInt(11), JInt(18), JInt(22))))))))))))))
```

### 查询Json

#### LINQ(继承语言查询)模式

JSON中的值可以通过for生成表达式来提取。

```scala
scala> import org.json4s._
scala> import org.json4s.native.JsonMethods._
scala> val json = parse("""
         { "name": "joe",
           "children": [
             {
               "name": "Mary",
               "age": 5
             },
             {
               "name": "Mazy",
               "age": 3
             }
           ]
         }
       """)

scala> for {
         JObject(child) <- json
         JField("age", JInt(age))  <- child
       } yield age
res0: List[BigInt] = List(5, 3)

scala> for {
         JObject(child) <- json
         JField("name", JString(name)) <- child
         JField("age", JInt(age)) <- child
         if age > 4
       } yield (name, age)
res1: List[(String, BigInt)] = List((Mary,5))
```

#### XPATH + HOFs

JSON AST还支持像XPath一样进行查询：

```scala
The example json is:

{
  "person": {
    "name": "Joe",
    "age": 35,
    "spouse": {
      "person": {
        "name": "Marilyn"
        "age": 33
      }
    }
  }
}

Translated to DSL syntax:

scala> import org.json4s._

scala> import org.json4s.native.JsonMethods._

or 

scala> import org.json4s.jackson.JsonMethods._

scala> import org.json4s.JsonDSL._

scala> val json =
  ("person" ->
    ("name" -> "Joe") ~
    ("age" -> 35) ~
    ("spouse" ->
      ("person" ->
        ("name" -> "Marilyn") ~
        ("age" -> 33)
      )
    )
  )

scala> json \\ "spouse"
res0: org.json4s.JsonAST.JValue = JObject(List(
      (person,JObject(List((name,JString(Marilyn)), (age,JInt(33)))))))

scala> compact(render(res0))
res1: String = {"person":{"name":"Marilyn","age":33}}

scala> compact(render(json \\ "name"))
res2: String = {"name":"Joe","name":"Marilyn"}

scala> compact(render((json removeField { _ == JField("name", JString("Marilyn")) }) \\ "name"))
res3: String = {"name":"Joe"}

scala> compact(render(json \ "person" \ "name"))
res4: String = "Joe"

scala> compact(render(json \ "person" \ "spouse" \ "person" \ "name"))
res5: String = "Marilyn"

scala> json findField {
         case JField("name", _) => true
         case _ => false
       }
res6: Option[org.json4s.JsonAST.JValue] = Some((name,JString(Joe)))

scala> json filterField {
         case JField("name", _) => true
         case _ => false
       }
res7: List[org.json4s.JsonAST.JField] = List(JField(name,JString(Joe)), JField(name,JString(Marilyn)))

scala> json transformField {
         case JField("name", JString(s)) => ("NAME", JString(s.toUpperCase))
       }
res8: org.json4s.JsonAST.JValue = JObject(List((person,JObject(List(
(NAME,JString(JOE)), (age,JInt(35)), (spouse,JObject(List(
(person,JObject(List((NAME,JString(MARILYN)), (age,JInt(33)))))))))))))

scala> json.values
res8: scala.collection.immutable.Map[String,Any] = Map(person -> Map(name -> Joe, age -> 35, spouse -> Map(person -> Map(name -> Marilyn, age -> 33))))
```

数组元素可以通过索引获取，还可以获取指定类型的元素：

```scala
scala> val json = parse("""
         { "name": "joe",
           "children": [
             {
               "name": "Mary",
               "age": 5
             },
             {
               "name": "Mazy",
               "age": 3
             }
           ]
         }
       """)

scala> (json \ "children")(0)
res0: org.json4s.JsonAST.JValue = JObject(List((name,JString(Mary)), (age,JInt(5))))

scala> (json \ "children")(1) \ "name"
res1: org.json4s.JsonAST.JValue = JString(Mazy)

scala> json \\ classOf[JInt]
res2: List[org.json4s.JsonAST.JInt#Values] = List(5, 3)

scala> json \ "children" \\ classOf[JString]
res3: List[org.json4s.JsonAST.JString#Values] = List(Mary, Mazy)
```

### 提取值

样式类也可以用来从JSON对象中提取值：

```scala
scala> import org.json4s._
scala> import org.json4s.jackson.JsonMethods._
scala> implicit val formats = DefaultFormats // Brings in default date formats etc.
scala> case class Child(name: String, age: Int, birthdate: Option[java.util.Date])
scala> case class Address(street: String, city: String)
scala> case class Person(name: String, address: Address, children: List[Child])
scala> val json = parse("""
         { "name": "joe",
           "address": {
             "street": "Bulevard",
             "city": "Helsinki"
           },
           "children": [
             {
               "name": "Mary",
               "age": 5,
               "birthdate": "2004-09-04T18:06:22Z"
             },
             {
               "name": "Mazy",
               "age": 3
             }
           ]
         }
       """)
```

默认情况下构造器参数必须与json的字段匹配，如果json中的字段名在scala不合法，可以这样解决：

- 使用反引号

```scala
scala> case class Person(`first-name`: String)
```

- 使用transform转换函数

```scala
scala> case class Person(firstname: String)
scala> json transformField {
         case ("first-name", x) => ("firstname", x)
       }
```

如果 case class 具有辅助构造函数，提取函数会尽量匹配最佳的构造函数：

```scala
scala> case class Bike(make: String, price: Int) {
         def this(price: Int) = this("Trek", price)
       }
scala> parse(""" {"price":350} """).extract[Bike]
res0: Bike = Bike(Trek,350)
```

Scala基本类型的值可以直接从JSON中提取：

```scala
scala> (json \ "name").extract[String]
res0: String = "joe"

scala> ((json \ "children")(0) \ "birthdate").extract[Date]
res1: java.util.Date = Sat Sep 04 21:06:22 EEST 2004
```

日期格式可以通过重写DefaultFormats方法实现：

```scala
scala> implicit val formats = new DefaultFormats {
         override def dateFormatter = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
       }
```

JSON对象也可以转换成Map[String, _]，每个字段将转换成键值对：

```scala
scala> val json = parse("""
         {
           "name": "joe",
           "addresses": {
             "address1": {
               "street": "Bulevard",
               "city": "Helsinki"
             },
             "address2": {
               "street": "Soho",
               "city": "London"
             }
           }
         }""")

scala> case class PersonWithAddresses(name: String, addresses: Map[String, Address])
scala> json.extract[PersonWithAddresses]
res0: PersonWithAddresses("joe", Map("address1" -> Address("Bulevard", "Helsinki"),
                                     "address2" -> Address("Soho", "London")))
```

### 序列化

样式类可以序列化和反序列化：

```scala
scala> import org.json4s._
scala> import org.json4s.jackson.Serialization
scala> import org.json4s.jackson.Serialization.{read, write}
scala> implicit val formats = Serialization.formats(NoTypeHints)
scala> val ser = write(Child("Mary", 5, None))
scala> read[Child](ser)
res1: Child = Child(Mary,5,None)
```

序列化支持：

- 任意深度的样式类
- 所有的基本类型，包括BigInt与Symbol
- List，Seq，Array，Set以及Map
- scala.Option
- java.util.Date
- Polymorphic Lists
- 递归类型
- 只含可序列化字段的类
- 自定义序列化函数