# 【Java】equals 和 hashCode

### 介绍

 Java *Object*类提供了方法的基本实现– *hashCode（）*和*equals（）。* 这些方法非常有用，尤其是在使用Collection框架时。 哈希表实现依赖于这些方法来存储和检索数据。 

在本教程中，我们将学习*hashCode（）*和*equals（）*之间的协定*（*它们的默认实现）。 我们还将讨论何时以及如何覆盖这些方法。 

### 默认行为

首先让我们看一下这些方法的默认实现： 

存在于 Object 类中的 equals() 方法只是比较对象引用： 

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

 因此，默认情况下， ***obj1.equals(obj2)***与 ***obj1 == obj2*** 相同。 

 *equals（）*方法比较诸如 *String* 等的类的实际值，因为它们在相应的类中被覆盖。 

 JDK 中*hashCode（）*方法的签名为： 

```
public native int hashCode();
```

 在这里， *native*关键字表示该方法是使用*JNI* （Java本机接口）以本机代码实现的。 

***hashcode方法返回该对象的哈希码值。支持该方法是为哈希表提供一些优点，例如，java.util.Hashtable 提供的哈希表。***

***hashCode() 方法返回一个int类型。默认情况下，返回值表示对象存储器地址。***

### 实施原则

 在覆盖*equals（）*和*hashCode（）*方法之前，我们先来看一下准则： 

 ***1. equals（）：***  我们对*equals（）*方法的实现必须是： 

-  **反射性：**对于任何参考值*obj* ， *obj.equals（obj）*应该返回*true* 
-  **对称性：**对于参考值*obj1*和*obj2* ，如果*obj1.equals（obj2）*为*true，*则*obj2.equals（obj2）*也应返回*true* 
-  **过渡性：**对于值*OBJ1*参考*，OBJ 2和* *OBJ 3，*如果*obj1.equals（OBJ 2）* *真实* *，obj2.equals（OBJ 3）*为*真* ，那么*obj1.equals（OBJ 3）*也应该返回*true* 
-  **一致：**只要我们没有更改实现， *equals（）*方法的多次调用必须始终返回相同的值 

 ***2. hashCode（）：***实现*hashCode（）时，*必须考虑以下几点： 

-  在一次执行中， *hashCode（）的*多次调用必须返回相同的值，前提是我们不更改*equals（）*实现中的属性 
-  相等的对象必须返回相同的*hashCode（）*值 
-  两个或更多不相等的对象可以具有相同的*hashCode（）*值 

 尽管在覆盖这些方法时要牢记上述所有原则，但是其中有一个流行的规则： 

 对于两个对象*obj1*和*obj2* ， 

-  如果*obj1.equals（obj2），*则*obj1.hashCode（）= obj2.hashCode（）*必须为*true* 
-  但是，如果*obj1.hashCode（）== obj2.hashCode（）* ，则*obj1.equals（obj2）*可以返回*true*或*false，即obj1*和*obj2*可能相等或不相等 

 ***这通常被称为 equals（）和 hashCode（）契约。***

>总的来说，Java中的集合（Collection）有两类，一类是List，再有一类是Set。前者集合内的元素是有序的，元素可以重复；后者元素无序，但元素不可重复。 
>
>要想保证元素不重复，可两个元素是否重复应该依据什么来判断呢？这就是Object.equals方法了。但是，如果每增加一个元素就检查一 次，那么当元素很多时，后添加到集合中的元素比较的次数就非常多了。也就是说，如果集合中现在已经有1000个元素，那么第1001个元素加入集合时，它 就要调用1000次equals方法。这显然会大大降低效率。 
>
>于是，Java采用了哈希表的原理。哈希算法也称为散列算法，是将数据依特定算法直接指定到一个地址上。这样一来，当集合要添加新的元素时，先调用这个元素的hashCode方法，就一下子能定位到它应该放置的物理位置上。如果这个位置上没有元素，它就可以 直接存储在这个位置上，不用再进行任何比较了；如果这个位置上已经有元素了，就调用它的equals方法与新元素进行比较，相同的话就不存了；不相同，也就是发生了Hash key相同导致冲突的情况,那么就在这个Hash  key的地方产生一个链表,将所有产生相同hashcode的对象放到这个单链表上去,串在一起。所以这里存在一个冲突解决的问题（很少出现）。这样一来实际调用equals方法的次数就大大降低了，几乎只需要一两次。 
>
>所以，Java 对于 eqauls 方法和 hashCode 方法是这样规定的： 
>
>1. 如果两个对象相等，那么它们的 hashCode 值一定要相等； 
>2. 如果两个对象的 hashCode 相等，它们并不一定相等。



### 为什么覆盖

***hashCode（）和 equals（）方法在基于哈希表的实现中存储和检索元素方面起着重要作用。 hashCode（）确定给定项映射到的存储桶。在存储桶中， equals（）方法用于查找给定的条目。***

假设我们有一个*Employee*类： 

```java
public class Employee {

    private int id;
    private String name;
    
    //constructors, getters, setters, toString implementations
}
```

还有一个存储*Employee*作为键的*HashMap* ： 

```java
Map<Employee, Integer> map = new HashMap<>();

map.put(new Employee(1, "Sam"), 1);
map.put(new Employee(2, "Sierra"), 2);
```

现在我们已经插入了两个条目，让我们尝试一个*containsKey（）*检查： 

```java
boolean containsSam = map.containsKey(new Employee(1, "Sam")); //false
```

尽管我们有*Sam*的条目，但是*containsKey（）*返回*false* 。 这是因为我们尚未覆盖*equals（）*和*hashCode（）*方法。 默认情况下， *equals（）*只会进行基于引用的比较。 

### 覆盖

根据Javadocs： 

***当我们覆盖 equals（）方法时，我们还必须覆盖 hashCode（）方法。***

这将有助于避免违反*equals-hashCode*合同。 

请注意，如果我们违反合同，编译器不会抱怨，但是当我们将此类对象作为键存储在HashMap中时，最终可能会遇到意外行为。 

我们可以使用IDE的功能快速覆盖这些方法。 使用Eclipse时，我们可以转到 Source-> Generate hashCode（）和equals（）。让我们看看为 Employee 类生成的实现： 

```java
public class Employee {

    ...

    @Override
    public int hashCode() {
        final int prime = 31;
        int result = 1;
        result = prime * result + id;
        result = prime * result + ((name == null) ? 0 : name.hashCode());
        return result;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj)
            return true;
        if (obj == null)
            return false;
        if (getClass() != obj.getClass())
            return false;
        Employee other = (Employee) obj;
        if (id != other.id)
            return false;
        if (name == null) {
            if (other.name != null)
                return false;
        } 
        else if(!name.equals(other.name))
            return false;
        return true;
    }
}
```

显然，相同的字段已用于实现 equals() 和 hashCode() 方法，以与合同保持一致。 

### 最佳做法：

 使用*equals（）*和*hashCode（）*时应遵循的一些最佳实践包括： 

-  实现*hashCode（）*以在各个存储桶之间平均分配项目。 这样做的目的是最大程度地减少碰撞次数，从而获得良好的性能 
-  对于*equals（）*和*hashCode（）*实现，我们应该使用相同的字段 
-  在*HashMap*中将不可变对象作为键，因为它们支持缓存哈希码值 
-  使用ORM工具时，请始终使用getter代替*hashCode（）*和*equals（）*方法定义中的字段。 那是因为有些字段可能是延迟加载的 

### 结论：

在本教程中，我们首先研究了*equals（）*和*hashCode（）*方法的默认实现。 稍后，我们讨论了何时以及如何覆盖这些方法。 

