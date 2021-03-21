# 【Json】Json解析工具--Jackson

## 一、Overview

目前国内的 Java 项目中主流的 Json 解析工具应该就两大类: 阿里的 FastJson 和 谷歌的 Gson。因为最近在用 Scala 开发, scala 中有一小众工具也很优秀, 它就是 Json4s。

但是最近在一个 spark 项目, 遇到一个报错, jackson-databind 版本问题, 检查后发现Spark 中使用了 Jackson 和 Json4s 中的 Jackson 版本冲突了。

于是很好奇, 翻墙后才知道 Jackson 在国外项目中用的极多, 且它性能非常优秀。项目中如果要解析 Json, 建议使用 Jackson 或谷歌的 Gson, Gson 完全实现了自己对于 Json 的解析, 它没有依赖 Jackson, 所以使用 Gson 不会存在和其他框架之间存在 Jackson 版本冲突的问题。

相对于其他 json 解析库，诸如 json-lib、gson包，Jackson具有以下优点：

- 功能全面，提供多种模式的 json 解析方式，“对象绑定”使用方便，利用注解包能为我们开发提供很多便利。

- 性能较高，“流模式”的解析效率超过绝大多数类似的 json 包。

发展至如今的 Jackson 存在两大分支, 也是两个版本的不同包名。Jackson 在 2.0 之前的包名使用的是 codehaus, 从 2.0 开始改用新的包名 fasterxml；除了包名不同，他们的 Maven artifact id 也不同。1.x 版本现在只提供 bug-fix，而 2.x 版本还在不断开发和发布中。如果是新项目，建议直接用 2.x，即 fasterxml jackson。

### 1.1 fasterxml jackson 

它的核心 jar 有三个：

- jackson-core 核心包(必备), 提供基于“流模式”解析的API。
- jackson-annotations 注解包(可选)，提供注解功能。 
- jackson-databind 数据绑定包(可选)，提供基于“对象绑定”和“树模型”相关API。

它们之间的依赖关系是: databind 依赖于 core 和 annotations。所以只需要在项目中引入databind，其他两个就会自动引入

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.1.0</version>
</dependency>
```

### 1.2 codehaus jackson

而它的核心 jar 只有两个:

- jackson-core-asl
- jackson-mapper-asl

它们之间的依赖关系是: mapper-asl 依赖于 core-asl。所以只需要引入 jackson-mapper-asl 的依赖就可以了。　　

```xml
<dependency>
  <groupId>org.codehaus.jackson</groupId>
  <artifactId>jackson-mapper-asl</artifactId>
  <version>1.9.11</version>
</dependency>
```



## 二、Tutorial

本章翻译自[baeldung的博客](https://www.baeldung.com/jackson)

### 2.1 Jackson Annotation Examples

在本教程中，我们将深入探讨 Jackson Annotations。我们将看到如何使用现有的注释，如何创建自定义注释，最后是如何禁用它们。

### 2.2 Jackson 序列化注释

首先，我们来看看序列化注释。

#### 2.2.1 @JsonAnyGetter

*@JsonAnyGetter* 注释非常的灵活, 它允许将*Map*字段作为标准属性输出。

- 方法是非静态，没有参数的,方法名随意
- 方法返回值必须是Map类型
- 在一个实体类中仅仅用在一个方法上
- 序列化的时候 json 字段的 key 就是返回 Map 的 key,value 就是 Map 的 value

例如，*ExtendableBean*实体具有*name*属性和一组*key/value*对形式的可扩展属性：

```java
public class ExtendableBean {
  public String name;
  private Map<String, String> properties;

  @JsonAnyGetter
  public Map<String, String> getProperties() {
    return properties;
  }
}
```

序列化 ExtendableBean 的实例时，我们将*Map*中的所有键值作为标准的纯属性获取：

```json
{
  "name":"My bean",
  "attr2":"val2",
  "attr1":"val1"
}
```

在实践中，此实体的序列化外观如下：

```java
@Test
public void whenSerializingUsingJsonAnyGetter_thenCorrect()
  throws JsonProcessingException {

  ExtendableBean bean = new ExtendableBean("My bean");
  bean.add("attr1", "val1");
  bean.add("attr2", "val2");

  String result = new ObjectMapper().writeValueAsString(bean);

  assertThat(result, containsString("attr1"));
  assertThat(result, containsString("val1"));
}
```

我们还可以将可选参数*enabled*设置为*false*的来禁用*@JsonAnyGetter()*, 在这种情况下，*Map*将转换为JSON，并在序列化后显示在*properties*变量下。

#### 2.2.2 @JsonGetter

注释 *@JsonGetter*是对注释 *@JsonProperty* 的一个替代，其标记的方法作为getter方法。

在下面的示例中，我们将方法*getTheName（）*指定为*MyBean*实体的*name*属性的getter方法：

```java
public class MyBean {
  public int id;
  private String name;

