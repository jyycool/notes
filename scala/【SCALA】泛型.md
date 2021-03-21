## 【SCALA】泛型



### 泛型限定

泛型中的常见符号: `<:`, `>:`, `<%`, `:`, `+`, `-`

| 符号        | 作用      |
| ----------- | --------- |
| ***[<:T]*** | 上边界是T |
| ***[>:T]*** | 下边界是T |
| ***[<%T]*** | 视界      |
| ***[:T]***  | 上下文    |
| ***[+T]***  | 协变      |
| ***[-T]***  | 逆变      |

### 类型变量界定(Bounds)

- `<:` 指明上界，表达了泛型的类型必须是"某种类型"或某种类型的"子类"
- `>:` 指明下界，表达了泛型的类型必须是"某种类型"或某种类型的"父类"

#### 上边界 <:

```scala
class Pair[T <: Comparable[T]](val first:T, val second:T) {
    def smaller = {
        if (first.compareTo(second) < 0) {
            println("first < second")
        } else if (first.compareTo(second) > 0) {
            println("first > second")
        } else {
            println("first = second")
        }
    }
}

val p = new Pair[String]("10","20")
p.smaller
```



#### 下边界 >:

```scala
class Father(val name: String)
class Child(name: String) extends Father(name)

def getIDCard[R >: Child](person:R) {
    if (person.getClass == classOf[Child]) {
        println("please tell us your parents' names.")
    } else if (person.getClass == classOf[Father]) {
        println("sign your name for your child's id card.")
    } else {
        println("sorry, you are not allowed to get id card.")
    }
}

val c = new Child("ALice")
getIDCard(c)
```



### 边界之视图边界(View Bounds)

- view bounds其实就是bounds 上边界的加强版本，对bounds的补充 <变成<%
- 可以利用implicit隐式转换将实参类型转换成目标类型

在上边界的例子中，`class Pair[T <: Comprable[T]]`

如果尝试 ***new Pair(2,3)***, 肯定会报错，提示Int不是Comparable[Int]的子类，Scala Int类型没有实现Comparable接口，( 注意Java的包装类型是可以的 )，但是 ***RichInt*** 实现了***Comparable[Int]***，同时还有一个 ***Int*** 到 ***RichInt*** 的隐式转换，这里就可以使用视图界定。

> 注意：视图边界用来要求一个可用的隐式转换函数（隐式视图）来将一种类型自动转换为另外一种类型，定义如下：
>
> ```Scala
> def func[A <% B](p: A) = 函数体
> 
> // 它其实等价于以下函数定义方式
> def func[A](p: A)(implicit m: A => B) = 函数体
> ```

```Scala
class Person(val name: String) {
    def sayHello = println("Hello, I'm " + name)
    def makeFriends(p: Person) {
        sayHello
        p.sayHello
    }
}

class Student(name: String) extends Person(name)
class Dog(val name: String) {
    def sayHello = println("汪汪, I'm " + name)
    implicit def dog2person(dog: Object): Person = {
        if(dog.isInstanceOf[Dog]) {
            val _dog = dog.asInstanceOf[Dog];
            new Person(_dog.name);
        } else {
            null
        }
    }
}

class Party[T <% Person](p1: T,p2: T)
```



#### 视图界定 View Bounds

我们知道 ***View  Bounds: T <% V*** 必须要求存在一个 T 到 V 的隐式转换。而上下文界定的形式就是 [T : M]，其中M是另一个泛型类，他要求必须存在一个类型为M[T]的隐式值

```scala
def func[T:S](p:T) = 函数体
```

表示这个函数参数p的类型是T，但是在调用函数func的时候，必须有一个隐式值S[T]存在，也可以这样写：

```scala 
def func[T](p:T)(implicit arg:S[T]) = 函数体
```

类型参数可以有一个形式为T:M的上下文界定，其中M是另外一个泛型类型。它要求作用域存在一个M[T]的隐式值

举个例子:

```scala
class Fraction[T:Ordering] {}
// 他要求存在一个类型为Ordering[T]的隐式值，该隐式值可以用于该类的方法中，考虑如下例子：
class Fraction[T:Ordering](val a: T, val b: T) {
    def small(implicit order:Ordering[T]) = {
        if (order.compare(a,b) < 0) println(a.toString) else println(b.toString)
    }
}


// 假设现在有一对象Model，需要传入Fraction中，那么：
class Model(val name:String, val age:Int) {
    println(s"构造对象 {$name, $age}")
}
val f = new Fraction(new Model("Shelly",28),new Model("Alice",35))
f.small

//由于需要一个Ordering[Model]的隐式值，但是我们又没有提供，所以上面编译是有问题的, 怎么办呢？既然需要Ordering[Model]的隐式值，那么我们就先创建一个Ordering[Model]的类
class ModelOrdering extends Ordering[Model]{
    override def compare(x: Model, y: Model): Int = {
        if (x.name == y.name) {
            x.age - y.age
        } else if (x.name > y.name) {
            1
        } else {
            -1
        }
    }
}
// 这个类重写compare方法,然后，编译器需要能够找到这个隐式值这个隐式值 可以是变量也可以是函数。
object ImplicitClient extends App {
    //implicit valmo = new ModelOrdering
    implicit def mo = new ModelOrdering
    /*由于需要一个Ordering[Model]的隐式值，所以一下编译是有问题的*/
    val f = new Fraction(new Model("Shelly",28),new Model("Alice",35))
    f.small
}

```

