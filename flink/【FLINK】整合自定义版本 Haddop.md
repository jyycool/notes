## 【FLINK】整合自定义版本 Hadoop

 Flink1.10一个划时代的版本，它标志着对 Blink[1] 的整合宣告完成。而且随着对 Hive 的生产级别集成及对 TPC-DS 的全面覆盖，Flink 在增强流式 SQL 处理能力的同时也具备了成熟的批处理能力。

       众所周知，Apache Flink官网下载安装包不能支持CDH，需要编译后进行安装，参照网上很多资料，尝试了多天，终于成功，供大家参考。
### 一、环境准备

1. 环境：Jdk 1.8、Maven 3.5.4 和 Scala-2.11

2. 源码和CDH 版本：flink 1.10.1 、 hadoop-2.6.0-cdh5.7.0

   

### 二、安装包准备



1. 不同的 Flink 版本使用的 Flink-shaded不同，1.10 版本使用 10.0：

    https://mirrors.tuna.tsinghua.edu.cn/apache/flink/flink-shaded-10.0/flink-shaded-10.0-src.tgz

2. flink1.10.1 tar包：

    https://mirrors.tuna.tsinghua.edu.cn/apache/flink/flink-1.10.0/flink-1.10.1-src.tgz


### 三、编译flink1.10.1源码

1. 解压tar包;

   ***tar -zxvf flink-1.10.1-src.tgz***

2. 编译

   ***cd flink-1.10.1/***

   ***mvn clean install -DskipTests -Dfast -Drat.skip=true -Dhaoop.version=2.6.0-cdh5.7.0 -Pvendor-repos -Dinclude-hadoop -Dscala-2.11 -T2C***

   > Tip: 注意修改成自己对应的版本
   >
   > 注：hadoop版本，cdh版本，scala版本根据自己的集群情况自行修改。
   >
   > 参数的含义：
   >
   > 1. ***-Dfast***  #在flink根目录下pom.xml文件中fast配置项目中含快速设置,其中包含了多项构建时的跳过参数. #例如apache的文件头(rat)合法校验，代码风格检查，javadoc生成的跳过等，详细可阅读pom.xml
   > 2. ***install*** maven的安装命令
   > 3. ***-T2C*** #支持多处理器或者处理器核数参数,加快构建速度,推荐Maven3.3及以上
   > 4. ***-Pinclude-hadoop***  将 hadoop的 jar包，打入到lib/中
   > 5. ***-Pvendor-repos***   # 使用cdh、hdp 的hadoop 需要添加该参数
   > 6. ***-Dscala-2.11***     # 指定scala的版本为2.11
   > 7. ***-Dhadoop.version=2.6.0-cdh5.7.0***  指定 hadoop 的版本

3. 编译报错:

   

   

   原因官网上已经说得很明白。

   > If the used Hadoop version is not listed on the download page (possibly due to being a Vendor-specific version), then it is necessary to build flink-shaded against this version. You can find the source code for this project in the Additional Components section of the download page.
   >
   > Flink最近的版本呢，不像1.7版本一样，有编译好的指定的hadoop 版本官网上可供下载，对于供应商的hadoop版本的编译，需要先编译安装flink-shaded这个项目。

   

​	





### 三、编译对应的flink-shaded 版本

1. 解压tar包

   ***tar -zxvf flink-shaded-10.0-src.tgz***

2. 修改pom.xml

   ***vim flink-shaded-10.0/pom.xml***

   添加如下配置：

   ```xml
   <profile>
       <id>vendor-repos</id>
       <activation>
           <property>
               <name>vendor-repos</name>
           </property>
       </activation>
    
       <!-- Add vendor maven repositories -->
       <repositories>
           <!-- Cloudera -->
           <repository>
               <id>cloudera-releases</id>
               <url>https://repository.cloudera.com/artifactory/cloudera-repos</url>
               <releases>
                   <enabled>true</enabled>
               </releases>
               <snapshots>
                   <enabled>false</enabled>
               </snapshots>
           </repository>
           <!-- Hortonworks -->
           <repository>
               <id>HDPReleases</id>
               <name>HDP Releases</name>
               <url>https://repo.hortonworks.com/content/repositories/releases/</url>
               <snapshots><enabled>false</enabled></snapshots>
               <releases><enabled>true</enabled></releases>
           </repository>
           <repository>
               <id>HortonworksJettyHadoop</id>
               <name>HDP Jetty</name>
               <url>https://repo.hortonworks.com/content/repositories/jetty-hadoop</url>
               <snapshots><enabled>false</enabled></snapshots>
               <releases><enabled>true</enabled></releases>
           </repository>
           <!-- MapR -->
           <repository>
               <id>mapr-releases</id>
               <url>https://repository.mapr.com/maven/</url>
               <snapshots><enabled>false</enabled></snapshots>
               <releases><enabled>true</enabled></releases>
           </repository>
       </repositories>
   </profile>
   ```

3. 编译：

   ***cd flink-shaded-10.0/***

   ***mvn -T2C clean install -DskipTests -Pvendor-repos -Dhadoop.version=2.6.0-cdh5.7.0 -Dscala-2.11 -Drat.skip=true***

   > 1. 



### 四、

测试

***./bin/flink run -m yarn-cluster -ynm test_wordcount ./examples/batch/WordCount.jar --input hdfs://cluster_name/tmp/words.txt***

> Tip：注意在hdfs上添加测试的words.txt