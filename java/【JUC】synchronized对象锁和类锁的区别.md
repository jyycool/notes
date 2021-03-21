# 【JUC】synchronized对象锁和类锁的区别

#Lang/Java

### 一、synchronized关键字

synchronized关键字有如下两种用法：

1. 在需要同步的方法的方法签名中加入synchronized关键字。

   ```java
   synchronized public void getValue() {
   	System.out.println("getValue method thread name=" + Thread.currentThread().getName() + ",username=" + username + " password=" + password);
   }
   ```

   > 上面的代码修饰的synchronized是非静态方法，如果修饰的是静态方法（static）含义是完全不一样的。具体不一样在哪里，后面会详细说清楚。

   ```java
   synchronized static public void getValue() {
   	System.out.println("getValue method thread name=" + Thread.currentThread().getName() + " username=" + username + " password=" + password);
   }
   ```

   

2. 使用synchronized块对需要进行同步的代码段进行同步。

   ```java
   public void serviceMethod() {
   	try {
   		synchronized (this) {
   			System.out.println("begin time=" + System.currentTimeMillis());
   			Thread.sleep(2000);
   			System.out.println("end=" + System.currentTimeMillis());
   		}
   	} catch (InterruptedException e) {
   		e.printStackTrace();
   	}
   }
   ```

   > 上面的代码块是synchronized (this)用法，还有synchronized (非this对象)以及synchronized (类.class)这两种用法，这些使用方式的含义也是有根本的区别的。我们先带着这些问题继续往下看。



### 二、Java中的对象锁和类锁

***上面提到锁，这里先引出锁的概念:***