  @JsonGetter("name")
  public String getTheName() {
    return name;
  }
}
```

这实际上是如何工作的：

```java
@Test
public void whenSerializingUsingJsonGetter_thenCorrect()
  throws JsonProcessingException {

  MyBean bean = new MyBean(1, "My bean");

  String result = new ObjectMapper().writeValueAsString(bean);
  assertThat(result, containsString("My bean"));
  assertThat(result, containsString("1"));
}
```

#### 2.2.3 @JsonPropertyOrder

我们可以使用 *@JsonPropertyOrder* 注释指定**序列化属性的顺序**。

让我们为 *MyBean *实体的属性设置自定义顺序：

```java
@JsonPropertyOrder({ "name", "id" })
public class MyBean {
  public int id;
  public String name;
}
```

这是序列化的输出：

```bash
{
	"name":"My bean",
	"id":1
}
```

然后我们可以做一个简单的测试：

```java
@Test
public void whenSerializingUsingJsonPropertyOrder_thenCorrect()
  throws JsonProcessingException {

  MyBean bean = new MyBean(1, "My bean");

  String result = new ObjectMapper().writeValueAsString(bean);
  assertThat(result, containsString("My bean"));
  assertThat(result, containsString("1"));
}
```

我们还可以使用 *@JsonPropertyOrder(alphabetic = true)*按字母顺序对属性进行排序。在这种情况下，序列化的输出将是：

```bash
{
    "id":1,
    "name":"My bean"
}
```

#### 2.2.4 @JsonRawValue

*@JsonRawValue注释* 可以指示 Jackson 按原样序列化属性。

在以下示例中，我们使用*@JsonRawValue*嵌入一些自定义 JSON 作为实体的值：

```java
public class RawBean {
  public String name;

  @JsonRawValue
  public String json;
}
```

序列化实体的输出为：

```java
{
  "name":"My bean",
  "json":{
    "attr":false
  }
}
```

接下来是一个简单的测试：

```java
@Test
public void whenSerializingUsingJsonRawValue_thenCorrect()
  throws JsonProcessingException {

  RawBean bean = new RawBean("My bean", "{\"attr\":false}");

  String result = new ObjectMapper().writeValueAsString(bean);
  assertThat(result, containsString("My bean"));
  assertThat(result, containsString("{\"attr\":false}"));
}
```

我们还可以使用可选的 boolean 型参数 *value*，定义此注释是否处于活动状态。

#### 2.2.5 @JsonRootName

启用 *@JsonRootName*注释时，以指定包装中使用的根目录的名称。

包装意味着不要将 *User* 类序列化为以下内容：

```javascript
{
    "id": 1,
    "name": "John"
}
```

它会像这样包装：

```javascript
{
    "User": {
        "id": 1,
        "name": "John"
    }
}
```

因此，让我们看一个例子。我们将使用 *@JsonRootName* 注释，以表明这个潜在的包装实体的名称：

```java
@JsonRootName(value = "user")
public class UserWithRoot {
  public int id;
  public String name;
}
```

默认情况下，包装器的名称将为类的名称– *UserWithRoot*。通过使用注释，我们得到了看上去更干净的*用户：*

```java
@Test
public void whenSerializingUsingJsonRootName_thenCorrect()
  throws JsonProcessingException {

  UserWithRoot user = new User(1, "John");

  ObjectMapper mapper = new ObjectMapper();
  mapper.enable(SerializationFeature.WRAP_ROOT_VALUE);
  String result = mapper.writeValueAsString(user);

  assertThat(result, containsString("John"));
  assertThat(result, containsString("user"));
}
```

这是序列化的输出：

```java
{
    "user":{
        "id":1,
        "name":"John"
    }
}
```

从Jackson 2.4开始，新的可选参数 *namespace* 可用于XML等数据格式。如果添加它，它将成为标准名称的一部分：

```java
@JsonRootName(value = "user", namespace="users")
public class UserWithRootNamespace {
  public int id;
  public String name;
  // ...
}
```

如果我们使用*XmlMapper*对其进行序列化*，*输出将是：

```xml
<user xmlns="users">
  <id xmlns="">1</id>
  <name xmlns="">John</name>
  <items xmlns=""/>
