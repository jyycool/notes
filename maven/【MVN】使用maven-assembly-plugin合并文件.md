## 【MVN】使用maven-assembly-plugin合并文件

### 说明

assembly是个很好的打包工具，通俗讲，我们可以用它把七零八落的一些文件按一定规则重新组织，打成个大包。

例如，一个工程依赖了好多个jar包，我们可以用它把所有依赖的jar包里的class提取出来，重新组织成一个大jar包。

这时会遇到一个问题，如果原来jar里有同名文件，就会相互覆盖，最终导致打出的jar包不可用。

spring.handlers、spring.schemas就是个例子。

很多jar包对spring的配置做扩展，都会用这两个文件定义XML的schemas和解析器。如果要合并成一个jar包，正确的做法是把所有的spring.handlers、spring.schemas合并，而非相关覆盖。

### 配置

那么该如何做呢？

assembly提供了一个 ***containerDescriptorHandlers*** 配置可以解决这个问题。

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  ....
  <containerDescriptorHandlers>
    <containerDescriptorHandler>
      <handlerName>metaInf-spring</handlerName>
    </containerDescriptorHandler>
  </containerDescriptorHandlers>
</assembly>
```

如上，***metaInf-spring*** 会把所有 ***META-INF/spring.**** 合并成一个大文件。

那其他文件呢？例如多个***my.properties***

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  ....
  <containerDescriptorHandlers>
    <containerDescriptorHandler>
      <handlerName>file-aggregator</handlerName>
      <configuration>
        <filePattern>.*/my.properties</filePattern>
        <outputPath>my.properties</outputPath>
      </configuration>
    </containerDescriptorHandler>
  </containerDescriptorHandlers>
</assembly>
```

如上，可以用file-aggregator，通过表达式方式合并所有my.properties为一个大文件。

