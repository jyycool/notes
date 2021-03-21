# 【Mockito】使用指南



## mock和Mockito的关系

在软件开发中提及”mock”，通常理解为模拟对象。

为什么需要模拟? 在我们一开始学编程时,我们所写的对象通常都是独立的，并不依赖其他的类，也不会操作别的类。但实际上，软件中是充满依赖关系的，比如我们会基于service类写操作类,而service类又是基于数据访问类(DAO)的，依次下去，形成复杂的依赖关系。

单元测试的思路就是我们想在不涉及依赖关系的情况下测试代码。这种测试可以让你无视代码的依赖关系去测试代码的有效性。核心思想就是如果代码按设计正常工作，并且依赖关系也正常，那么他们应该会同时工作正常。

有些时候，我们代码所需要的依赖可能尚未开发完成，甚至还不存在，那如何让我们的开发进行下去呢？使用mock可以让开发进行下去，**mock技术的目的和作用就是模拟一些在应用中不容易构造或者比较复杂的对象，从而把测试与测试边界以外的对象隔离开**。

我们可以自己编写自定义的Mock对象实现mock技术，但是编写自定义的Mock对象需要额外的编码工作，同时也可能引入错误。现在实现mock技术的优秀开源框架有很多，**Mockito就是一个优秀的用于单元测试的mock框架**。Mockito已经在github上开源，详细请点击：https://github.com/mockito/mockito

除了Mockito以外，还有一些类似的框架，比如：

- EasyMock：早期比较流行的MocK测试框架。它提供对接口的模拟，能够通过录制、回放、检查三步来完成大体的测试过程，可以验证方法的调用种类、次数、顺序，可以令 Mock 对象返回指定的值或抛出指定异常
- PowerMock：这个工具是在EasyMock和Mockito上扩展出来的，目的是为了解决EasyMock和Mockito不能解决的问题，比如对static, final, private方法均不能mock。其实测试架构设计良好的代码，一般并不需要这些功能，但如果是在已有项目上增加单元测试，老代码有问题且不能改时，就不得不使用这些功能了
- JMockit：JMockit 是一个轻量级的mock框架是用以帮助开发人员编写测试程序的一组工具和API，该项目完全基于 Java 5 SE 的 java.lang.instrument 包开发，内部使用 ASM 库来修改Java的Bytecode

Mockito已经被广泛应用，所以这里重点介绍Mockito。

## Mockito使用举例

这里我们直接通过一个代码来说明mockito对单元测试的帮助，代码有三个类，分别如下： 
Person类：

```java
public class Person {

    private final int    id;
    private final String name;

    public Person(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }
}123456789101112131415161718
```

PersonDAO：

```java
public interface PersonDao {

    Person getPerson(int id);

    boolean update(Person person);
}123456
```

PersonService：

```java
public class PersonService {

    private final PersonDao personDao;

    public PersonService(PersonDao personDao) {
        this.personDao = personDao;
    }

    public boolean update(int id, String name) {
        Person person = personDao.getPerson(id);
        if (person == null) {
            return false;
        }

        Person personUpdate = new Person(person.getId(), name);
        return personDao.update(personUpdate);
    }
}123456789101112131415161718
```

在这里，我们要**进行测试的是PersonService类的update方法**，我们发现，update方法依赖PersonDAO，在开发过程中，**PersonDAO很可能尚未开发完成**，所以我们测试PersonService的时候，所以该怎么测试update方法呢？连接口都还没实现，怎么知道返回的是true还是false？在这里，我们可以这样认为，单元测试的思路就是我们想在不涉及依赖关系的情况下测试代码。这种测试可以让你无视代码的依赖关系去测试代码的有效性。核心思想就是如果代码按设计正常工作，并且依赖关系也正常，那么他们应该会同时工作正常。所以我们的做法是mock一个PersonDAO对象，至于实际环境中，PersonDAO行为是否能按照预期执行，比如update是否能成功，查询是否返回正确的数据，就跟PersonService没关系了。PersonService的单元测试只测试自己的逻辑是否有问题

下面编写测试代码：

```java
public class PersonServiceTest {

    private PersonDao     mockDao;
    private PersonService personService;

    @Before
    public void setUp() throws Exception {
        //模拟PersonDao对象
        mockDao = mock(PersonDao.class);
        when(mockDao.getPerson(1)).thenReturn(new Person(1, "Person1"));
        when(mockDao.update(isA(Person.class))).thenReturn(true);

        personService = new PersonService(mockDao);
    }

    @Test
    public void testUpdate() throws Exception {
        boolean result = personService.update(1, "new name");
        assertTrue("must true", result);
        //验证是否执行过一次getPerson(1)
        verify(mockDao, times(1)).getPerson(eq(1));
        //验证是否执行过一次update
        verify(mockDao, times(1)).update(isA(Person.class));
    }

    @Test
    public void testUpdateNotFind() throws Exception {
        boolean result = personService.update(2, "new name");
        assertFalse("must true", result);
        //验证是否执行过一次getPerson(1)
        verify(mockDao, times(1)).getPerson(eq(1));
        //验证是否执行过一次update
        verify(mockDao, never()).update(isA(Person.class));
    }
}1234567891011121314151617181920212223242526272829303132333435
```