> 多线程的线程同步机制实际上是靠锁的概念来控制的。
>
> 在Java程序运行时环境中，JVM需要对两类线程共享的数据进行协调：
>
> 1. 保存在堆中的实例变量
> 2. 保存在方法区中的类变量
>
> 这两类数据是被所有线程共享的。(程序不需要协调保存在Java栈当中的数据。因为这些数据是属于拥有该栈的线程所私有的）

***JVM内存模型:(粗略说明)***

- **方法区**

  > 与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开来。

- **栈**

  > 在Java中，JVM中的栈记录了线程的方法调用。每个线程拥有一个栈。在某个线程的运行过程中，如果有新的方法调用，那么该线程对应的栈就会增加一个存储单元，即帧(frame)。在frame中，保存有该方法调用的参数、局部变量和返回地址。

- **堆**

  > JVM中一块可自由分配给对象的区域。***当我们谈论垃圾回收(garbage collection)时，我们主要回收堆(heap)的空间。*** Java的普通对象存活在堆中。与栈不同，堆的空间不会随着方法调用结束而清空。因此，在某个方法中创建的对象，可以在方法调用结束之后，继续存在于堆中。这带来的一个问题是，如果我们不断的创建新的对象，内存空间将最终消耗殆尽。(对象如果创建在方法内是可以被回收的, 详见JDK1.8的方法逃逸)

在java虚拟机中，每个对象和类在逻辑上都是和一个监视器相关联的。 对于对象来说，相关联的监视器保护对象的实例变量。

对于类来说，监视器保护类的类变量。

（如果一个对象没有实例变量，或者一个类没有变量，相关联的监视器就什么也不监视。） 为了实现监视器的排他性监视能力，java虚拟机为每一个对象和类都关联一个锁。代表任何时候只允许一个线程拥有的特权。线程访问实例变量或者类变量不需锁。

但是如果线程获取了锁，那么在它释放这个锁之前，就没有其他线程可以获取同样数据的锁了。（锁住一个对象就是获取对象相关联的监视器）

类锁实际上用对象锁来实现。当虚拟机装载一个class文件的时候，它就会创建一个java.lang.Class类的实例。当锁住一个对象的时候，实际上锁住的是那个类的Class对象。

一个线程可以多次对同一个对象上锁。对于每一个对象，java虚拟机维护一个加锁计数器，线程每获得一次该对象，计数器就加1，每释放一次，计数器就减 1，当计数器值为0时，锁就被完全释放了。

java编程人员不需要自己动手加锁，对象锁是java虚拟机内部使用的。

在java程序中，只需要使用synchronized块或者synchronized方法就可以标志一个监视区域。当每次进入一个监视区域时，java 虚拟机都会自动锁上对象或者类。

> 上述总结:
>
> 1. 锁的实质就是获取类或对象的监视器
> 2. 类锁实质上还是对象锁, 只不过这个对象是`Class<T> clazz`



### 三、synchronized用法与实例

- **synchronized、synchronized (this)用法锁的是对象**

- **sychronized static、synchronized (xxx.class)的用法锁的是类**

**本文的实例均来自于《Java多线程编程核心技术》这本书里面的例子。**

#### 1、非线程安全实例

```java
public class Run {
 
    public static void main(String[] args) {
 
        HasSelfPrivateNum numRef = new HasSelfPrivateNum();
 
        ThreadA athread = new ThreadA(numRef);
        athread.start();
 
        ThreadB bthread = new ThreadB(numRef);
        bthread.start();
 
    }
 
}
 
class HasSelfPrivateNum {
 
    private int num = 0;
 
    public void addI(String username) {
        try {
            if (username.equals("a")) {
                num = 100;
                System.out.println("a set over!");
                Thread.sleep(2000);
            } else {
                num = 200;
                System.out.println("b set over!");
            }
            System.out.println(username + " num=" + num);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
 
}
 
class ThreadA extends Thread {
 
    private HasSelfPrivateNum numRef;
 
    public ThreadA(HasSelfPrivateNum numRef) {
        super();
        this.numRef = numRef;
    }
 
    @Override
    public void run() {
        super.run();
        numRef.addI("a");
    }
 
}
 
 class ThreadB extends Thread {
 
    private HasSelfPrivateNum numRef;
 
    public ThreadB(HasSelfPrivateNum numRef) {
        super();
        this.numRef = numRef;
    }
 
    @Override
    public void run() {
        super.run();
        numRef.addI("b");
    }
 
}
```

运行结果:

```java
a set over!
b set over!
b num=200
a num=200
```

修改HasSelfPrivateNum如下，方法用synchronized修饰如下：

```java
class HasSelfPrivateNum {
 
    private int num = 0;
 
    synchronized public void addI(String username) {
        try {
            if (username.equals("a")) {
                num = 100;
                System.out.println("a set over!");
                Thread.sleep(2000);
            } else {
                num = 200;
                System.out.println("b set over!");
            }
            System.out.println(username + " num=" + num);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
 
}
```

运行结果是线程安全的：

```javascript
b set over!
b num=200
a set over!
a num=100
```

> **实验结论：两个线程访问同一个对象中的同步方法是一定是线程安全的。本实现由于是同步访问，所以先打印出a，然后打印出b**
>
> **这里线程获取的是HasSelfPrivateNum的对象实例的锁——对象锁。**

#### 2、多个对象多个锁

就上面的实例，我们将Run改成如下：

```javascript
public class Run {
 
    public static void main(String[] args) {
 
        HasSelfPrivateNum numRef1 = new HasSelfPrivateNum();
        HasSelfPrivateNum numRef2 = new HasSelfPrivateNum();
 
        ThreadA athread = new ThreadA(numRef1);
        athread.start();
 
        ThreadB bthread = new ThreadB(numRef2);
        bthread.start();
 
    }
 
}
```

```javascript
a set over!
b set over!
b num=200
a num=200
```

> **这里是非同步的，因为线程athread获得是numRef1的对象锁，而bthread线程获取的是numRef2的对象锁，他们并没有在获取锁上有竞争关系，因此，出现非同步的结果**
>
> **这里插播一下：同步不具有继承性**

#### 3、synchronized(this)

**代码实例（Run.java）**

```javascript
public class Run {
 
    public static void main(String[] args) {
        ObjectService service = new ObjectService();
 
        ThreadA a = new ThreadA(service);
        a.setName("a");
        a.start();
 
        ThreadB b = new ThreadB(service);
        b.setName("b");
        b.start();
    }
 
}
 
class ObjectService {
 
    public void serviceMethod() {
        try {
            synchronized (this) {
                System.out.println("begin time=" + System.currentTimeMillis());
                Thread.sleep(2000);
                System.out.println("end    end=" + System.currentTimeMillis());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
 
class ThreadA extends Thread {
 
    private ObjectService service;
 
    public ThreadA(ObjectService service) {
        super();
        this.service = service;
    }
 
    @Override
    public void run() {
        super.run();
        service.serviceMethod();
    }
 
}
 
class ThreadB extends Thread {
    private ObjectService service;
 
    public ThreadB(ObjectService service) {
        super();
        this.service = service;
    }
 
    @Override
    public void run() {
        super.run();
        service.serviceMethod();
    }
}
```

运行结果：

```javascript
begin time=1466148260341
end    end=1466148262342
begin time=1466148262342
end    end=1466148264378
```

> **这样也是同步的，线程获取的是同步块synchronized (this)括号（）里面的对象实例的对象锁，这里就是ObjectService实例对象的对象锁了。**
>
> **需要注意的是`synchronized (){}`的{}前后的代码依旧是异步的**

#### 4、synchronized(非this对象)

**代码实例(Run.java)**

```javascript
public class Run {
 
    public static void main(String[] args) {
 
        Service service = new Service("xiaobaoge");
 
        ThreadA a = new ThreadA(service);
        a.setName("A");
        a.start();
 
        ThreadB b = new ThreadB(service);
        b.setName("B");
        b.start();
 
    }
 
}
 
class Service {
 
    String anyString = new String();
 
    public Service(String anyString){
        this.anyString = anyString;
    }
 
    public void setUsernamePassword(String username, String password) {
        try {
            synchronized (anyString) {
                System.out.println("线程名称为：" + Thread.currentThread().getName()
                        + "在" + System.currentTimeMillis() + "进入同步块");
                Thread.sleep(3000);
                System.out.println("线程名称为：" + Thread.currentThread().getName()
                        + "在" + System.currentTimeMillis() + "离开同步块");
            }
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
 
}
 
class ThreadA extends Thread {
    private Service service;
 
    public ThreadA(Service service) {
        super();
        this.service = service;
    }
 
    @Override
    public void run() {
        service.setUsernamePassword("a", "aa");
 
    }
 
}
 
class ThreadB extends Thread {
 
    private Service service;
 
    public ThreadB(Service service) {
        super();
        this.service = service;
    }
 
    @Override
    public void run() {
        service.setUsernamePassword("b", "bb");
 
    }
 
}
```

> **不难看出，这里线程争夺的是anyString的对象锁，两个线程有竞争同一对象锁的关系，出现同步**
>
> **Q：一个类里面有两个非静态同步方法，会有影响么？**
>
> **A：如果对象实例M，线程1获得了对象M的对象锁，那么其他线程就不能进入需要获得对象实例M的对象锁才能访问的同步代码（包括同步方法和同步块）。**

#### 5、静态synchronized

**代码实例：**

```java
public class Run {
 
    public static void main(String[] args) {
 
        ThreadA a = new ThreadA();
        a.setName("A");
        a.start();
 
        ThreadB b = new ThreadB();
        b.setName("B");
        b.start();
 
    }
 
}
 
class Service {
 
    synchronized public static void printA() {
        try {
            System.out.println("线程名称为：" + Thread.currentThread().getName()
                    + "在" + System.currentTimeMillis() + "进入printA");
            Thread.sleep(3000);
            System.out.println("线程名称为：" + Thread.currentThread().getName()
                    + "在" + System.currentTimeMillis() + "离开printA");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
 
    synchronized public static void printB() {
        System.out.println("线程名称为：" + Thread.currentThread().getName() + "在"
                + System.currentTimeMillis() + "进入printB");
        System.out.println("线程名称为：" + Thread.currentThread().getName() + "在"
                + System.currentTimeMillis() + "离开printB");
    }
 
}
 
class ThreadA extends Thread {
    @Override
    public void run() {
        Service.printA();
    }
 
}
 
class ThreadB extends Thread {
    @Override
    public void run() {
        Service.printB();
    }
}
```

运行结果：

```javascript
线程名称为：A在1466149372909进入printA
线程名称为：A在1466149375920离开printA
线程名称为：B在1466149375920进入printB
线程名称为：B在1466149375920离开printB
```

#### 6、synchronized (class)

**对上面Service类代码修改成如下：**

```java
class Service {
 
    public static void printA() {
        synchronized (Service.class) {
            try {
                System.out.println("线程名称为：" + Thread.currentThread().getName()
                        + "在" + System.currentTimeMillis() + "进入printA");
                Thread.sleep(3000);
                System.out.println("线程名称为：" + Thread.currentThread().getName()
                        + "在" + System.currentTimeMillis() + "离开printA");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
 
    }
 
    public static void printB() {
        synchronized (Service.class) {
            System.out.println("线程名称为：" + Thread.currentThread().getName()
                    + "在" + System.currentTimeMillis() + "进入printB");
            System.out.println("线程名称为：" + Thread.currentThread().getName()
                    + "在" + System.currentTimeMillis() + "离开printB");
        }
    }
}
```

运行结果：

```javascript
线程名称为：A在1466149372909进入printA
线程名称为：A在1466149375920离开printA
线程名称为：B在1466149375920进入printB
线程名称为：B在1466149375920离开printB
```

> **两个线程依旧在争夺同一个类锁，因此同步**
>
> **需要特别说明：对于同一个类A，线程1争夺A对象实例的对象锁，线程2争夺类A的类锁，这两者不存在竞争关系。也就说对象锁和类锁互补干预内政**
>
> **静态方法则一定会同步，非静态方法需在单例模式才生效，但是也不能都用静态同步方法，总之用得不好可能会给性能带来极大的影响。另外，有必要说一下的是Spring的bean默认是单例的。**