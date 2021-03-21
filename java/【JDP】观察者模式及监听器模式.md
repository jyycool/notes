# 【JDP】观察者模式及监听器模式

#Lang/Java

在netty, spark, flink源码中很多都用到了监听器模式，感觉非常实用，以前看设计模式的时候只是看，并没有用上。其实这是一个非常重要并实用的设计模式，在很多框架里面都用到了。

netty里面的应用：

```java
serverBootstrap.bind(8000).addListener(new GenericFutureListener<Future<? super Void>>() {
  public void operationComplete(Future<? super Void> future) {
    if (future.isSuccess()) {
      System.out.println("端口绑定成功!");
    } else {
      System.err.println("端口绑定失败!");
    }
  }
});
```

### 1. 回调函数

为什么先提到回调函数呢？因为回调函数是理解监听器、观察者模式的关键。

[wiki中的定义](https://zh.wikipedia.org/wiki/回调函数)：

> 回调函数，或简称回调（Callback 即call then back 被主函数调用运算后会返回主函数），是指通过函数参数传递到其它代码的，某一块可执行代码的引用。这一设计允许了底层代码调用在高层定义的子程序。

通俗的定义：就是程序员A写了一段程序（程序a），其中预留有回调函数接口，并封装好了该程序。程序员B要让a调用自己的程序b中的一个方法，于是，他通过a中的接口回调自己b中的方法。

demo：这里有两个实体，回调抽象者接口、回调者（程序a）

- 回调接口（ICallBack）

```java
public interface ICallBack {
    public void callBack();
}
```

- 回调者（用于调用回调函数的类）

```java
public class Caller {
    
    public void call(ICallBack callBack) {
        System.out.println("start...");
        callBack.callBack();
        System.out.println("end...");
    }
    
}
```

- 回调测试

```java
public class CallDemo {

    public static void main(String[] args) {
        Caller caller = new Caller();

        caller.call(() -> System.out.println("回调成功"));
    }

}
```

- 控制台输出

```
start...
回调成功
end...
```

这模型和执行一个线程Thread很像. ***Thread就是回调者***，***Runnable就是一个回调接口***。

### 2. 监听器模式

> 事件监听器就是自己监听的事件一旦被触发或改变，立即得到通知，做出响应。

Java的事件监听机制可概括为3点：

1. Java的事件监听机制涉及到**事件源**，**事件监听器**，**事件对象**三个组件,监听器一般是接口，用来约定调用方式
2. **当事件源对象上发生操作时，它将调用事件监听器的一个方法，并将事件对象传递过去**
3. 事件监听器实现类，通常是由开发人员编写，开发人员通过事件对象拿到事件源，从而对事件源上的操作进行处理

这里为了方便，直接用了 jdk，EventListener 监听器

#### 监听器接口

```java
public interface EventListener extends java.util.EventListener {
    //事件处理
    public void handleEvent(EventObject event);
}
```

#### 事件对象

```java
public class EventObject extends java.util.EventObject{
    private static final long serialVersionUID = 1L;
    public EventObject(Object source){
        super(source);
    }
    public void doEvent(){
        System.out.println("通知一个事件源 source :"+ this.getSource());
    }

}
```

#### 事件源

事件源是事件对象的入口，包含监听器的注册、撤销、通知

```java
public class EventSource {
   //监听器列表，监听器的注册则加入此列表
    private Vector<EventListener> ListenerList = new Vector<EventListener>();
    //注册监听器
    public void addListener(EventListener eventListener){
        ListenerList.add(eventListener);
    }
    //撤销注册
    public void removeListener(EventListener eventListener){
        ListenerList.remove(eventListener);
    }
 //接受外部事件
    public void notifyListenerEvents(EventObject event){        
        for(EventListener eventListener:ListenerList){
                eventListener.handleEvent(event);
        }
    }
    
}
```

#### 测试执行

```java
public static void main(String[] args) {
        EventSource eventSource = new EventSource();
        
        eventSource.addListener(new EventListener(){
            @Override
            public void handleEvent(EventObject event) {
                event.doEvent();
                if(event.getSource().equals("closeWindows")){
                    System.out.println("doClose");
                } 
            }
            
        });


        /*
         * 传入openWindows事件，通知listener，事件监听器，
         对open事件感兴趣的listener将会执行
         **/
        eventSource.notifyListenerEvents(new EventObject("openWindows"));
        
}
```

控制台显示：

> 通知一个事件源 source :openWindows

> 通知一个事件源 source :openWindows

> doOpen something...

到这里你应该非常清楚的了解，什么是事件监听器模式了吧。 那么哪里是回调接口，哪里是回调者，对！EventListener是一个回调接口类，handleEvent是一个回调函数接口，通过回调模型，EventSource 事件源便可回调具体监听器动作。

### 3. 观察者模式

以《Head First 设计模式》这本书中的定义：

　**观察者模式**：它定义了对象之间的一（Subject）对多（Observer）的依赖，这样一来，当一个对象（Subject）改变时，它的所有的依赖者都会收到通知并自动更新。

![observer](/Users/sherlock/Desktop/notes/allPics/Java/observer.png)

- **主题（Subject）接口**：对象使用此接口注册为观察者，或者把自己从观察者中移除。
- **观察者（Observer）接口**：所有潜在的观察者都必须实现该接口，这个接口只有update一个方法，他就是在主题状态发生变化的时候被调用。
- **具体主题（ConcreteSubject）类**：它是要实现Subject接口，除了注册（registerObserver）和撤销（removeObserver）外，它还有一个通知所有的观察者（notifyObservers）方法。
- **具体观察者（ConcreteObserver）类**：它是要实现ObserverJ接口，并且要注册主题，以便接受更新。

#### 以微信订阅号来深入介绍观察者模式

看了上面定义及类图好像不太容易理解，微信订阅号我相信大家都不陌生，接下来就微信订阅号的例子来介绍下观察者模式。首先看下面一张图：

![wechat](/Users/sherlock/Desktop/notes/allPics/Java/wechat.png)

如上图所示，微信订阅号就是我们的主题，用户就是观察者。他们在这个过程中扮演的角色及作用分别是：

1. 订阅号就是主题，业务就是推送消息
2. 观察者想要接受推送消息，只需要订阅该主题即可
3. 当不再需要消息推送时，取消订阅号关注即可
4. 只要订阅号还在，观察者可以一直去进行关注

接下来让我们通过一段示例，三位同事 tom、lucy、jack 订阅人民日报订阅号为例来介绍观察者模式（Obsever），代码如下：

```java
package com.github.jdp.observer;

/**
 * Descriptor: 观察者模式中的subject接口
 *             本例中是订阅者订阅的报纸的接口类
 * Author: sherlock
 */
public interface NewsPaper<T> {

    // 添加订阅的方法
    T addObserver(Observer observer);

    // 取消订阅的方法
    T removeObserver(Observer observer);

    // 当有更新时, 通知所有Observer
    void notifyObservers();
}
```

```java
package com.github.jdp.observer;

/**
 * Descriptor: 观察者Observer接口
 *             本例中则是报纸的订阅者的接口
 * Author: sherlock
 */
public interface Observer {

    // 当有更新时, 告知Observer的方法
    void update();
}
```

```java
package com.github.jdp.observer;

import java.util.ArrayList;
import java.util.List;

/**
 * Descriptor: NewsPaper的实现类, 人民日报
 * Author: sherlock
 */
public class PeopleDaily implements NewsPaper<PeopleDaily> {

    private List<Observer> observers = new ArrayList<>();

    @Override
    public PeopleDaily addObserver(Observer observer) {
        observers.add(observer);
        return this;
    }

    @Override
    public PeopleDaily removeObserver(Observer observer) {
        observers.remove(observer);
        return this;
    }

    // 更新时,回调Observer的update方法来通知Observer
    @Override
    public void notifyObservers() {
        if (observers.size() > 0) {
            observers.stream().forEach(Observer::update);
        }
    }

}
```

```java
package com.github.jdp.observer;

import lombok.AllArgsConstructor;
import lombok.Getter;

/**
 * Descriptor: 人民日报的订阅者类
 * Author: sherlock
 */
@AllArgsConstructor
@Getter
public class Subscriber implements Observer {

    private String name;
    private String newsPaper;

    @Override
    public void update() {
        System.out.printf("用户[%s]订阅的[%s]更新啦......\n", this.getName(), this.getNewsPaper());
    }
}
```

```java
package com.github.jdp.observer;

/**
 * Descriptor: 测试
 * Author: sherlock
 */
public class Main {

    public static void main(String[] args) {

        Subscriber tom = new Subscriber("tom", "人民日报");
        Subscriber jack = new Subscriber("jack", "人民日报");
        Subscriber lucy = new Subscriber("lucy", "人民日报");

        PeopleDaily peopleDaily = new PeopleDaily();
        peopleDaily.addObserver(tom)
                .addObserver(jack)
                .addObserver(lucy);

        // lucy取消了订阅
        peopleDaily.removeObserver(lucy);

        // 有更新通知
        peopleDaily.notifyObservers();
    }
}
```

> 测试结果:
>
> 用户[tom]订阅的[人民日报]更新啦......
> 用户[jack]订阅的[人民日报]更新啦......

#### 再来说设计模式的推拉模型

在观察者模式中，又分为推模型和拉模型两种方式。

- **推模型**：主题对象向观察者推送主题的详细信息，不管观察者是否需要，每次有新的信息就会推送给它的所有的观察者。
- **拉模型**：主题对象是根据观察者需要更具体的信息，由观察者主动到主题对象中获取，相当于是观察者从主题对象中拉数据。

而它们的区别在于：

“推”的好处包括：
1、高效。如果没有更新发生，不会有任何更新消息推送的动作，即每次消息推送都发生在确确实实的更新事件之后，所以这种推送是有意义的。
2、实时。事件发生后的第一时间即可触发通知操作。
“拉”的好处包括：
1、如果观察者众多，那么主题要维护订阅者的列表臃肿，把订阅关系解脱到Observer去完成，什么时候要自己去拉数据就好了。
2、Observer可以不理会它不关心的变更事件，只需要去获取自己感兴趣的事件即可。

根据上面的描述，发现前面的例子就是典型的推模型，下面我先来介绍下java内置的拉模型设计模式实现，再给出一个拉模型的实例。

在JAVA编程语言的java.util类库里面，提供了一个Observable类以及一个Observer接口，用来实现JAVA语言对观察者模式的支持。
　　Observer接口：这个接口代表了观察者对象，它只定义了一个方法，即update()方法，每个观察者都要实现这个接口。当主题对象的状态发生变化时，主题对象的notifyObservers()方法就会调用这一方法。

```java
public interface Observer {
    void update(Observable o, Object arg);
}
```

Observable类：这个类代表了主题对象，主题对象可以有多个观察者，主题对象发生变化时，会调用Observable的notifyObservers()方法，此方法调用所有的具体观察者的update()方法，从而使所有的观察者都被通知更新自己

```java
package java.util;

public class Observable {
    private boolean changed = false;
    private Vector obs;


    public Observable() {
        obs = new Vector<>();
    }
    
   //添加一个观察者
    public synchronized void addObserver(Observer o) {
        if (o == null)
            throw new NullPointerException();
        if (!obs.contains(o)) {
            obs.addElement(o);
        }
    }

   //删除一个观察者
    public synchronized void deleteObserver(Observer o) {
        obs.removeElement(o);
    }

    public void notifyObservers() {
        notifyObservers(null);
    }

   //通知所有的观察者
    public void notifyObservers(Object arg) {

        Object[] arrLocal;

        synchronized (this) {
          
            if (!changed)
                return;
            arrLocal = obs.toArray();
            clearChanged();
        }

        //调用Observer类通知所有的观察者
        for (int i = arrLocal.length-1; i>=0; i--)
            ((Observer)arrLocal[i]).update(this, arg);
    }
    
    public synchronized void deleteObservers() {
        obs.removeAllElements();
    }

    protected synchronized void setChanged() {
        changed = true;
    }

    protected synchronized void clearChanged() {
        changed = false;
    }

    //省略......
}
```

接下来再介绍我用java这种内置的观察者设计模式在项目中的一个实际应用，详细请看我的这篇博文：[观察者模式实际应用：监听线程，意外退出线程后自动重启](http://www.cnblogs.com/zishengY/p/7056948.html)

**项目场景**：用户那边会不定期的上传文件到一个ftp目录，我需要实现新上传的文件做一个自动检测，每次只要有文件新增，自动解析新增文件内容入库，并且要保证该功能的稳定性！！

**实现思路**：

1、监听器初始化创建：首先在tomcat启动的时候，利用监听器初始化创建一个监控文件新增线程，如下：　　

```java
@Component
public class ThreadStartUpListenser implements ServletContextListener
{
    //监控文件新增线程
    private static WatchFilePathTask r = new WatchFilePathTask();

    private Log log = LogFactory.getLog(ThreadStartUpListenser.class);

    @Override
    public void contextDestroyed(ServletContextEvent paramServletContextEvent)
    {
        // r.interrupt();

    }

    @Override
    public void contextInitialized(ServletContextEvent paramServletContextEvent)
    {
        //将监控文件类添加为一个观察者，并启动一个线程
        ObserverListener listen = new ObserverListener();
        r.addObserver(listen);
        new Thread(r).start();
        // r.start();
        log.info("ImportUserFromFileTask is started!");
    }

}
```

2、**主体对象**：即下面的监控文件新增类WatchFilePathTask ，每次有新文件进来，自动解析该文件，挂掉之后，调用动doBusiness()里面的notifyObservers()方法，伪代码如下：

```java
//继承java内置观察者模式实现的Observable 类
public class WatchFilePathTask extends Observable implements Runnable
{
    private Log log = LogFactory.getLog(WatchFilePathTask.class);

    private static final String FILE_PATH = ConfigUtils.getInstance()
            .getValue("userfile_path");

    private WatchService watchService;

    /**
     * 此方法一经调用，立马可以通知观察者，在本例中是监听线程 
     */
    public void doBusiness()
    {
        if (true)
        {
            super.setChanged();
        }
        notifyObservers();
    }

    @Override
    public void run()
    {
        try
        {
            //这里省略监控新增文件的方法
        }catch (Exception e)
        {
            e.printStackTrace();
            
            doBusiness();// 在抛出异常时调用，通知观察者，让其重启线程
        }
    }
}
```

3、**观察者对象：**即上面出现的**ObserverListener**类，当主题对象的的notifyObservers()方法被调用的时候，就会调用该类的update()方法，伪代码如下：

```java
//实现java内置观察者模式实现的Observer接口，并且注册主题WatchFilePathTask,以便线程挂掉的时候，再重启这个线程
public class ObserverListener implements Observer
{
    private Log log = LogFactory.getLog(ObserverListener.class);
     
    /**
     * @param o 
     * @param arg 
     */
    public void update(Observable o, Object arg)
    {
        log.info("WatchFilePathTask挂掉");
        WatchFilePathTask run = new WatchFilePathTask();
        run.addObserver(this);
        new Thread(run).start();
        log.info("WatchFilePathTask重启");
    }
}
```