</user>
```

#### 2.2.6 @JsonSerialize

*@JsonSerialize* 表示在编组实体时要使用的自定义序列化程序。

让我们看一个简单的例子。我们将使用 *@JsonSerialize* 通过*CustomDateSerializer*来序列化*eventDate*属性：

```java
public class EventWithSerializer {
  public String name;

  @JsonSerialize(using = CustomDateSerializer.class)
  public Date eventDate;
}
```

这是简单的自定义Jackson序列化器：

```java
public class CustomDateSerializer extends StdSerializer<Date> {

  private static SimpleDateFormat formatter 
    = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");

  public CustomDateSerializer() { 
    this(null); 
  } 

  public CustomDateSerializer(Class<Date> t) {
    super(t); 
  }

  @Override
  public void serialize(
    Date value, JsonGenerator gen, SerializerProvider arg2) 
    throws IOException, JsonProcessingException {
    gen.writeString(formatter.format(value));
  }
}
```

现在让我们在测试中使用它们：

```java
@Test
public void whenSerializingUsingJsonSerialize_thenCorrect()
  throws JsonProcessingException, ParseException {

  SimpleDateFormat df
    = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");

  String toParse = "20-12-2014 02:30:00";
  Date date = df.parse(toParse);
  EventWithSerializer event = new EventWithSerializer("party", date);

  String result = new ObjectMapper().writeValueAsString(event);
  assertThat(result, containsString(toParse));
}
```

### 2.3 Jackson 反序列化注释

接下来，让我们探索 Jackson 反序列化注释。

#### 2.3.1 @JsonCreator

我们可以使用 *@JsonCreator* 来调整反序列化中使用的构造函数/工厂。

当我们需要反序列化一些与我们需要获取的目标实体不完全匹配的JSON时，这非常有用。

让我们来看一个例子。假设我们需要反序列化以下JSON：

```bash
{
	"id":1,
  "theName":"My bean"
}
```

但是，我们的目标实体中没有*theName*字段，只有*name*字段。现在，我们不希望用注释的构造改变实体本身，我们只需要在解组过程中一点点更多的控制*@JsonCreator，*并使用*@JsonProperty*注释以及：

```java
public class BeanWithCreator {
  public int id;
  public String name;

  @JsonCreator
  public BeanWithCreator(
    @JsonProperty("id") int id, 
    @JsonProperty("theName") String name) {
    this.id = id;
    this.name = name;
  }
}
```

让我们来看看实际情况：

```java
@Test
public void whenDeserializingUsingJsonCreator_thenCorrect()
  throws IOException {

  String json = "{\"id\":1,\"theName\":\"My bean\"}";

  BeanWithCreator bean = new ObjectMapper()
    .readerFor(BeanWithCreator.class)
    .readValue(json);
  assertEquals("My bean", bean.name);
}
```

#### 2.3.2 @JacksonInject

*@JacksonInject* 表示属性将从注入中获取其值，而不是从JSON数据中获取。

在下面的示例中，我们使用*@JacksonInject*注入属性*id*：

```java
public class BeanWithInject {
  @JacksonInject
  public int id;

  public String name;
}
```

运作方式如下：

```java
@Test
public void whenDeserializingUsingJsonInject_thenCorrect()
  throws IOException {

  String json = "{\"name\":\"My bean\"}";

  InjectableValues inject = new InjectableValues.Std()
    .addValue(int.class, 1);
  BeanWithInject bean = new ObjectMapper().reader(inject)
    .forType(BeanWithInject.class)
    .readValue(json);

  assertEquals("My bean", bean.name);
  assertEquals(1, bean.id);
}
```

#### 2.3.3 @JsonAnySetter

- 用在非静态方法上，注解的方法必须有两个参数，第一个是json字段中的key，第二个是value
- 方法名随意
- 也可以用在Map对象属性上面，建议用在Map对象属性上面，简单呀
- 反序列化的时候将对应不上的字段全部放到Map里面

示例:

```java
public class User {
  private String username;
  private String password;
  private Integer age;
  @JsonSetter  //方法和属性上只需要一个就可以
  private Map<String,String> map = new HashMap<>();

