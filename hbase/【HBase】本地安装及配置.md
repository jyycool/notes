# 【HBase】本地安装及配置

前提安装配置好 MySQL, Hadoop, Hive, ZK, 并测试各服务正常.



## 准备

1. 解压 tar包,建立软连接, 配置环境变量

   ```sh
   # 1. 解压,建立软连接
   ➜  ~ tar -zxvf ~/Downloads/softpack/apache soft/hbase/hbase-2.2.6-bin.tar.gz -C ~/devTools/opt
   ➜  ~ sudo ln -s ~/devTools/opt/hbase-2.2.6 /usr/local/hbase
   
   # 2. 配置环境变量(修改 ~/.bash_profile, 添加如下内容)
   export HBASE_HOME=/usr/local/hbase
   export PATH=$HBASE_HOME/bin:$PATH
   ```



## 配置文件

1. ***hive-env.sh***

   ```sh
   export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
   export HBASE_MANAGES_ZK=false
   ```

   

2. ***hive-site.xml***

   ```xml
   	<property>
       <name>hbase.rootdir</name>
       <value>hdfs://localhost:9000/hbase2</value>
     </property>
   
     <property>
       <name>hbase.cluster.distributed</name>
       <value>true</value>
     </property>
   
     <property>
       <name>hbase.zookeeper.quorum</name>
       <value>localhost</value>
     </property>
   
     <property>
       <name>hbase.zookeeper.property.datadir</name>
       <value>/Users/sherlock/devTools/opt/zookeeper-3.5.8/data</value>
     </property>
   
     <property>
       <name>hbase.tmp.dir</name>
       <value>./tmp</value>
     </property>
   
     <property>
       <name>hbase.unsafe.stream.capability.enforce</name>
       <value>false</value>
     </property>
   
   	<!-- hbase在zk上默认的根目录 -->  
     <property>  
       <name>zookeeper.znode.parent</name>  
       <value>/hbase-2</value>  
     </property>
   ```

3. ***regionservers***

   ```sh
   localhost
   ```

   

4. 将 hadoop 配置文件软连接到 hbase/conf 下

   ```sh
   ➜  local ln -s /Users/sherlock/devTools/opt/hadoop-2.7.7/etc/hadoop/core-site.xml /Users/sherlock/devTools/opt/hive-2.3.7/conf/core-site.xml
   
   ➜  local ln -s /Users/sherlock/devTools/opt/hadoop-2.7.7/etc/hadoop/hdfs-site.xml /Users/sherlock/devTools/opt/hive-2.3.7/conf/hdfs-site.xml
   ```

   

## 服务测试

启动 hbase 服务

```sh
# 确认已经启动 hadoop 和 zk
➜  local start-hbase.sh
```

