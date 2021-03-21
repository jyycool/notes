## 【SpringBoot】配置文件、加载顺序及配置原理

### 一、配置文件

1. SpringBoot 使用一个全局的配置文件

   - application.properties

   - application.yml

2. 配置文件的存放路径

   - src/main/resources
   - 类路径/config

3. 全局配置文件的作业 : 可以修改 spring boot 的自动的配置值



### 二、YAML语法

YAML 全称为 YAML Ain't Markup Language, 它包含以下两种意思

-  YAML is A Markup Language
-  YAML isn't Markup Language

它是以数据为中心, 比 json、xml 更适合做配置文件.

在 YAML 中如果我们配置 Tomcat 的端口号为 8090, 它的写法如下:

```yaml
server:
 port: 8090
```



#### 2.1 YAML的基本语法

- 使用缩进表示层级关系
- ***缩进时不允许使用 TAB键, 只允许使用空格键***

- 缩进的空格数目不重要, 只要相同层级左对齐即可
- 大小写敏感
- ***值和键的冒号之间需要有一个空格***

```yaml
person:
 name: lucy
 age: 15
 gender: female
```



#### 2.2 多文档支持

- YAML支持使用三个破折号分割文档**---**
- **---** 上下两部分会作为两个独立的文档同等对待



#### 2.3 注释

- 使用**#**表明注释，从*字符一直到行尾，都会被解析器忽略



#### 2.4 数据结构

##### 2.2.1 常量

常量值最基本的数字、字符串、布尔值等等

Tips: 特殊字符需要加单引号

```yaml
number-value: 42
floating-point-value: 3.141592
boolean-value: true
# 对于字符串，在YAML中允许不添加引号。
# 但是 '和" 的含义不同
# ""中内容就是纯字符串内容
# ''中内容如果可以转义则会被转义, 如 '\n'表示换行
string-value: 'Bonjour'
```



##### 2.4.2 字典

- 使用**：**表征一个键值对 `my_key: my_value`
- 冒号后面必须间隔至少一个空格

```yaml
my_key: my_value
#或者表达为：
my_key:
  my_value
```



##### 2.4.3 字典嵌套列表(类数组, 列表 )

- 破折号可用于描述列表结构
- 破折号后必须有空格做间隔
   列表和键值对嵌套使用：

```yaml
my_dictionary:
  - list_value_one
  - list_value_two
  - list_value_three
```



#### 2.5 区块字符

  再次强调，字符串不需要包在引号之内。有两种方法书写多行文字（multi-line strings），一种可以保存新行（使用|字符），另一种可以折叠新行（使用>字符）

- 保存新行(Newlines preserved)

```yaml
data: |                                     # 译者注：這是一首著名的五行民谣
   There once was a man from Darjeeling     # 这里曾有一个人來自大吉领
   Who got on a bus bound for Ealing        # 他搭上一班往伊灵的公车
       It said on the door                  # 门上这么说的
       "Please don't spit on the floor"     # "请勿在地上吐痰"
   So he carefully spat on the ceiling      # 所以他小心翼翼的吐在天花板上
```



- 折叠新行(Newlines folded)

```yaml
data: >
   Wrapped text         # 摺疊的文字
   will be folded       # 將會被收
   into a single        # 進單一一個
   paragraph            # 段落
   
   Blank lines denote   # 空白的行代表
   paragraph breaks     # 段落之间的间隔
```

和保存新行不同的是，换行字符会被转换成空白字符。而引领空白字符则会被自动消去。



#### 2.6 YAML小抄

```yaml
%YAML 1.1   # Reference card
---
Collection indicators:
    '? ' : Key indicator.
    ': ' : Value indicator.
    '- ' : Nested series entry indicator.
    ', ' : Separate in-line branch entries.
    '[]' : Surround in-line series branch.
    '{}' : Surround in-line keyed branch.
Scalar indicators:
    '''' : Surround in-line unescaped scalar ('' escaped ').
    '"'  : Surround in-line escaped scalar (see escape codes below).
    '|'  : Block scalar indicator.
    '>'  : Folded scalar indicator.
    '-'  : Strip chomp modifier ('|-' or '>-').
    '+'  : Keep chomp modifier ('|+' or '>+').
    1-9  : Explicit indentation modifier ('|1' or '>2').
           # Modifiers can be combined ('|2-', '>+1').
Alias indicators:
    '&'  : Anchor property.
    '*'  : Alias indicator.
Tag property: # Usually unspecified.
    none    : Unspecified tag (automatically resolved by application).
    '!'     : Non-specific tag (by default, "!!map"/"!!seq"/"!!str").
    '!foo'  : Primary (by convention, means a local "!foo" tag).
    '!!foo' : Secondary (by convention, means "tag:yaml.org,2002:foo").
    '!h!foo': Requires "%TAG !h! <prefix>" (and then means "<prefix>foo").
    '!<foo>': Verbatim tag (always means "foo").
Document indicators:
    '%'  : Directive indicator.
    '---': Document header.
    '...': Document terminator.
Misc indicators:
    ' #' : Throwaway comment indicator.
    '`@' : Both reserved for future use.