  /*@JsonSetter
  public void testGet(String key, String value) {
  	map.put(key,value);
  }*/
}
```

测试方法，自己重写toString

```java
public static void main(String[] args) throws IOException {
  String jsonStr = 
    "{" +
    		"\"username\":\"wkw\"," +
    		"\"password\":\"123\"," +
    		"\"age\":null," +
    		"\"test2\":\"testTwo\"," +
    		"\"test1\":\"testOne\"" +
		"}";
  ObjectMapper objectMapper = new ObjectMapper();
  User user = objectMapper.readValue(jsonStr, User.class);
  System.out.println(user);
}
/*
	User{username='wkw', password='123', age=null, map={test2=testTwo, test1=testOne}}
 */
```

#### 2.3.4 @JsonSetter

*@JsonSetter*是一种替代 *@JsonProperty* 该标记的方法作为setter方法。

当我们需要读取一些JSON数据，但是目标实体类与该数据不完全匹配时，这个注释是非常有用的，因为我们可以调整过程以使其适合。

在以下示例中，我们将在*MyBean*实体中将方法 *setTheName()* 指定为*name*属性的设置方法：

```java
public class MyBean {
  public int id;
  private String name;

  @JsonSetter("Name")
  public void setName(String name) {
    this.name = name;
  }
}
```

现在，当我们需要解组一些 JSON 数据时，这可以很好地工作：

```java
@Test
public void whenDeserializingUsingJsonSetter_thenCorrect()
  throws IOException {

  String json = "{\"id\":1,\"Name\":\"My bean\"}";

  MyBean bean = new ObjectMapper()
    .readerFor(MyBean.class)
    .readValue(json);
  assertEquals("My bean", bean.getName());
}
```

#### 2.3.6 @JsonAlias

*@JsonAlias* 定义反序列化过程为属性的一个或多个的替代名称。

让我们用一个简单的例子看看这个注释是如何工作的：

```java
public class AliasBean {
  @JsonAlias({ "fName", "f_name" })
  private String firstName;   
  private String lastName;
}
```

这里有一个POJO，我们想将*fName* / *f_name* 和 *firstName* 等值反序列化 JSON 到 POJO 的 *firstName* 变量中。

下面是一项测试，确保此批注按预期工作：

```java
@Test
public void whenDeserializingUsingJsonAlias_thenCorrect() throws IOException {
  String json = "{\"fName\": \"John\", \"lastName\": \"Green\"}";
  AliasBean aliasBean = new ObjectMapper().readerFor(AliasBean.class).readValue(json);
  assertEquals("John", aliasBean.getFirstName());
}
```

### 2.4 Jackson 属性包含注释

#### 2.4.1 @JsonIgnoreProperties

*@JsonIgnoreProperties* 是一个类级别的注释，用于标记一个属性或一系列属性, 确保它们会被 Jackson 忽略。

让我们看一个简单的示例，该示例忽略序列化中的属性*ID*：

```java
@JsonIgnoreProperties({ "id" })
public class BeanWithIgnore {
  public int id;
  public String name;
}
```

现在是确保忽略发生的测试：

```java
@Test
public void whenSerializingUsingJsonIgnoreProperties_thenCorrect()
  throws JsonProcessingException {

  BeanWithIgnore bean = new BeanWithIgnore(1, "My bean");
  String result = new ObjectMapper()
    .writeValueAsString(bean);

  assertThat(result, containsString("My bean"));
  assertThat(result, not(containsString("id")));
}
```

我们可以设置注解 *@JsonIgnoreProperties* 的参数 *ignoreUnknown=true*, 来忽略 JSON 输入中所有未知的属性。

#### 2.4.2 @JsonIgnore

相反，*@JsonIgnore* 批注用于标记在字段级别要忽略的属性。

让我们使用 *@JsonIgnore* 忽略序列化中的属性*ID*：

```java
public class BeanWithIgnore {
  @JsonIgnore
  public int id;