我们对PersonDAO进行mock，并且设置stubbing，stubbing设置如下：

- 当getPerson方法传入1的时候，返回一个Person对象，否则默认返回空
- 当调update方法的时候，返回true

我们验证了两种情况：

- 更新id为1的Person的名字，预期：能在DAO中找到Person并更新成功
- 更新id为2的Person的名字，预期：不能在DAO中找到Person，更新失败

这样，根据PersonService的update方法的逻辑，通过这两个test case之后，我们认为代码是没有问题的。mockito在这里扮演了一个为我们模拟DAO对象，并且帮助我们验证行为（比如验证是否调用了getPerson方法及update方法）的角色

## Mockito使用方法

Mockito的使用，有详细的api文档，具体可以查看：http://site.mockito.org/mockito/docs/current/org/mockito/Mockito.html，下面是整理的一些常用的使用方式。

### 验证行为

> 一旦创建，mock会记录所有交互，你可以验证所有你想要验证的东西

```java
@Test
public void testVerify() throws Exception {
    //mock creation
    List mockedList = mock(List.class);

    //using mock object
    mockedList.add("one");
    mockedList.add("two");
    mockedList.add("two");
    mockedList.clear();

    //verification
    verify(mockedList).add("one");//验证是否调用过一次 mockedList.add("one")方法，若不是（0次或者大于一次），测试将不通过
    verify(mockedList, times(2)).add("two");
    //验证调用过2次 mockedList.add("two")方法，若不是，测试将不通过
    verify(mockedList).clear();//验证是否调用过一次 mockedList.clear()方法，若没有（0次或者大于一次），测试将不通过
}1234567891011121314151617
```

### Stubbing

```java
@Test
public void testStubbing() throws Exception {
    //你可以mock具体的类，而不仅仅是接口
    LinkedList mockedList = mock(LinkedList.class);

    //设置桩
    when(mockedList.get(0)).thenReturn("first");
    when(mockedList.get(1)).thenThrow(new RuntimeException());

    //打印 "first"
    System.out.println(mockedList.get(0));

    //这里会抛runtime exception
    System.out.println(mockedList.get(1));

    //这里会打印 "null" 因为 get(999) 没有设置
    System.out.println(mockedList.get(999));

    //Although it is possible to verify a stubbed invocation, usually it's just redundant
    //If your code cares what get(0) returns, then something else breaks (often even before verify() gets executed).
    //If your code doesn't care what get(0) returns, then it should not be stubbed. Not convinced? See here.
    verify(mockedList).get(0);
}1234567891011121314151617181920212223
```

对于stubbing，有以下几点需要注意：

- 对于有返回值的方法，mock会默认返回null、空集合、默认值。比如，为int/Integer返回0，为boolean/Boolean返回false
- stubbing可以被覆盖，但是请注意覆盖已有的stubbing有可能不是很好
- 一旦stubbing，不管调用多少次，方法都会永远返回stubbing的值
- 当你对同一个方法进行多次stubbing，最后一次stubbing是最重要的

### 参数匹配

```java
@Test
public void testArgumentMatcher() throws Exception {
    LinkedList mockedList = mock(LinkedList.class);
    //用内置的参数匹配器来stub
    when(mockedList.get(anyInt())).thenReturn("element");

    //打印 "element"
    System.out.println(mockedList.get(999));

    //你也可以用参数匹配器来验证，此处测试通过
    verify(mockedList).get(anyInt());
    //此处测试将不通过，因为没调用get(33)
    verify(mockedList).get(eq(33));
}1234567891011121314
```

> 如果你使用了参数匹配器，那么所有参数都应该使用参数匹配器

```java
 verify(mock).someMethod(anyInt(), anyString(), eq("third argument"));
 //上面是正确的，因为eq返回参数匹配器

 verify(mock).someMethod(anyInt(), anyString(), "third argument");
 //上面将会抛异常，因为第三个参数不是参数匹配器，一旦使用了参数匹配器来验证，那么所有参数都应该使用参数匹配12345
```

### 验证准确的调用次数，最多、最少、从未等

