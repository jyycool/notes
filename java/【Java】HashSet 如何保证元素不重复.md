# 【Java】HashSet 如何保证元素不重复

首先先搞清楚 `hashcode()` 和 `equlas()` 方法的主要作用和复写原则

弄清怎么个逻辑达到元素不重复的，源码先上 `HashSet` 类中的 `add()` 方法：

```java
public boolean add(E e) {
  return map.put(e, PRESENT)==null;
}
```

类中map和PARENT的定义：

```java
private transient HashMap<E,Object> map;
// Map用来匹配Map中后面的对象的一个虚拟值
private static final Object PRESENT = new Object();
```

`put()` 方法：

```java
public V put(K key, V value) {
  if (key == null)
    return putForNullKey(value);
  int hash = hash(key.hashCode());
  int i = indexFor(hash, table.length);
  for (Entry<K,V> e = table[i]; e != null; e = e.next) {
    Object k;
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
      V oldValue = e.value;
      e.value = value;
      e.recordAccess(this);
      return oldValue;
    }
  }
  modCount++;
  addEntry(hash, key, value, i);
  return null;
}
```

可以看到for循环中，遍历table中的元素， 

1. 如果hash码值不相同，说明是一个新元素，存；

2. 如果没有元素和传入对象（也就是add的元素）的hash值相等，那么就认为这个元素在table中不存在，将其添加进table；

3. 如果hash码值相同，且equles判断相等，说明元素已经存在，不存；

   如果hash码值相同，且equles判断不相等，说明元素不存在，存；

如果有元素和传入对象的hash值相等，那么，继续进行equles()判断，如果仍然相等，那么就认为传入元素已经存在，不再添加，结束，否则仍然添加；