  public String name;
}
```

然后，我们将进行测试以确保*ID*被成功忽略：

```java
@Test
public void whenSerializingUsingJsonIgnore_thenCorrect()
  throws JsonProcessingException {

  BeanWithIgnore bean = new BeanWithIgnore(1, "My bean");

  String result = new ObjectMapper()
    .writeValueAsString(bean);

  assertThat(result, containsString("My bean"));
  assertThat(result, not(containsString("id")));
}
```

#### 2.4.3 @JsonIgnoreType

*@JsonIgnoreType* 将带注释类型的所有属性标记为忽略。

我们可以使用注释来标记要忽略的*Name*类型的所有属性：

```java
public class User {
  public int id;
  public Name name;

  @JsonIgnoreType
  public static class Name {
    public String firstName;
    public String lastName;
  }
}
```

我们还可以进行测试以确保忽略操作正确进行：

```java
@Test
public void whenSerializingUsingJsonIgnoreType_thenCorrect()
  throws JsonProcessingException, ParseException {

  User.Name name = new User.Name("John", "Doe");
  User user = new User(1, name);

  String result = new ObjectMapper()
    .writeValueAsString(user);

  assertThat(result, containsString("1"));
  assertThat(result, not(containsString("name")));
  assertThat(result, not(containsString("John")));
}
```

#### 2.4.4 @JsonInclude

我们可以使用 *@JsonInclude* 排除具有空/空/默认值的属性。

让我们看一个从序列化中排除空值的示例：

```java
@JsonInclude(Include.NON_NULL)
public class MyBean {
  public int id;
  public String name;
}
```

这是完整的测试：

```java
public void whenSerializingUsingJsonInclude_thenCorrect()
  throws JsonProcessingException {

  MyBean bean = new MyBean(1, null);

  String result = new ObjectMapper()
    .writeValueAsString(bean);

  assertThat(result, containsString("1"));
  assertThat(result, not(containsString("name")));
}
```

#### 2.4.5 @JsonAutoDetect

*@JsonAutoDetect*可以覆盖默认语义，即哪些属性可见，哪些属性不可见。

首先，让我们来看一个简单的示例，该注释如何非常有用；让我们启用序列化私有属性：

```java
@JsonAutoDetect(fieldVisibility = Visibility.ANY)
public class PrivateBean {
  private int id;
  private String name;
}
```

然后测试：

```java
@Test
public void whenSerializingUsingJsonAutoDetect_thenCorrect()
  throws JsonProcessingException {

  PrivateBean bean = new PrivateBean(1, "My bean");

  String result = new ObjectMapper()
    .writeValueAsString(bean);

  assertThat(result, containsString("1"));
  assertThat(result, containsString("My bean"));
}
```

> jackson默认的字段属性发现规则如下：
>
> 所有被 public 修饰的字段->所有被 public 修饰的 getter->所有被 public 修饰的 setter

### 2.5 Jackson 多态类型处理注释

接下来，让我们看一下 Jackson 的多态类型处理注释：

- *@JsonTypeInfo* – 指示要在序列化中包含哪些类型信息的详细信息
- *@JsonSubTypes* – 指示带注释类型的子类型
- *@JsonTypeName* – 定义用于注释类的逻辑类型名称

让我们来看一个更复杂的示例，并使用全部三个*@JsonTypeInfo*，*@JsonSubTypes*和*@JsonTypeName*来序列化/反序列化实体*Zoo*：

```java
public class Zoo {
  public Animal animal;

