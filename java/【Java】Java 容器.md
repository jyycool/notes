# 【Java】Java 容器

#Lang/Java

![java集合](/Users/sherlock/Desktop/notes/allPics/Java/java集合.png)

上图为整理的集合类图关系，带✅标志的为线程安全类。



### 一、基本概念

Java容器类类库的用途是“保存对象”，并将其划分为两个不同的概念：

1. ***Collection***

   一个独立元素的序列，这些元素都服从一条或多条规则。List必须按照插入的顺序保存元素，而Set不能有重复元素。Queue按照排队规则来确定对象产生的顺序（通常与它们被插入的顺序相同）。

2. ***Map***

   一组成对的“键值对”对象(Entry)，允许你使用键来查找值。ArrayList允许你使用***数字(索引)***来查找值，因此在某种意义上讲，它将***数字(索引)***与对象关联在了一起。映射表允许我们使用另一个对象来查找某个对象，它也被称为“关联数组”，因为它将某些对象与另外一些对象关联在了一起；或者被称为“字典”，因为你可以使用键对象来查找值对象，就像在字典中使用单词来定义一样。Map是强大的编程工具。

在大部分情况下, Java 鼓励向上转型, 像如下那样创建一个 List:

```java
List<String> list = new ArrayList<String>();
```

但是这种方式并非总能奏效, 因为某些类具有额外的功能，例如，LinkedList 具有在 List 接口中未包含的额外方法，而 TreeMap 也具有在 Map 接口中未包含的方法。如果你需要使用这些方法，就不能将它们向上转型为更通用的接口。

Java容器类库中的***两种主要类型 (Collection 和 Map)***，它们的***区别在于容器中每个“槽”保存的元素个数***。

***Collection*** 在每个槽中只能保存一个元素。此类容器包括：

1. ***List***，它以特定的顺序保存一组元素；
2. ***Set***，元素不能重复；
3. ***Queue***，只允许在容器的一“端”插入对象，并从另外一“端”移除对象（对于本例来说，这只是另外一种观察序列的方式，因此并没有展示它）。

***Map*** 在每个槽内保存了两个对象，即键和与之相关联的值。



#### 1.1 Collection

```java
public class PrintingContainers {

    static Collection fill(Collection<String> collection) {
        collection.add("rat");
        collection.add("cat");
        collection.add("dog");
        collection.add("dog");
        return collection;
    }

    static Map fill(Map<String, String> map) {
        map.put("rat", "Fuzzy");
        map.put("cat", "Rags");
        map.put("dog", "Bosco");
        map.put("dog", "Bosco");
        return map;
    }

    public static void main(String[] args) {

        out.println("ArrayList: " + fill(new ArrayList<String>()));
        out.println("LinkedList: " + fill(new LinkedList<>()));
        out.println("HashSet: " + fill(new HashSet<>()));
        out.println("TreeSet: " + fill(new TreeSet<>()));
        out.println("LinkedHashSet: " + fill(new LinkedHashSet<>()));
        out.println("HashMap: " + fill(new HashMap<>()));
        out.println("TreeMap: " + fill(new TreeMap<>()));
        out.println("LinkedHashMap: " + fill(new LinkedHashMap<>()));
    }
  	
  	/*
  	  ArrayList: [rat, cat, dog, dog]
			LinkedList: [rat, cat, dog, dog]
			HashSet: [rat, cat, dog]
			TreeSet: [cat, dog, rat]
			LinkedHashSet: [rat, cat, dog]
			HashMap: {rat=Fuzzy, cat=Rags, dog=Bosco}
			TreeMap: {cat=Rags, dog=Bosco, rat=Fuzzy}
		  LinkedHashMap: {rat=Fuzzy, cat=Rags, dog=Bosco}
  	 */
}
```

##### 1.1.1 List

***ArrayList*** 和 ***LinkedList*** 都是 List 类型, 从输出可以看出, 他们都是按照被插入顺序保存元素. 两者不同之处不仅在于执行时某些类型操作的性能, 而且 LinkedList 包含的操作也多与 ArrayList.



##### 1.1.2 Set

***HashSet***、***TreeSet***和 ***LinkedHashSet*** 都是Set类型，每个相同的项只有保存一次，但是输出也显示了不同的Set实现存储元素的方式也不同。HashSet使用的是相当复杂的方式来存储元素的，这种方式将在第17章中介绍，此刻你只需要知道这种技术是最快的获取元素方式，因此，存储的顺序看起来并无实际意义（通常你只会关心某事物是否是某个Set的成员，而不会关心它在Set出现的顺序）。***如果存储顺序很重要，那么可以使用 TreeSet，它按照比较结果的升序保存对象；******或者使用LinkedHashSet，它按照被添加的顺序保存对象。***

#### 1.2 Map

Map（也被称为关联数组）使得你可以用键来查找对象，就像一个简单的数据库。键所关联的对象称为值。使用Map可以将美国州名与其首府联系起来，如果想知道Ohio的首府，可以将Ohio作为键进行查找，几乎就像使用数组下标一样。正由于这种行为，对于每一个键，Map只接受存储一次。

Map.put（key, value）方法将增加一个值（你想要增加的对象），并将它与某个键（你用来查找这个值的对象）关联起来。Map.get（key）方法将产生与这个键相关联的值。上面的示例只添加了键-值对，并没有执行查找，这将在稍后展示。

注意，你不必指定（或考虑）Map的尺寸，因为它自己会自动地调整尺寸。Map还知道如何打印自己，它会显示相关联的键和值。键和值在Map中的保存顺序并不是它们的插入顺序，因为HashMap实现使用的是一种非常快的算法来控制顺序。

本例使用了三种基本风格的Map：***HashMap***、***TreeMap*** 和 ***LinkedHashMap***。与HashSet一样，HashMap也提供了最快的查找技术，也没有按照任何明显的顺序来保存其元素。***TreeMap按照比较结果的升序保存键，而LinkedHashMap则按照插入顺序保存键，同时还保留了HashMap的查询速度。***



### 二、List

List承诺可以将元素维护在特定的序列中。

有两种类型的List：

- ***ArrayList***，它长于随机访问元素，但是在List的中间插入和移除元素时较慢。

- ***LinkedList***，它通过代价较低的在List中间进行的插入和删除操作，提供了优化的顺序访问。LinkedList在随机访问方面相对比较慢，但是它的特性集较ArrayList更大。



