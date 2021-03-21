# 【Scala】字符串的格式化

scala 字符串格式化 StringLike.format()

```scala
import scala.collection.immutable.StringLike

// 使用本地语言环境
format(args: Any*)
// 使用指定的语言环境
formatLocal(l: Locale, args: Any*) 
```

## 1. 各类型转换字符串

| 转 换 符 | 说  明                                      | 示  例   |
| -------- | ------------------------------------------- | -------- |
| %s       | 字符串类型                                  | "wwx"    |
| %c       | 字符类型                                    | 'w'      |
| %b       | 布尔类型                                    | true     |
| %d       | 整数类型（十进制）                          | 99       |
| %x       | 整数类型（十六进制）                        | FF       |
| %o       | 整数类型（八进制）                          | 77       |
| %f       | 浮点类型                                    | 99.99    |
| %a       | 十六进制浮点类型                            | FF.35AE  |
| %e       | 指数类型                                    | 9.668e+5 |
| %g       | 通用浮点类型（f和e类型中较短的）            |          |
| %h       | 散列码                                      |          |
| %%       | 百分比类型                                  | ％       |
| %n       | 换行符                                      |          |
| %tx      | 日期与时间类型（x代表不同的日期与时间转换符 |          |

## 2. 搭配转换符的标志

| 标  志 | 说  明                                                   | 示  例                 | 结  果           |
| ------ | -------------------------------------------------------- | ---------------------- | ---------------- |
| +      | 为正数或者负数添加符号                                   | ("%+d",112)            | +112             |
| −      | 左对齐                                                   | ("%-5d",112)           | \|112  \|        |
| 0      | 数字前面补0                                              | ("%05d", 99)           | 00099            |
| 空格   | 在整数之前添加指定数量的空格                             | ("% 5d", 99)           | \|  99\|         |
| ,      | 以“,”对数字分组                                          | ("%,f", 9999.99)       | 9,999.990000     |
| (      | 使用括号包含负数                                         | ("%(f", -99.99)        | (99.990000)      |
| #      | 如果是浮点数则包含小数点，如果是16进制或8进制则添加0x或0 | ("%#x", 99)("%#o", 99) | 0x630143         |
| <      | 格式化前一个转换符所描述的参数                           | ("%f和%<3.2f", 99.45)  | 99.450000和99.45 |
| $      | 被格式化的参数索引                                       | ("%1𝑑,d,s", 89,"abc")  | 89,abc           |

语法格式:

>%\[argument_index$]\[flags]\[width][.precision]conversion
>参数:
>
>1. argument_index
>   可选,表明参数在参数列表中的位置。第一个参数由 "1"引用，第二个参数由"2"引用，第二个参数由"2" 引用，依此类推。
>
>2. flags
>
>   可选,用来控制输出格式
>
>3. width
>
>   可选,是一个正整数，表示输出的最小长度
>
>4. precision
>
>   可选,用来限定输出字符数，精度
>
>5. conversion
>
>   必须,用来表示如何格式化参数的字符
## 3. 示例

```sh
scala> "%1$s-%2$s-%3$s".format("spark","scala","ml")
res29: String = spark-scala-ml
   
# 百分比
scala> "%d%%".format(86)
res31: String = 86%
   
# 精度为3，长度为8
scala> "%8.3f".format(11.56789)
res36: String = "  11.568"
  
scala> "%8.3f".format(11.56789).length()
res37: Int = 8
  
# 精度为3，长度为8，不足的用0填补
scala> "%08.3f".format(11.56789)
res38: String = 0011.568
  
# 长度为9，不足的用0填补
scala>  "%09d".format(11)
res23: String = 000000011
  
scala> "%.2f".format(11.23456)
res25: String = 11.23
   
scala> val wx = 36.6789
wx: Double = 36.6789

scala> f"${wx}%.2f"
res39: String = 36.68

scala> val name = "scala"
name: String = scala

scala> s"spark,$name"
res40: String = spark,scala

scala> "spark,$name"
res41: String = spark,$name

scala> s"spark\n$name"
res42: String =
spark
scala

scala> "spark\n$name"
res43: String =
spark
$name

scala> raw"spark\n$name"
res44: String = spark\nscala
```