```java
@Test
public void testInvocationTimes() throws Exception {

    LinkedList mockedList = mock(LinkedList.class);

    //using mock
    mockedList.add("once");

    mockedList.add("twice");
    mockedList.add("twice");

    mockedList.add("three times");
    mockedList.add("three times");
    mockedList.add("three times");

    //下面两个是等价的， 默认使用times(1)
    verify(mockedList).add("once");
    verify(mockedList, times(1)).add("once");

    //验证准确的调用次数
    verify(mockedList, times(2)).add("twice");
    verify(mockedList, times(3)).add("three times");

    //从未调用过. never()是times(0)的别名
    verify(mockedList, never()).add("never happened");

    //用atLeast()/atMost()验证
    verify(mockedList, atLeastOnce()).add("three times");
    //下面这句将不能通过测试
    verify(mockedList, atLeast(2)).add("five times");
    verify(mockedList, atMost(5)).add("three times");
}1234567891011121314151617181920212223242526272829303132
```

### 为void方法抛异常

```java
@Test
public void testVoidMethodsWithExceptions() throws Exception {

    LinkedList mockedList = mock(LinkedList.class);
    doThrow(new RuntimeException()).when(mockedList).clear();
    //下面会抛RuntimeException
    mockedList.clear();
}12345678
```

### 验证调用顺序

```java
@Test
public void testVerificationInOrder() throws Exception {
    // A. Single mock whose methods must be invoked in a particular order
    List singleMock = mock(List.class);

    //使用单个mock对象
    singleMock.add("was added first");
    singleMock.add("was added second");

    //创建inOrder
    InOrder inOrder = inOrder(singleMock);

    //验证调用次数，若是调换两句，将会出错，因为singleMock.add("was added first")是先调用的
    inOrder.verify(singleMock).add("was added first");
    inOrder.verify(singleMock).add("was added second");

    // 多个mock对象
    List firstMock = mock(List.class);
    List secondMock = mock(List.class);

    //using mocks
    firstMock.add("was called first");
    secondMock.add("was called second");

    //创建多个mock对象的inOrder
    inOrder = inOrder(firstMock, secondMock);

    //验证firstMock先于secondMock调用
    inOrder.verify(firstMock).add("was called first");
    inOrder.verify(secondMock).add("was called second");
}12345678910111213141516171819202122232425262728293031
```

### 验证mock对象没有产生过交互

```java
@Test
public void testInteractionNeverHappened() {
    List mockOne = mock(List.class);
    List mockTwo = mock(List.class);

    //测试通过
    verifyZeroInteractions(mockOne, mockTwo);

    mockOne.add("");
    //测试不通过，因为mockTwo已经发生过交互了
    verifyZeroInteractions(mockOne, mockTwo);
}123456789101112
```

### 查找是否有未验证的交互

> 不建议过多使用，api原文：A word of warning: Some users who did a lot of classic, expect-run-verify mocking tend to use verifyNoMoreInteractions() very often, even in every test method. verifyNoMoreInteractions() is not recommended to use in every test method. verifyNoMoreInteractions() is a handy assertion from the interaction testing toolkit. Use it only when it’s relevant. Abusing it leads to overspecified, less maintainable tests.

```java
@Test
public void testFindingRedundantInvocations() throws Exception {
    List mockedList = mock(List.class);
    //using mocks
    mockedList.add("one");
    mockedList.add("two");

    verify(mockedList).add("one");

    //验证失败，因为mockedList.add("two")尚未验证
    verifyNoMoreInteractions(mockedList);
}123456789101112
```

### @Mock注解

- 减少代码
- 增强可读性
- 让verify出错信息更易读，因为变量名可用来描述标记mock对象

```java
public class MockTest {

    @Mock
    List<String> mockedList;

    @Before
    public void initMocks() {
        //必须,否则注解无效
        MockitoAnnotations.initMocks(this);
    }

    @Test
    public void testMock() throws Exception {
        mockedList.add("one");
        verify(mockedList).add("one");
    }
}1234567891011121314151617
```

### 根据调用顺序设置不同的stubbing

```java
private interface MockTest {
    String someMethod(String arg);
}

@Test
public void testStubbingConsecutiveCalls() throws Exception {

    MockTest mock = mock(MockTest.class);
    when(mock.someMethod("some arg")).thenThrow(new RuntimeException("")).thenReturn("foo");

    //第一次调用，抛RuntimeException
    mock.someMethod("some arg");

    //第二次调用返回foo
    System.out.println(mock.someMethod("some arg"));

    //后续继续调用，返回“foo”，以最后一个stub为准
    System.out.println(mock.someMethod("some arg"));

    //下面是一个更简洁的写法
    when(mock.someMethod("some arg")).thenReturn("one", "two", "three");
}12345678910111213141516171819202122
```

### doReturn()|doThrow()| doAnswer()|doNothing()|doCallRealMethod()等用法

