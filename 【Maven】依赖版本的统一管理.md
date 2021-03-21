## 【Maven】依赖版本的统一管理

#maven

### Springboot

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.1.6.RELEASE</version>
</parent>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```



### 常用工具类

1. ***junit-4.12***
2. ***lombok-1.16.20***
3. ***joda-time-2.10***
4. ***slf4j-1.7.25***
5. ***log4j-1.2.17***
6. ***fastjson-1.2.75***
7. ***guava-21.0***
8. ***commons-lang-3.8.1***
9. ***commons-pool-2.6.2***
10. ***commons-configuration-2.1.1***
11. ***commons-cli-1.4***
12. ***commons-collections-3.2.2***



#### maven

```xml
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  <java.version>1.8</java.version>
  <scala.binary>2.11</scala.binary>
  <scalatest.version>3.1.1</scalatest.version>
  <enumeratum.version>1.5.15</enumeratum.version>
  <junit.version>4.12</junit.version>
  <lombok.version>1.16.20</lombok.version>
  <joda-time.version>2.10</joda-time.version>
  <fastjson.version>1.2.68</fastjson.version>
  <slf4j.version>1.7.25</slf4j.version>
  <log4j.version>1.2.17</log4j.version>
  <guava.version>21.0</guava.version>
  <commons-lang.version>3.8.1</commons-lang.version>
  <commons-pool.version>2.6.2</commons-pool.version>
  <commons-configuration.version>2.1.1</commons-configuration.version>
  <commons-cli.version>1.4</commons-cli.version>
  <commons-collections.version>3.2.2</commons-collections.version>
  <commons-io.version>2.5</commons-io.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>${junit.version}</version>
      <scope>test</scope>
    </dependency>
    
    <dependency>
      <groupId>org.scalatest</groupId>
      <artifactId>scalatest_${scala.binary}</artifactId>
      <version>${scalatest.version}</version>
      <scope>test</scope>
    </dependency>
    
    <dependency>
      <groupId>com.beachape</groupId>
      <artifactId>enumeratum_${scala.binary}</artifactId>
      <version>${enumeratum.version}</version>
    </dependency>
    
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>${lombok.version}</version>
      <scope>provided</scope>
    </dependency>
    
     <dependency>
      <groupId>joda-time</groupId>
      <artifactId>joda-time</artifactId>
      <version>${joda-time.version}</version>
    </dependency>

    <!-- slf4j & log4j -->
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>${slf4j.version}</version>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
      <version>${slf4j.version}</version>
    </dependency>
    <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
      <version>${log4j.version}</version>
    </dependency>
    
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>fastjson</artifactId>
      <version>${fastjson.version}</version>
    </dependency>

    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>${guava.version}</version>
    </dependency>
    
    <!-- apache commons tool -->
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-lang3</artifactId>
      <version>${commons-lang.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-pool2</artifactId>
      <version>${commons-pool.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.commons</groupId>
      <artifactId>commons-configuration2</artifactId>
      <version>${commons-configuration2.version}</version>
    </dependency>
    <dependency>
        <groupId>commons-cli</groupId>
        <artifactId>commons-cli</artifactId>
        <version>${commons-cli.version}</version>
    </dependency>
		<dependency>
    		<groupId>commons-collections</groupId>
    		<artifactId>commons-collections</artifactId>
    		<version>${commons-collections.version}</version>
		</dependency>
    <dependency>
    	<groupId>commons-io</groupId>
    	<artifactId>commons-io</artifactId>
    	<version>${commons-io.version}</version>
		</dependency>

  </dependencies>
</dependencyManagement>
```



### Scala

1. ***Scala枚举 : enumeratum_2.11-1.5.15***

2. ***Scala测试: scalatest-3.1.1***

3. ***scala-async_2.11-0.9.7***

   

#### maven

```xml
<dependency>
  <groupId>com.beachape</groupId>
  <artifactId>enumeratum_2.11</artifactId>
  <version>1.5.15</version>
</dependency>

<dependency>
  <groupId>org.scalatest</groupId>
  <artifactId>scalatest_2.11</artifactId>
  <version>3.1.1</version>
  <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.scala-lang.modules</groupId>
    <artifactId>scala-async_2.11</artifactId>
    <version>0.9.7</version>
</dependency>
```

### Spark

1. ***spark-core_2.11-2.4.4***
2. ***spark-sql-2.11-2.4.4***
3. ***spark-repl_2.11-2.4.4***



#### maven

```xml
<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-core_2.11</artifactId>
  <version>2.4.4</version>
</dependency>

<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql_2.11</artifactId>
    <version>2.4.4</version>
</dependency>

<dependency>
  <groupId>org.apache.spark</groupId>
  <artifactId>spark-repl_2.11</artifactId>
  <version>2.4.4</version>
</dependency>
```



### Flink

> flink版本要大于1.8.x, 因为1.9 及以后版本 flink-table模块重构了, maven依赖不一样了, 具体参考mavenrepo官网, 这里用的 scala.binary版本默认为 2.11

1. ***flink-core-1.10.1***  
2. ***flink-clients_2.11-1.10.1***
3. ***flink-yarn_2.11-1.10.1***
4. ***flink-java-1.10.1***
5. ***flink-streaming-java_2.11-1.10.1***
6. ***flink-scala_2.11-1.10.1***
7. ***flink-streaming_2.11-1.10.1***
8. ***flink-table-api-scala_2.11-1.10.1***
9. ***flink-table-api-scala-bridge_2.11-1.10.1***
10. ***flink-table-api-java-1.10.1***
11. ***flink-table-api-java-bridge_2.11-1.10.1***
12. ***flink-table-planner-blink_2.11-1.10.1***
13. ***flink-table-planner_2.11-1.10.1***
14. ***flink-statebackend-rocksdb_2.11-1.10.1***
15. ***flink-cep_2.11-1.10.1***
16. ***flink-cep-scala_2.11-1.10.1***



#### maven

```xml
<properties>
  <scala.binary>2.11</scala.binary>
	<flink.version>1.10.1</flink.version>
</properties>

<dependencies>
  <!-- flink-core -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-core</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <!-- flink-clients -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-clients_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <!-- flink-yarn -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-yarn_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <!-- flink java api -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-java</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-streaming-java_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <!-- flink scala api -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-scala_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-streaming-scala_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <!-- scala table api & bridge -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-scala_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-scala-bridge_${scala.binary}</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
  </dependency>
  <!-- java table api & bridge -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-java</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-java-bridge_${scala.binary}</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
  </dependency>
  <!-- blink panner -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner-blink_${scala.binary}</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
  </dependency>
  <!-- flink planner -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner_${scala.binary}</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
  </dependency>
  <!-- flink-statebackend-rockdb -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-statebackend-rocksdb_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <!-- flink-connector-kafka -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.10_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <!-- flink cep -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-cep_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-cep-scala_${scala.binary}</artifactId>
    <version>${flink.version}</version>
  </dependency>
</dependencies>
```



### Calcite

1. ***calcite-1.18.0***

#### maven

```xml
<dependency>
  <groupId>org.apache.calcite</groupId>
  <artifactId>calcite-core</artifactId>
  <version>1.18.0</version>
</dependency>
```



### Netty

1. ***netty-all-4.1.42.Final***

#### maven

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.42.Final</version>
</dependency>
```



### Akka

1. ***akka-actor_2.11-2.5.31***
2. ***akka-remote_2.11-2.5.31***

#### maven

```xml
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-actor_2.11</artifactId>
    <version>2.5.31</version>
</dependency>
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-remote_2.11</artifactId>
    <version>2.5.31</version>
</dependency>
```

### Boilerpipe

1. ***boilerpipe-1.2.2***



#### maven

```xml
<dependency>
    <groupId>com.syncthemall</groupId>
    <artifactId>boilerpipe</artifactId>
    <version>1.2.2</version>
</dependency>
```



### Zookeeper

1. ***zkclient-0.11***

#### maven

```xml
<!-- https://mvnrepository.com/artifact/com.101tec/zkclient -->
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.11</version>
</dependency>
```

### Curator

1. ***curator-framework-2.13.0***

   

2. ***curator-recipes-2.13.0***



i

### Json4s

1. ***json4s-native_2.11-3.6.9***
2. ***json4s-jackson_2.11-3.6.9***

#### maven

```xml
<dependency>
  <groupId>org.json4s</groupId>
  <artifactId>json4s-native_2.11</artifactId>
  <version>3.6.9</version>
</dependency>
<dependency>
  <groupId>org.json4s</groupId>
  <artifactId>json4s-jackson_2.11</artifactId>
  <version>3.6.9</version>
</dependency>
```



### Jackson

1. ***jackson-core-2.9.10***

2. ***jackson-databind-2.9.10.4***

3. ***jackson-module-scala_2.11-2.9.10***

4. ***jackson-annotations-2.9.10***

   

#### maven

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.9.10</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.10.4</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.module</groupId>
    <artifactId>jackson-module-scala_2.11</artifactId>
    <version>2.9.10</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
    <version>2.9.10</version>
</dependency>
```

