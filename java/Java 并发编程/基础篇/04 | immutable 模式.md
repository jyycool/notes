# 04 | immutable 模式

## 一、定义

Immutable是“永恒的”“不会改变”的意思。在Immutable Patttern中，有着能够保证实例状态绝不会改变的类（immutable  类）。因为访问这个实例时，可以省去使用共享互斥机制所会浪费的时间，提高系统性能。java.lang.String就是一个Immutable的类。

## 二、模式案例

*案例：*
Person类，具有姓名（name）、地址（address）等字段。字段都是私有的，只能通过构造器来设置，且只有get方法，没有set方法。这时，即使有多个线程同时访问相同实例，Person类也是安全的，它的所有方法都不需要定义成synchronized。

*Person定义：*

```
public final class Person {
    private final String name;
    private final String address;
    public Person(String name, String address) {
        this.name = name;
        this.address = address;
    }
    public String getName() {
        return name;
    }
    public String getAddress() {
        return address;
    }
    public String toString() {
        return "[ Person: name = " + name + ", address = " + address + " ]";
    }
}
```

*线程定义：*

```
public class PrintPersonThread extends Thread {
    private Person person;
    public PrintPersonThread(Person person) {
        this.person = person;
    }
    public void run() {
        while (true) {
            System.out.println(Thread.currentThread().getName() + " prints " + person);
        }
    }
}
```

*执行：*

```
public class Main {
    public static void main(String[] args) {
        Person alice = new Person("Alice", "Alaska");
        new PrintPersonThread(alice).start();
        new PrintPersonThread(alice).start();
        new PrintPersonThread(alice).start();
    }
}
```

## 三、模式讲解

Immutable模式的角色如下：

- Immutable(不变的)参与者

Immutable参与者是一个字段值无法更改的类，也没有任何用来更改字段值的方法。当Immutable参与者的实例建立后，状态就完全不再变化。

![img](https://segmentfault.com/img/remote/1460000015558565)

*适用场景：*
Immutable模式的优点在于，“不需要使用synchronized保护”。而“不需要使用synchronized保护”的最大优点就是可在不丧失安全性与生命性的前提下，提高程序的执行性能。若实例由多数线程所共享，且访问非常频繁，Immutable模式就能发挥极大的优点。