```java
@Test
public void testDoXXX() throws Exception {
    List mockedList = mock(List.class);
    doThrow(new RuntimeException()).when(mockedList).clear();
    //以下会抛异常
    mockedList.clear();
}1234567
```

### spy监视真正的对象

- spy是创建一个拷贝，如果你保留原始的list，并用它来进行操作，那么spy并不能检测到其交互
- spy一个真正的对象+试图stub一个final方法，这样是会有问题的

```java
@Test
public void testSpy() throws Exception {
    List list = new LinkedList();
    List spy = spy(list);

    //可选的，你可以stub某些方法
    when(spy.size()).thenReturn(100);

    //调用"真正"的方法
    spy.add("one");
    spy.add("two");

    //打印one
    System.out.println(spy.get(0));

    //size()方法被stub了，打印100
    System.out.println(spy.size());

    //可选，验证spy对象的行为
    verify(spy).add("one");
    verify(spy).add("two");

    //下面写法有问题，spy.get(10)会抛IndexOutOfBoundsException异常
    when(spy.get(10)).thenReturn("foo");
    //可用以下方式
    doReturn("foo").when(spy).get(10);
}123456789101112131415161718192021222324252627
```

### 为未stub的方法设置默认返回值

```java
@Test
public void testDefaultValue() throws Exception {

    List listOne = mock(List.class, Mockito.RETURNS_SMART_NULLS);
    List listTwo = mock(List.class, new Answer() {
        @Override
        public Object answer(InvocationOnMock invocation) throws Throwable {

            // TODO: 2016/6/13 return default value here
            return null;
        }
    });
}12345678910111213
```

### 参数捕捉

```java
@Test
public void testCapturingArguments() throws Exception {
    List mockedList = mock(List.class);
    ArgumentCaptor<String> argument = ArgumentCaptor.forClass(String.class);
    mockedList.add("John");
    //验证后再捕捉参数
    verify(mockedList).add(argument.capture());
    //验证参数
    assertEquals("John", argument.getValue());
}12345678910
```

### 真正的部分模拟（TODO：尚未搞清楚啥意思。。。）

```java
    //you can create partial mock with spy() method:
    List list = spy(new LinkedList());

    //you can enable partial mock capabilities selectively on mocks:
    Foo mock = mock(Foo.class);
    //Be sure the real implementation is 'safe'.
    //If real implementation throws exceptions or depends on specific state of the object then you're in trouble.
    when(mock.someMethod()).thenCallRealMethod();12345678
```

### 重置mocks

> Don’t harm yourself. reset() in the middle of the test method is a code smell (you’re probably testing too much).

```java
@Test
public void testReset() throws Exception {
    List mock = mock(List.class);
    when(mock.size()).thenReturn(10);
    mock.add(1);
    reset(mock);
    //从这开始，之前的交互和stub将全部失效
}12345678
```

### Serializable mocks

> WARNING: This should be rarely used in unit testing.

```java
@Test
public void testSerializableMocks() throws Exception {
    List serializableMock = mock(List.class, withSettings().serializable());
}1234
```

### 更多的注解：@Captor, @Spy, @InjectMocks

- @Captor 创建ArgumentCaptor
- @Spy 可以代替spy(Object).
- @InjectMocks 如果此注解声明的变量需要用到mock对象，mockito会自动注入mock或spy成员

```java
//可以这样写
@Spy BeerDrinker drinker = new BeerDrinker();
//也可以这样写，mockito会自动实例化drinker
@Spy BeerDrinker drinker;

//会自动实例化
@InjectMocks LocalPub;1234567
```

### 超时验证

```java
private interface TimeMockTest {
    void someMethod();
}

@Test
public void testTimeout() throws Exception {

    TimeMockTest mock = mock(TimeMockTest.class);
    //测试程序将会在下面这句阻塞100毫秒，timeout的时候再进行验证是否执行过someMethod()
    verify(mock, timeout(100)).someMethod();
    //和上面代码等价
    verify(mock, timeout(100).times(1)).someMethod();

    //阻塞100ms，timeout的时候再验证是否刚好执行了2次
    verify(mock, timeout(100).times(2)).someMethod();

    //timeout的时候，验证至少执行了2次
    verify(mock, timeout(100).atLeast(2)).someMethod();

    //timeout时间后，用自定义的检验模式验证someMethod()
    VerificationMode yourOwnVerificationMode = new VerificationMode() {
        @Override
        public void verify(VerificationData data) {
            // TODO: 2016/12/4 implement me
        }
    };
    verify(mock, new Timeout(100, yourOwnVerificationMode)).someMethod();
}12345678910111213141516171819202122232425262728
```

### 查看是否mock或者spy

```java
   Mockito.mockingDetails(someObject).isMock();
   Mockito.mockingDetails(someObject).isSpy();
```