Special keys:
    '='  : Default "value" mapping key.
    '<<' : Merge keys from another mapping.
Core types: # Default automatic tags.
    '!!map' : { Hash table, dictionary, mapping }
    '!!seq' : { List, array, tuple, vector, sequence }
    '!!str' : Unicode string
More types:
    '!!set' : { cherries, plums, apples }
    '!!omap': [ one: 1, two: 2 ]
Language Independent Scalar types:
    { ~, null }              : Null (no value).
    [ 1234, 0x4D2, 02333 ]   : [ Decimal int, Hexadecimal int, Octal int ]
    [ 1_230.15, 12.3015e+02 ]: [ Fixed float, Exponential float ]
    [ .inf, -.Inf, .NAN ]    : [ Infinity (float), Negative, Not a number ]
    { Y, true, Yes, ON  }    : Boolean true
    { n, FALSE, No, off }    : Boolean false
    ? !!binary >
        R0lG...BADS=
    : >-
        Base 64 binary value.
Escape codes:
 Numeric   : { "\x12": 8-bit, "\u1234": 16-bit, "\U00102030": 32-bit }
 Protective: { "\\": '\', "\"": '"', "\ ": ' ', "\<TAB>": TAB }
 C         : { "\0": NUL, "\a": BEL, "\b": BS, "\f": FF, "\n": LF, "\r": CR,
               "\t": TAB, "\v": VTAB }
 Additional: { "\e": ESC, "\_": NBSP, "\N": NEL, "\L": LS, "\P": PS }
...
```



### 三、默认全局配置文件值的注入

默认全局配置文件值的注入, 这里一定是默认的全局配置文件, 如果是自定义的配置文件, 则需要第四章中的注解.

#### 方式 1

第一种实现配置文件值的注入的方式, 主需要两个注解

- @Component

  将当前类加载到容器中, 将其交给容器管理其实例化, 才可以使用容器提供的其他注解功能, 如@ConfigurationProperties等

- @ConfigurationProperties(prefix = "xxx")

  将 spring boot 配置文件中前缀为 "xxx" 下配置的属性值和本类的字段映射绑定

##### 示例

1. 配置文件

   ```yaml
   person:
     name: Lily
     age: 12
     gender: female
     birth: 2008/01/01 12:12:12
     maps: {k1: v1, k2: v2}
     pets: [Ragdoll, Cibotium]
   ```

2. JavaBean

   ```java
   package cgs.springdata.bean;
   
   import lombok.Getter;
   import lombok.Setter;
   import lombok.ToString;
   import org.springframework.boot.context.properties.ConfigurationProperties;
   import org.springframework.stereotype.Component;
   
   import java.util.Date;
   import java.util.List;
   import java.util.Map;
   
   /**
    * @Description
    *      将 spring boot 配置文件中配置的属性值, 映射到当前 Java bean 对象中
    *      @ConfigurationProperties(prefix = "person")
    *          将 spring boot 配置文件中前缀为 "person" 下配置的属性值和本类的字段映射绑定
    *      @Component
    *          将当前类加载到容器中, 将其交给容器管理其实例化, 才可以使用 @ConfigurationProperties 这个功能
    * @Author sherlock
    * @Date
    */
   
   @Getter
   @Setter
   @ToString
   @Component
   @ConfigurationProperties(prefix = "person")
   public class Person {
   
       private String name;
       private int age;
       private String gender;
       private Date birth;
   
       private Map<String, Object> maps;
       private List<String> pets;
   }
   ```

3. 在依赖中添加配置文件处理器

   ```xml
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-configuration-processor</artifactId>
     <optional>true</optional>
   </dependency>
   ```

#### 方式 2

第一种实现配置文件值的注入的方式, 也主需要两个注解

1. @Component

   将当前类加载到容器中, 将其交给容器管理其实例化, 才可以使用容器提供的其他注解功能, 如@ConfigurationProperties, @Value等

2. @Value

   它直接标注在字段属性上, 功能类似以 Spring 中的 bean标签的语法

   ```xml
   <bean>
     <property name="age" value="字面量/${key}/#{SpEl}"></property>
   </bean>
   ```

##### 示例

1. 配置文件

   ```properties
   management.endpoints.web.exposure.include=*
           
   person.name=Tom
   person.gender=male
   person.pets=dog,cat
   ```

2. JavaBean

   ```java
   package cgs.springdata.bean;
   
   import lombok.Getter;
   import lombok.Setter;
   import lombok.ToString;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.boot.context.properties.ConfigurationProperties;
   import org.springframework.stereotype.Component;
   
   import java.util.Date;
   import java.util.List;
   import java.util.Map;
   
   @Getter
   @Setter
   @ToString
   @Component
   public class Person {
   
       @Value("${person.name}")
       private String name;
       @Value("#{10*1.8}")
       private int age;
       @Value("${person.gender}")
       private String gender;
       private Date birth;
       private Map<String, Object> maps;
       @Value("${person.pets}")
       private List<String> pets;
   }
   
   /**
   	测试结果:
   	Person: Person(name=Tom, age=18, gender=male, birth=null, maps=null, pets=[dog, cat])
    */
   ```

   

#### @ConfigurationProperties和@Value比较

|                            | @ConfigurationProperties                  | @Value                   |
| -------------------------- | ----------------------------------------- | ------------------------ |
| 功能                       | 指定 prefix 后可以批量注入配置中的属性    | 必须单个单个的字段来指定 |
| 松散绑定                   | 支持, 松散语法(驼峰的大写字母= -小写字母) | 不支持                   |
| SpEl                       | 不支持                                    | 支持                     |
| JSR303数据校验(@Validated) | 支持                                      | 不支持                   |

#### 数据校验

仅仅在使用 @ConfigurationProperties 时支持 @Validated 语法校验

```java
package cgs.springdata.bean;

import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import org.hibernate.validator.constraints.Email;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;

import java.util.Date;
import java.util.List;
import java.util.Map;


@Getter
@Setter
@ToString
@Component
@ConfigurationProperties(prefix = "person")
@Validated
public class Person {

    @Email
    private String name;
    private int age;
    private String gender;
    private Date birth;
    private Map<String, Object> maps;
    private List<String> pets;
}

/**
	运行报错, 因为 name 的值不符合 email 地址规范
 */

```



### 四、自定义配置文件值的注入

spring boot 中默认配置文件是 application.yml/application.properties.

但是如果在开发中我们自己定义了一个 db.properties, 并且需要把值注入到字段中, 这时就需要使用注解 ***@PropertySource*** 和 ***@Value***

#### 4.1 @PropertySource

配置文件db.properties

```properties
db.name=cy
db.password=1234qwer
db.driver=com.mysql.jdbc.Driver
```

JavaBean

```java
package cgs.springdata.bean;

import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.PropertySource;
import org.springframework.stereotype.Component;

/**
 * @Description TODO
 * @Author sherlock
 * @Date
 */

@Getter
@Setter
@ToString
@Component
@PropertySource(value = {"classpath:db.properties"})
public class Db {

    @Value("${db.name}")
    private String name;
    @Value("${db.password}")
    private String password;
    @Value("${db.driver}")
    private String driver;
}

/**
	测试结果:
  	Db(name=cy, password=1234qwer, driver=com.mysql.jdbc.Driver)
 */
```



### 五、自定义 Spring 配置文件的引入

如果要在 Spring boot 中使用一个自定义的 spring 的 xml 配置文件, 需要在启动类(配置类)上使用注解 ***@ImportResource(locations = {"classpath:xxx.xml"})***

#### 5.1 @ImportResource

启动类(主配置类)

```java
package cgs.springdata;

import cgs.springdata.bean.Person;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ImportResource;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

/**
 * @Description TODO
 * @Author sherlock
 * @Date
 */

@Slf4j
@SpringBootApplication
@ImportResource(locations = {"classpath:beans.xml"})
public class DataSourceDemoApplicationBackup implements CommandLineRunner {

    @Autowired
    private DataSource dataSource;
    @Autowired
    private Person person;

    @Override
    public void run(String... args) throws Exception {
//        showConnection();
        printPerson();
    }

    private void showConnection() throws SQLException {
        log.info(String.format("### DataSource: %s\n", dataSource.toString()));
        Connection connection = dataSource.getConnection();
        log.info(String.format("### H2 Connection: %s\n", connection.toString()));
        connection.close();
    }

    private void printPerson() {
        log.info(String.format("### Person: %s\n", person.toString()));
    }

    public static void main(String[] args) {

        SpringApplication.run(DataSourceDemoApplicationBackup.class, args);
    }

}
```

beans.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="helloService" class="cgs.springdata.service.HelloService">
    </bean>

</beans>
```

测试

```java
package cgs.test;

import cgs.springdata.DataSourceDemoApplicationBackup;
import cgs.springdata.bean.Db;
import cgs.springdata.bean.Person;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.ApplicationContext;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * @Description TODO
 * @Author sherlock
 * @Date
 */

@RunWith(SpringRunner.class)
@SpringBootTest(classes = {DataSourceDemoApplicationBackup.class})
public class SpringBootApplicationTest {

    @Autowired
    private Person person;
    @Autowired
    private Db db;
    @Autowired
    private ApplicationContext ioc;

    @Test
    public void contextLoad() {
        System.out.println(person.toString());
    }

    @Test
    public void contextLoad2() {
        System.out.println(db.toString());
    }

  	/**
  	 * 测试结果为 true
  	 */
    @Test
    public void loadHelloService() {
        System.out.println(ioc.containsBean("helloService"));
    }
}
```



### 六、自定义配置类的引入

Spring Boot 不推荐使用xml配置文件, 它建议我们使用全注解的方式来配置 Spring, 所以在对于上面的 beans.xml 这种方式并不推荐, 我们可以自定义一个配置类然后在配置类中注册 HelloService 字段.