## Maven插件

### plugin

maven中的各个插件放在标签<build>的<plugins>中

示例;

```xml
 <!--引用的默认插件信息。该插件配置项直到被引用时才会被解析或绑定到生命周期。给定插件的任何本地配置都会覆盖这里的配置-->
 <buid>
	 <!--使用的插件列表 。-->
   <plugins>
   	<!--plugin元素包含描述插件所需要的信息。-->
    <plugin>
     <!--插件在仓库里的坐标,及版本号-->
     <groupId/>
     <artifactId/>
     <version/>
     <!--是否从该插件下载Maven扩展（例如打包和类型处理器），由于性能原因，只有在真需要下载时，该元素才被设置成enabled。-->
     <extensions/>
     <!--在构建生命周期中执行一组目标的配置。每个目标可能有不同的配置。-->
     <executions>
      <!--execution元素包含了插件执行需要的信息-->
      <execution>
       <!--执行目标的标识符，用于标识构建过程中的目标，或者匹配继承过程中需要合并的执行目标-->
       <id/>
       <!--绑定了目标的构建生命周期阶段，如果省略，目标会被绑定到源数据里配置的默认阶段-->
       <phase/>
       <!--配置的执行目标-->
       <goals/>
       <!--配置是否被传播到子POM-->
       <inherited/>
       <!--作为DOM对象的配置-->
       <configuration/>
      </execution>
     </executions>
     <!--项目引入插件所需要的额外依赖-->
     <dependencies>
      <!--参见dependencies/dependency元素-->
      <dependency>
       ......
      </dependency>
     </dependencies>     
     <!--任何配置是否被传播到子项目-->
     <inherited/>
     <!--作为DOM对象的配置-->
     <configuration/>
    </plugin>
   </plugins>
 </buid>
```



### 常用插件

#### Java编译插件

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.6.0</version>
  <configuration>                                                                                                                           
    <!-- 一般而言，target与source是保持一致的，但是，有时候为了让程序能在其他版本的jdk中运行(对于低版本目标jdk，源代码中不能使用低版本jdk中不支持的语法)，会存在target不同于source的情况 -->                    
  	<source>1.8</source> <!-- 源代码使用的JDK版本 -->                                                                                             
    <target>1.8</target> <!-- 需要生成的目标class文件的编译版本 -->                                                                                     
    <encoding>UTF-8</encoding><!-- 字符集编码 -->
    <skipTests>true</skipTests><!-- 跳过测试 -->                                                                             
    <verbose>true</verbose>
    <showWarnings>true</showWarnings>                                                                                                               
    <fork>true</fork><!-- 要使compilerVersion标签生效，还需要将fork设为true，用于明确表示编译版本配置的可用 -->                                                        
    <executable><!-- path-to-javac --></executable><!-- 使用指定的javac命令，例如：<executable>${JAVA_1_4_HOME}/bin/javac</executable> -->           
    <compilerVersion>1.3</compilerVersion><!-- 指定插件将使用的编译器的版本 -->                                                                         
    <meminitial>128m</meminitial><!-- 编译器使用的初始内存 -->                                                                                      
    <maxmem>512m</maxmem><!-- 编译器使用的最大内存 -->                                                                                              
    <compilerArgument>-verbose -bootclasspath ${java.home}\lib\rt.jar</compilerArgument><!-- 这个选项用来传递编译器自身不包含但是却支持的参数选项 -->               
    </configuration>    
</plugin>
```

#### maven默认打包插件

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <version>3.0.2</version>
    <executions>
      <execution>
      <id>copy-dependencies</id>
      <phase>package</phase>
      <goals>   
      	<goal>copy-dependencies</goal>
      </goals>
      <configuration>
  			<outputDirectory>${project.build.directory}/lib</outputDirectory>
      	<overWriteReleases>false</overWriteReleases>
      	<overWriteSnapshots>false</overWriteSnapshots>
      	<overWriteIfNewer>true</overWriteIfNewer>
      </configuration>
  	</execution>
	</executions>
</plugin>

```

这个可以将所有依赖放到target下lib目录里, 但是项目jar不是fat jar!!!!

### 特定插件

#### Scala编译插件

当我们使用Scala和Java语言混合编程时可能出现的问题: 

- java代码中引用了 scala 的代码, 运行不报错, 打包会找不到scala定义的包
- java代码放在 scala包里面，编译时会直接忽略这个类，不参与编译。



> 引入scala编译插件

```xml
<plugin>
	<groupId>net.alchim31.maven</groupId>
	<artifactId>scala-maven-plugin</artifactId>
	<version>3.2.2</version>
	<executions>
    <execution>
      <id>scala-compile-first</id>
      <phase>process-resources</phase>
      <goals>
      	<goal>add-source</goal>
      	<goal>compile</goal>
      </goals>
    </execution>

    <execution>
    	<phase>compile</phase>
    	<goals>
    		<goal>compile</goal>
    		<goal>testCompile</goal>
    	</goals>
    </execution>
	</executions>
	<configuration>
		<scalaVersion>${scala.version}</scalaVersion>
	</configuration>
</plugin>
```

#### assembly打包插件

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
  <version></version>
	<configuration>
    <appendAssemblyId>false</appendAssemblyId>
    <descriptorRefs>
      <descriptorRef>jar-with-dependencies</descriptorRef>
    </descriptorRefs>
    <archive>
      <manifest>
      <!-- 此处指定main方法入口的class -->
        <mainClass>com.panda521.SpringPracServerApplication</mainClass>
      </manifest>
    </archive>
	</configuration>
	<executions>
    <execution>
      <id>make-assembly</id>
      <phase>package</phase>
      <goals>
      	<goal>single</goal>
      </goals>
    </execution>
	</executions>
</plugin>
```

使用 `maven assembly:assembly`进行打包操作

#### shade打包插件

通过 maven-shade-plugin 生成一个 ***uber-jar(super-jar/fat-jar)***，它包含所有的依赖 jar 包。

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>3.0.0</version>
  <executions>
    <!-- Run shade goal on package phase -->
    <execution>
      <phase>package</phase>
      <goals>
        <goal>shade</goal>
      </goals>
      <configuration>
        <artifactSet>
          <excludes>
            <exclude>org.apache.flink:force-shading</exclude>
            <exclude>com.google.code.findbugs:jsr305</exclude>
            <exclude>org.slf4j:*</exclude>
            <exclude>log4j:*</exclude>
          </excludes>
        </artifactSet>
        <filters>
          <filter>
            <!-- Do not copy the signatures in the META-INF folder.
         Otherwise, this might cause SecurityExceptions when using the JAR. -->
            <artifact>*:*</artifact>
            <excludes>
              <exclude>META-INF/*.SF</exclude>
              <exclude>META-INF/*.DSA</exclude>
              <exclude>META-INF/*.RSA</exclude>
            </excludes>
          </filter>
        </filters>
        <transformers>
          <!-- 指定 main class -->
          <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
            <mainClass></mainClass>
          </transformer>
          <!-- 合并 akka的配置文件 reference.conf -->
          <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
          	<resource>reference.conf</resource>
        	</transformer>
        </transformers>
      </configuration>
    </execution>
  </executions>
</plugin>
```

