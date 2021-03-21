## 【SpringBoot】multi-modules 中指定自己的 parent 模块

首先, 我们在多模块开发中一般有以下两种来组织 parent 模块的位置

1. ***项目根目录为 parent 模块***

   ```
   project
   	--| moduleA
   		--| pom.xml
   	--| moduleB
   		--| pom.xml
   		
   	-- pom.xml
   ```

   对于这种组织形式, moduleA, moduleB 都是普通业务功能模块, 而 project 既是项目根目录, 也是 parent 模块, 对于这种组织形式, 可以在moduleA, moduleB 这种模块中使用自己指定的 parent 模块, 而不必定使用 springboot 的 parent 模块 ***spring-boot-starter-parent***

   这时, 我们可以使用如下配置来配置项目的 pom.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <groupId>com.github.sherlock</groupId>
       <artifactId>project</artifactId>
       <version>1.0-SNAPSHOT</version>
       <modules>
           <module>moduleA</module>
           <module>moduleB</module>
       </modules>
       <packaging>pom</packaging>
   
     	<properties>
           <java.version>1.8</java.version>
         	<springboot.version>2.1.6.RELEASE</springboot.version>
       	  <lombok.version>1.16.20</lombok.version>
           <joda-time.version>2.10</joda-time.version>
           <commons-lang3.version>3.8.1</commons-lang3.version>
         	<!-- 省略... -->
       </properties>
     
     	<dependencyManagement>
           <dependencies>
               <dependency>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-dependencies</artifactId>
                   <version>${springboot.version}</version>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>
             	
             	<!-- 省略其他 dependency -->
         	</dependencies>
         </dependencyManagement>
   
   
   </project>
   ```

   而对于普通 业务功能模块的 pom.xml 可以这么配置

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <parent>
           <artifactId>project</artifactId>
           <groupId>com.github.sherlock</groupId>
           <version>1.0-SNAPSHOT</version>
           <relativePath/>
       </parent>
       <modelVersion>4.0.0</modelVersion>
   
       <artifactId>moduleA</artifactId>
       <packaging>jar</packaging>
   
       <dependencies>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
               <scope>provided</scope>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-configuration-processor</artifactId>
               <optional>true</optional>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
           </dependency>
       </dependencies>
   
       <build>
           <plugins>
               <plugin>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-maven-plugin</artifactId>
                   <configuration>
                       <mainClass>${your_main_class_name}</mainClass>
                   </configuration>
                   <executions>
                       <execution>
                           <goals>
                               <goal>repackage</goal>
                           </goals>
                       </execution>
                   </executions>
               </plugin>
           </plugins>
       </build>
   
   </project>
   ```

   

2. ***项目子模块为 parent 模块***

   ```
   project
   	--| parent
   		--| pom.xml
   	--| moduleA
   		--| pom.xml
   	--| moduleB
   		--| pom.xml
   		
   	-- pom.xml
   ```

   对于这种组织形式, 目前暂未找到解决方法, 如果在 parernt 模块中使用上面的项目根目录的 pom.xml 配置是没问题的.

   但是在 业务功能模块 moduleA, moduleB 中配置 pom.xml时 需要指定  ***\<relativePath>../parent/pom.xml\</relativePath>*** 这个配置会报错, 显示无法识别这个路径, 但是如果项目不是 springboot 项目, 普通多模块项目,这样做是完全没问题的. 暂未知道问题出在哪

###  