## 【AKKA】reference.conf 文件合并

当使用 Maven 程序集插件而不是 SBT 将 Akka 应用程序打包为具有所有依赖项的 FAT JAR时，我遇到了这样的的问题。

> 当我在 IDEA 运行一个 Akka 程序的时候, 一切都是正常的, 但是，当我将应用程序作为 jar 运行时，突然所有操作都失败了。并且报错: "***No configuration setting found for key 'akka.remote.artery'***"

### 问题分析

潜在的问题在于，当打包 FAT JAR时，默认情况下，构建系统会覆盖位于同一路径（例如`/reference.conf`）的文件。因此，当使用多个Akka模块时，一个模块`reference.conf`将最终覆盖所有其他模块，因此`reference.conf`在JAR文件中最终将只有一个模块，而不是多个模块`reference.conf`，然后在加载它们时由配置库合并。

这会导致找不到配置错误，因为`reference.conf`缺少某些模块的默认配置设置（在被覆盖的模块中找不到）。

### 解决

> 需要我们使用 ***管理工具(SBT/MAVEN......)*** 来合并资源文件

1. SBT

   ```java
   assemblyMergeStrategy in assembly := {
      case PathList("META-INF", xs @ _*) => MergeStrategy.discard
      case "reference.conf" => MergeStrategy.concat
      case x => MergeStrategy.first
   }
   ```

2. MAVEN-ASSEMBLY

   ```xml
   <containerDescriptorHandlers>
   	<containerDescriptorHandler>
       <handlerName>file-aggregator</handlerName>
       <configuration>
       	<filePattern>.*/my.properties</filePattern>
         <outputPath>my.properties</outputPath>
     	</configuration>
   	</containerDescriptorHandler>
   </containerDescriptorHandlers>
   ```

3. MAVEN-SHADE

   ```xml
   <transformers>
   	<transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
   		<resource>reference.conf</resource>
   	</transformer>
   </transformers>
   ```

   