  @JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME, 
    include = As.PROPERTY, 
    property = "type")
  @JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "cat")
  })
  public static class Animal {
    public String name;
  }

  @JsonTypeName("dog")
  public static class Dog extends Animal {
    public double barkVolume;
  }

  @JsonTypeName("cat")
  public static class Cat extends Animal {
    boolean likesCream;
    public int lives;
  }
}
```

当我们进行序列化时：

```java
@Test
public void whenSerializingPolymorphic_thenCorrect()
  throws JsonProcessingException {
  Zoo.Dog dog = new Zoo.Dog("lacy");
  Zoo zoo = new Zoo(dog);

  String result = new ObjectMapper()
    .writeValueAsString(zoo);

  assertThat(result, containsString("type"));
  assertThat(result, containsString("dog"));
}
```

这是使用*Dog*序列化*Zoo*实例的结果：

```javascript
{
    "animal": {
        "type": "dog",
        "name": "lacy",
        "barkVolume": 0
    }
}
```

现在进行反序列化。让我们从以下JSON输入开始：

```bash
{
    "animal":{
        "name":"lacy",
        "type":"cat"
    }
}
```

然后让我们看看如何将其解组到*Zoo*实例：

```java
@Test
public void whenDeserializingPolymorphic_thenCorrect()
  throws IOException {
  String json = "{\"animal\":{\"name\":\"lacy\",\"type\":\"cat\"}}";

  Zoo zoo = new ObjectMapper()
    .readerFor(Zoo.class)
    .readValue(json);

  assertEquals("lacy", zoo.animal.name);
  assertEquals(Zoo.Cat.class, zoo.animal.getClass());
}
```

### 2.6 Jackson 通用注释

接下来，让我们讨论一下 Jackson 的一些更一般的注释。

#### 2.6.1 @JsonProperty

我们可以添加的 *@JsonProperty* 注释，以表明在 JSON 中的属性名。

在处理非标准的 getter 和 setter 时，让我们使用 *@JsonProperty* 对属性*名称*进行序列化/反序列化：

```java
public class MyBean {
  public int id;
  private String name;

  @JsonProperty("name")
  public void setTheName(String name) {
    this.name = name;
  }

  @JsonProperty("name")
  public String getTheName() {
    return name;
  }
}
```

接下来是我们的测试：

```java
@Test
public void whenUsingJsonProperty_thenCorrect()
  throws IOException {
  MyBean bean = new MyBean(1, "My bean");

  String result = new ObjectMapper().writeValueAsString(bean);

  assertThat(result, containsString("My bean"));
  assertThat(result, containsString("1"));

  MyBean resultBean = new ObjectMapper()
    .readerFor(MyBean.class)
    .readValue(result);
  assertEquals("My bean", resultBean.getTheName());
}
```

#### 2.6.2 @JsonFormat

该注解 *@JsonFormat* 使用指定的格式来序列化日期/时间值。

在下面的示例中，我们使用*@JsonFormat*来控制属性*eventDate*的格式：

```java
public class EventWithFormat {
  public String name;

  @JsonFormat(
    shape = JsonFormat.Shape.STRING,
    pattern = "dd-MM-yyyy hh:mm:ss")
  public Date eventDate;
}
```

然后是测试：

```java
@Test
public void whenSerializingUsingJsonFormat_thenCorrect()
  throws JsonProcessingException, ParseException {
  SimpleDateFormat df = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");
  df.setTimeZone(TimeZone.getTimeZone("UTC"));

  String toParse = "20-12-2014 02:30:00";
  Date date = df.parse(toParse);
  EventWithFormat event = new EventWithFormat("party", date);

  String result = new ObjectMapper().writeValueAsString(event);

  assertThat(result, containsString(toParse));
}
```

#### 2.6.3 @JsonUnwrapped

*@JsonUnwrapped* 定义在序列化/反序列化时应该解包/展平的值。

让我们确切地了解它是如何工作的。我们将使用注释解开属性 *name*：

```java
public class UnwrappedUser {
  public int id;

  @JsonUnwrapped
  public Name name;

  public static class Name {
    public String firstName;
    public String lastName;
  }
}
```

现在让我们序列化此类的实例：

```java
@Test
public void whenSerializingUsingJsonUnwrapped_thenCorrect()
  throws JsonProcessingException, ParseException {
  UnwrappedUser.Name name = new UnwrappedUser.Name("John", "Doe");
  UnwrappedUser user = new UnwrappedUser(1, name);

  String result = new ObjectMapper().writeValueAsString(user);

  assertThat(result, containsString("John"));
  assertThat(result, not(containsString("name")));
}
```

最后，输出结果如下所示：静态嵌套类的字段与其他字段一起展开：

```java
{
    "id":1,
    "firstName":"John",
    "lastName":"Doe"
}
```

#### 2.6.4 @JsonView

*@JsonView* 指示将在其中包含属性以进行序列化/反序列化的View。

例如，我们将使用*@JsonView*序列化*Item*实体的实例。

首先，让我们从视图开始：

```java
public class Views {
  public static class Public {}
  public static class Internal extends Public {}
}
```

接下来是使用视图的*Item*实体：

```java
public class Item {
    @JsonView(Views.Public.class)
    public int id;

