# 【Hadoop】本地单节点伪分布安装

本地的 Hadoop 的 yarn又炸了, 好桑心, 又双要重装...

索性不用 CDH 版本的 Hadoop 包了 (因为 CDH 的包似乎只找到 Hadoop-2.6.0的), 这次装了较新的 Hadoop-2.7.7

下载 Hadoop-2.7.7.tar.gz

```sh
wget http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-2.7.7/hadoop-2.7.7.tar.gz
```

有个小插曲: wget 又炸了

```sh
dyld: Library not loaded: /usr/local/opt/openssl/lib/libssl.1.0.0.dylib
  Referenced from: /usr/local/bin/wget
  Reason: image not found
[1]    73150 abort      wget
```

所以又重新装了下 wget

```sh
➜  Downloads brew uninstall --ignore-dependencies gettext
...此处省略一大堆安装情况...
➜  Downloads brew install gettext
...此处省略一大堆安装情况...
==> Checking for dependents of upgraded formulae...
==> No broken dependents to reinstall!
```

然后就愉快的下载 Hadoop-2.7.7.tar.gz 啦......

## 准备

1. 解压 tar 包

   ```sh
   tar -zxvf ~/Downloads/hadoop-2.7.7.tar.gz -C ~/devTools/opt
   ```

   ***${HADOOP_HOME} = /Users/sherlock/devTools/opt/hadoop-2.9.2***

2. 删除 docs文件夹 (基本无用)

   本地 Mac 存储空间有限,  删除 share/docs 文件夹, 节省一点空间

   ```shell
   ➜  ~ cd ~/devTools/opt/hadoop-2.9.2
   ➜  hadoop-2.9.2 rm -rf share/doc
   ```

3. 环境配置

   - ***hadoop-env.sh, mapred-env.sh, yarn-env.sh***  中配置本地 JAVA_HOME

     ***export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home***

     

   - ***~/.bash_profile*** 中配置全局环境变量

     ***export HADOOP_HOME=/Users/sherlock/devTools/cdh5.7.0/hadoop-2.6.0-cdh5.7.0***
     ***export PATH=\$PATH:$HADOOP_HOME/bin***
     ***export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native***
     ***export HADOOP_OPTS="-Djava.library.path=\$HADOOP_HOME/lib:$HADOOP_COMMON_LIB_NATIVE_DIR"***
     ***export HADOOP_CONF_DIR=/Users/sherlock/devTools/opt/hadoop-2.7.7-cdh5.7.0/etc/hadoop***



## 配置HDFS

### core-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>

    <!-- NameNode服务地址, 默认服务端口8020，HA模式下服务端口9000 -->
	<property>
	  <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
    
    <!--   HDFS的元数据在namenode节点的存放路径 -->
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/Users/sherlock/devTools/opt/hadoop-2.7.7/metadata</value>
	</property>
  
  <property>
    <name>hadoop.proxyuser.root.hosts</name>
    <value>*</value>
  </property>
  <property>
    <name>hadoop.proxyuser.root.groups</name>
    <value>*</value>
  </property>
    
</configuration>
```

### hdfs-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>

    <!--  hdfs上的文件的副本数，伪分布式配置为1 -->
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>

    <!-- 关闭权限检查用户或用户组 -->
    <property>
        <name>dfs.permissions.enabled</name>
        <value>false</value>
    </property>
    
</configuration>
```



### 格式化 namenode 节点

> 每次格式化之前需要确保已经删除元数据保存目录下的所有文件
>
> Tips: 元数据保存路径配置在 ***hdfs-site.xml配置文件中的dfs.namenode.name.dir属性值, 默认是core-site.xml 里的 ${hadoop.tmp.dir}***

```shell
➜  hadoop-2.7.7 bin/hdfs namenode -format
20/06/24 07:12:34 INFO namenode.NameNode: STARTUP_MSG:
/************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = sherlock-MBP/192.168.31.108
STARTUP_MSG:   args = [-format]
STARTUP_MSG:   version = 2.7.7
STARTUP_MSG:   classpath = /Users/sherlock/devTools/opt/hadoop-2.7.7
.....此处省略粘贴了好多个jar包.....
20/06/24 07:12:36 INFO namenode.FSImage: Allocated new BlockPoolId: BP-131590295-192.168.31.108-1592953956182
20/06/24 07:12:36 INFO common.Storage: Storage directory /Users/sherlock/devTools/opt/hadoop-2.7.7/metadata/dfs/name has been successfully formatted.
20/06/24 07:12:36 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
20/06/24 07:12:36 INFO util.ExitUtil: Exiting with status 0
20/06/24 07:12:36 INFO namenode.NameNode: SHUTDOWN_MSG:
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at sherlock-MBP/192.168.31.108
************************************************************/
```

控制台打印了这句话

***20/06/24 07:12:36 INFO common.Storage: Storage directory /Users/sherlock/devTools/cdh5.7.0/hadoop-2.6.0-cdh5.7.0/data/dfs/name has been successfully formatted.*** 说明格式化成功

##  配置 YARN

本地模式, yarn真的很吃配置, 难过......

### yarn-site.xml

```xml
<?xml version="1.0"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<configuration>

    <!-- 指定ResorceManager所在服务器的主机名 -->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>localhost</value>
    </property>

    <!-- 指明在执行MapReduce的时候使用shuffle -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

</configuration>
```

### mapred-site.xml

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
    
    <!-- 指定MapReduce基于Yarn来运行 -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

</configuration>
```



## 配置日志聚合

### mapred-site.xml

<configuration>标签中追加内容

```xml
<!--指定jobhistory服务的主机及RPC端口号-->
<property>
	<name>mapreduce.jobhistory.address</name>
	<!--配置实际的主机名和端口-->
	<value>localhost:10020</value>
</property>

<!--  指定jobhistory服务的web访问的主机及RPC端口号  -->
<property>
	<name>mapreduce.jobhistory.webapp.address</name>
	<value>localhost:19888</value>
</property>
```

### yarn-site.xml

<configuration>标签中追加内容

```xml
<!-- 开启yarn日志聚合-->
<property>
	<name>yarn.log-aggregation-enable</name>
	<value>true</value>
</property>
<!--  日志保存时间7天  -->
<property>
	<name>yarn.log-aggregation.retain-seconds</name>
	<value>604800</value>
</property>
```

## 服务测试

1. 启动进程 hdfs 和 yarn

   ```shell
   sbin/hadoop-daemon.sh start namenode
   sbin/hadoop-daemon.sh start datanode
   sbin/yarn-daemon.sh start resourcemanager
   sbin/yarn-daemon.sh start nodemanager
   ```

2. 启动历史服务进程

   ```sh
   sbin/mr-jobhistory-daemon.sh start historyserver
   ```

3. 停止所有进程

   ```sh
   sbin/yarn-daemon.sh stop nodemanager
   sbin/yarn-daemon.sh stop resourcemanager
   sbin/hadoop-daemon.sh stop datanode
   sbin/hadoop-daemon.sh stop namenode
   sbin/mr-jobhistory-daemon.sh stop historyserver
   ```

   ​				

## Hadoop 中常用端口

1. namenode 的web端口: 50070
2. namenode 服务的端口: 8020/9000