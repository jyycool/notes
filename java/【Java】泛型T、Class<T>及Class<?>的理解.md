# 【Java Generic】泛型T、Class\<T>及Class<?>的理解

#Lang/Java

首先看下Class类 ,普通的非泛型类Class。

> 注意：class是java的关键字, 在声明java类时使用;

Class类的实例表示Java应用运行时的类(class ans enum)或接口(interface and annotation)                               

每个java类运行时都在JVM里表现为一个Class对象，可通过类名.class,类型.getClass(),Class.forName("类名")等方法获取Class对象）。

数组同样也被映射为为Class 对象的一个类，所有具有相同元素类型和维数的数组都共享该 Class 对象。基本类型boolean，byte，char，short，int，long，float，double和关键字void同样表现为 Class  对象。

T  bean ;

Class\<T> bean;

Class<?> bean;

在利用反射获取属性时，遇到这样的写法，对此专门查些资料研究了一下。

单独的T 代表一个类型, 而 Class\<T>和Class<?>代表这个类型所对应的类

Class\<T>在实例化的时候，T要替换成具体类
Class<?>它是个通配泛型，?可以代表任何类型   

<? extends T>受限统配，表示T的一个未知子类。
<? super T>下限统配，表示T的一个未知父类。

public T find(Class\<T> clazz, int id);
根据类来反射生成一个实例，而单独用T没法做到。

Object类中包含一个方法名叫getClass，利用这个方法就可以获得一个实例的类型类。类型类指的是代表一个类型的类，因为一切皆是对象，类型也不例外，在Java使用类型类来表示一个类型。所有的类型类都是Class类的实例。getClass() 会看到返回Class<?>。

JDK中，普通的Class.newInstance()方法的定义返回Object，要将该返回类型强制转换为另一种类型;

但是使用泛型的Class\<T>，Class.newInstance()方法具有一个特定的返回类型;