    @JsonView(Views.Public.class)
    public String itemName;

    @JsonView(Views.Internal.class)
    public String ownerName;
}
```

最后，全面测试：

```java
@Test
public void whenSerializingUsingJsonView_thenCorrect()
  throws JsonProcessingException {
  
  Item item = new Item(2, "book", "John");

  String result = new ObjectMapper()
    .writerWithView(Views.Public.class)
    .writeValueAsString(item);

  assertThat(result, containsString("book"));
  assertThat(result, containsString("2"));
  assertThat(result, not(containsString("John")));
}
```

#### 2.6.5 @JsonManagedReference，@JsonBackReference

该 *@JsonManagedReference* 和 *@JsonBackReference* 注释可以处理父/子关系围绕循环工作。

在以下示例中，我们使用*@JsonManagedReference*和*@JsonBackReference*来序列化*ItemWithRef*实体：

```java
public class ItemWithRef {
    public int id;
    public String itemName;

    @JsonManagedReference
    public UserWithRef owner;
}
```

我们的*UserWithRef*实体：

```java
public class UserWithRef {
  public int id;
  public String name;

  @JsonBackReference
  public List<ItemWithRef> userItems;
}
```

然后测试：

```java
@Test
public void whenSerializingUsingJacksonReferenceAnnotation_thenCorrect()
  throws JsonProcessingException {
  UserWithRef user = new UserWithRef(1, "John");
  ItemWithRef item = new ItemWithRef(2, "book", user);
  user.addItem(item);

  String result = new ObjectMapper().writeValueAsString(item);

  assertThat(result, containsString("book"));
  assertThat(result, containsString("John"));
  assertThat(result, not(containsString("userItems")));
}
```

#### 2.6.6 @JsonIdentityInfo

*@JsonIdentityInfo*指示在对值进行序列化/反序列化时（例如，在处理无限递归类型的问题时）应使用对象标识。

在下面的例子中，我们有一个 *ItemWithIdentity* 用的双向关系的实体 *UserWithIdentity*实体：

```java
@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id")
public class ItemWithIdentity {
  public int id;
  public String itemName;
  public UserWithIdentity owner;
}
```

该*UserWithIdentity*实体：

```java
@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id")
public class UserWithIdentity {
  public int id;
  public String name;
  public List<ItemWithIdentity> userItems;
}
```

现在**让我们看一下无限递归问题是如何处理的**：

```java
@Test
public void whenSerializingUsingJsonIdentityInfo_thenCorrect()
  throws JsonProcessingException {
  UserWithIdentity user = new UserWithIdentity(1, "John");
  ItemWithIdentity item = new ItemWithIdentity(2, "book", user);
  user.addItem(item);

  String result = new ObjectMapper().writeValueAsString(item);

  assertThat(result, containsString("book"));
  assertThat(result, containsString("John"));
  assertThat(result, containsString("userItems"));
}
```

这是序列化项目和用户的完整输出：

```javascript
{
    "id": 2,
    "itemName": "book",
    "owner": {
        "id": 1,
        "name": "John",
        "userItems": [
            2
        ]
    }
}
```

#### 2.6.7 @JsonFilter

*@JsonFilter* 注释指定序列化过程中使用一个过滤器。

首先，我们定义实体并指向过滤器：

```java
@JsonFilter("myFilter")
public class BeanWithFilter {
  public int id;
  public String name;
}
```

现在，在完整测试中，我们定义过滤器，该过滤器从序列化中排除*名称*以外的所有其他属性：

```java
@Test
public void whenSerializingUsingJsonFilter_thenCorrect()
  throws JsonProcessingException {
  BeanWithFilter bean = new BeanWithFilter(1, "My bean");

  FilterProvider filters 
    = new SimpleFilterProvider().addFilter(
    "myFilter", 
    SimpleBeanPropertyFilter.filterOutAllExcept("name"));

  String result = new ObjectMapper()
    .writer(filters)
    .writeValueAsString(bean);

  assertThat(result, containsString("My bean"));
  assertThat(result, not(containsString("id")));
}
```