

# 1.cloudera manager简单介绍

Cloudera Manager是一个拥有集群自动化安装、中心化管理、集群监控、报警功能的一个工具（软件）,使得安装集群从几天的时间缩短在几个小时内，运维人员从数十人降低到几人以内，极大的提高集群管理的效率。

安装完成的界面.如下图:

![1570799640007](assets/1570799640007.png)



# 2.cloudera manager主要核心功能

    • 管理：对集群进行管理，如添加、删除节点等操作。
    
    • 监控：监控集群的健康情况，对设置的各种指标和系统运行情况进行全面监控。
    
    • 诊断：对集群出现的问题进行诊断，对出现的问题给出建议解决方案。
    
    • 集成：多组件进行整合。
# 3.cloudera manager 的架构

![1574057384454](assets/1574057384454.png)

# 4.准备三台虚拟机

参考【0.大数据环境前置准备】

# 5.准备cloudera安装包

```shell
由于是离线部署，因此需要预先下载好需要的文件。
需要准备的文件有:

Cloudera Manager 5
文件名: cloudera-manager-centos7-cm5.14.0_x86_64.tar.gz
下载地址: https://archive.cloudera.com/cm5/cm/5/
CDH安装包（Parecls包）
版本号必须与Cloudera Manager相对应
下载地址: https://archive.cloudera.com/cdh5/parcels/5.14.0/
需要下载下面3个文件：
CDH-5.14.0-1.cdh5.14.0.p0.23-el7.parcel
CDH-5.14.0-1.cdh5.14.0.p0.23-el7.parcel.sha1
manifest.json
MySQL jdbc驱动
文件名: mysql-connector-java-.tar.gz
下载地址: https://dev.mysql.com/downloads/connector/j/
解压出: mysql-connector-java-bin.jar
```

# 6.所有机器安装安装jdk

# 7.所有机器安装依赖包

```shell
yum -y install chkconfig python bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse portmap fuse-libs redhat-lsb
```

# 8.安装mysql数据库

在第二台机器上(随机选择的机器，计划在第一台机器上安装cloudera管理服务比较耗费资源,所以在第二台机器上安装mysql数据库)安装mysql数据库.

参考【MySQL安装之yum安装教程】

# 9.安装cloudera服务端

## 9.1 解压服务端管理安装包

```shell
#所有节点上传cloudera-manager-centos7-cm5.14.0_x86_64.tar.gz文件并解压
[root@node01 ~]# tar -zxvf cloudera-manager-centos7-cm5.14.2_x86_64.tar.gz -C /opt
[root@node02 ~]# tar -zxvf cloudera-manager-centos7-cm5.14.2_x86_64.tar.gz -C /opt
[root@node03 ~]# tar -zxvf cloudera-manager-centos7-cm5.14.2_x86_64.tar.gz -C /opt
```

解压完可以在/opt目录下看到文件

```shell
[root@node01 ~]# cd /opt/
[root@node01 opt]# ll
total 0
drwxr-xr-x. 4 1106 4001 36 Apr  3  2018 cloudera
drwxr-xr-x. 9 1106 4001 88 Apr  3  2018 cm-5.14.2
[root@node01 opt]# cd cloudera/
[root@node01 cloudera]# ll
total 0
drwxr-xr-x. 2 1106 4001 6 Apr  3  2018 csd
drwxr-xr-x. 2 1106 4001 6 Apr  3  2018 parcel-repo
[root@node01 cloudera]# 
```
## 9.2 创建客户端运行目录

```shell
#所有节点手动创建文件夹
[root@node01 ~]# mkdir /opt/cm-5.14.2/run/cloudera-scm-agent
[root@node02 ~]# mkdir /opt/cm-5.14.2/run/cloudera-scm-agent
[root@node03 ~]# mkdir /opt/cm-5.14.2/run/cloudera-scm-agent

```

## 9.3 创建cloudera-scm用户

```shell
#所有节点创建cloudera-scm用户
useradd --system --home=/opt/cm-5.14.0/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
```

## 9.4 初始化数据库

初始化数据库（只需要在Cloudera Manager Server节点执行）

将提供的msyql驱动包上传到第一台机器的root home目录下，然后将mysql jdbc驱动放入相应位置:

```shell
[root@node01 ~]# cp mysql-connector-java.jar /opt/cm-5.14.2/share/cmf/lib/
[root@node01 ~]#  /opt/cm-5.14.2/share/cmf/schema/scm_prepare_database.sh mysql -h node02 -uroot -p'!Qaz123456' --scm-host node01 scm scm '!Qaz123456'    
JAVA_HOME=/usr/java/jdk1.8.0_211-amd64
Verifying that we can write to /opt/cm-5.14.2/etc/cloudera-scm-server
Creating SCM configuration file in /opt/cm-5.14.2/etc/cloudera-scm-server
Executing:  /usr/java/jdk1.8.0_211-amd64/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/opt/cm-5.14.2/share/cmf/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /opt/cm-5.14.2/etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
[                          main] DbCommandExecutor              INFO  Successfully connected to database.

#显示初始化成功
All done, your SCM database is configured correctly!
[root@node01 ~]# 
```

脚本参数说明:
${数据库类型} -h ${数据库所在节点ip/hostname} -u${数据库用户名} -p${数据库密码} –scm-host ${Cloudera Manager Server节点ip/hostname} scm(数据库)  scm(用户名) scm(密码)



**mysql-connector-java.jar驱动同时需要复制到node02相同目录下.**

## 9.5 修改**所有节点**客户端配置

```shell
#将其中的server_host参数修改为Cloudera Manager Server节点的主机名
[root@node01 ~]# vi /opt/cm-5.14.2/etc/cloudera-scm-agent/config.ini
[root@node01 ~]# vi /opt/cm-5.14.2/etc/cloudera-scm-agent/config.ini 
[General]
# 将默认的server_host=localhost 修改成node01
server_host=node01

```

## 9.6 上传CDH安装包

```shell
#将如下文件放到Server节点的/opt/cloudera/parcel-repo/目录中:
#CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel
#CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha1
#manifest.json
# 重命名sha1文件
[root@node01 parcel-repo]# mv CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha1 CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha

```

## 9.7 更改安装目录用户组权限

**所有节点**更改cm相关文件夹的用户及用户组

```shell
[root@node01 ~]# chown -R cloudera-scm:cloudera-scm /opt/cloudera
[root@node01 ~]# chown -R cloudera-scm:cloudera-scm /opt/cm-5.14.2
[root@node01 ~]# 
```

## 9.8 启动Cloudera Manager和agent

Server(node01)节点

```shell
[root@node01 ~]# /opt/cm-5.14.2/etc/init.d/cloudera-scm-server start
Starting cloudera-scm-server:                              [  OK  ]
#客户端需要在所有节点上启动
[root@node01 ~]# /opt/cm-5.14.2/etc/init.d/cloudera-scm-agent start 
Starting cloudera-scm-agent:                               [  OK  ]
[root@node01 ~]# 

```

# 10.服务安装

使用浏览器登录cloudera-manager的web界面,用户名和密码都是admin

![1570789521954](assets/1570789521954.png)



登陆之后，在协议页面勾选接受协议,点击继续

![1570790224972](assets/1570790224972.png)



选择免费版本，免费版本已经能够满足我们日常业务需求,选择免费版即可.点击继续

![1570790460382](assets/1570790460382.png)



如下图，点击继续

![1570790639728](assets/1570790639728.png)

如下图，点击当前管理的机器，然后选择机器，点击继续

![1570790871033](assets/1570790871033.png)



如下图，然后选择你的parcel对应版本的包

![1570790984431](assets/1570790984431.png)

点击后，进入安装页面，稍等片刻

如下图，集群安装中 

![1570791090600](assets/1570791090600.png)



如下图，安装包分配成功，点击继续

![1570793156892](assets/1570793156892.png)



![1570793245342](assets/1570793245342.png)



针对这样的警告，需要在每一台机器输入如下命令：

```
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo 'vm.swappiness=10'>> /etc/sysctl.conf
sysctl vm.swappiness=10

echo never > /sys/kernel/mm/transparent_hugepage/defrag”和
“echo never > /sys/kernel/mm/transparent_hugepage/enabled”
```

如下图，然后点击重新运行，不出以为，就不会在出现警告了，点击完成,进入hadoop生态圈服务组件的安装

![1570793435494](assets/1570793435494.png)



如下图，选择自定义服务，我们先安装好最基础的服务组合。那么在安装之前，如果涉及到hive和oozie的安装，那么先去mysql中，自己创建数据库，并赋予权限；

因此：

```sql
create database hive;
create database oozie;

grant all on *.* to hive identified by '!Qaz123456';
grant all on *.* to oozie identified by '!Qaz123456';

```

如果出现如下错误:

```shell
mysql> grant all on *.* to oozie identified by '!Qaz123456';
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
mysql> update mysql.user set Grant_priv='Y',Super_priv='Y' where user = 'root' and host = 'localhost';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> flush privileges;
mysql> quit
Bye
You have new mail in /var/spool/mail/root
[root@node02 ~]# systemctl restart mysqld.service
[root@node02 ~]# mysql -u root -p                
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.27 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> grant all on *.* to hive identified by '!Qaz123456';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> grant all on *.* to oozie identified by '!Qaz123456';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> 
```



这样再安装软件！

那么，选择自定义服务,如果我们后续需要其他服务时我们在进行添加

![1570795590845](assets/1570795590845.png)

然后点击继续，进入选择服务添加分配页面，分配即可

![1570795837600](assets/1570795837600.png)



选择完成后服务，如下图,可以点击按照主机查看服务分部情况

![1570795867075](assets/1570795867075.png)

![1570795808683](assets/1570795808683.png)



点击继续后，如下图，输入mysql数据库中数数据库scm，用户名scm，密码!Qaz123456,点击测试连接，大概等30s，显示成功，点击继续

![1570796013065](assets/1570796013065.png)



一路点击继续,剩下的就是等待

![1570796339035](assets/1570796339035.png)



如上图，如果等待时间过长，我们可以将manager所在机器(也就是node01)停止后把内存调整的大一些建议如果是笔记本4g以上，如果是云环境8g以上，我们这里先调整为4g以上，重新启node01机器后重新启动cloudera的server和agent

```shell
[root@node01 ~]# cd /opt/cm-5.14.2/etc/init.d
#启动server
[root@node01 init.d]# ./cloudera-scm-server start
#启动agent
[root@node01 init.d]# ./cloudera-scm-agent start
```

# 11.重新登录cloudera manager

登录成功后，如下图，重新启动集群,接下来就是等待.

![1570799640007](assets/1570799640007.png)

# 12.集群测试

## 12.1 文件系统测试

```shell
#切换hdfs用户对hdfs文件系统进行测试是否能够进行正常读写
[root@node01 ~]# su hdfs
[hdfs@node01 ~]# hadoop dfs -ls /
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

Found 1 items
d-wx------   - hdfs supergroup          0 2019-10-11 08:21 /tmp
[hdfs@node01 ~]# touch words
[hdfs@node01 ~]# vi words 
hello world

[hdfs@node01 ~]$ hadoop dfs -put words /test
[hdfs@node01 ~]$ hadoop dfs -ls /
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

Found 2 items
drwxr-xr-x   - hdfs supergroup          0 2019-10-11 09:09 /test
d-wx------   - hdfs supergroup          0 2019-10-11 08:21 /tmp
[hdfs@node01 ~]$ hadoop dfs -ls /test
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

Found 1 items
-rw-r--r--   3 hdfs supergroup         12 2019-10-11 09:09 /test/words
[hdfs@node01 ~]$ hadoop dfs -text /test/words
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

hello world

```

## 12.2 yarn集群测试

```shell
[hdfs@node01 ~]$ hadoop jar /opt/cloudera/parcels/CDH-5.14.2-1.cdh5.14.2.p0.3/jars/hadoop-mapreduce-examples-2.6.0-cdh5.14.2.jar wordcount /test/words /test/output
19/10/11 22:47:59 INFO client.RMProxy: Connecting to ResourceManager at node03.kaikeba.com/192.168.52.120:8032
19/10/11 22:47:59 INFO mapreduce.JobSubmissionFiles: Permissions on staging directory /user/hdfs/.staging are incorrect: rwx---rwx. Fixing permissions to correct value rwx------
19/10/11 22:48:00 INFO input.FileInputFormat: Total input paths to process : 1
19/10/11 22:48:00 INFO mapreduce.JobSubmitter: number of splits:1
19/10/11 22:48:00 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1570847238197_0001
19/10/11 22:48:01 INFO impl.YarnClientImpl: Submitted application application_1570847238197_0001
19/10/11 22:48:01 INFO mapreduce.Job: The url to track the job: http://node03.kaikeba.com:8088/proxy/application_1570847238197_0001/
19/10/11 22:48:01 INFO mapreduce.Job: Running job: job_1570847238197_0001
19/10/11 22:48:28 INFO mapreduce.Job: Job job_1570847238197_0001 running in uber mode : false
19/10/11 22:48:28 INFO mapreduce.Job:  map 0% reduce 0%
19/10/11 22:50:10 INFO mapreduce.Job:  map 100% reduce 0%
19/10/11 22:50:17 INFO mapreduce.Job:  map 100% reduce 17%
19/10/11 22:50:19 INFO mapreduce.Job:  map 100% reduce 33%
19/10/11 22:50:21 INFO mapreduce.Job:  map 100% reduce 50%
19/10/11 22:50:24 INFO mapreduce.Job:  map 100% reduce 67%
19/10/11 22:50:25 INFO mapreduce.Job:  map 100% reduce 83%
19/10/11 22:50:29 INFO mapreduce.Job:  map 100% reduce 100%
19/10/11 22:50:29 INFO mapreduce.Job: Job job_1570847238197_0001 completed successfully
19/10/11 22:50:30 INFO mapreduce.Job: Counters: 49
        File System Counters
                FILE: Number of bytes read=144
                FILE: Number of bytes written=1044048
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=118
                HDFS: Number of bytes written=16
                HDFS: Number of read operations=21
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=12
        Job Counters 
                Launched map tasks=1
                Launched reduce tasks=6
                Data-local map tasks=1
                Total time spent by all maps in occupied slots (ms)=100007
                Total time spent by all reduces in occupied slots (ms)=24269
                Total time spent by all map tasks (ms)=100007
                Total time spent by all reduce tasks (ms)=24269
                Total vcore-milliseconds taken by all map tasks=100007
                Total vcore-milliseconds taken by all reduce tasks=24269
                Total megabyte-milliseconds taken by all map tasks=102407168
                Total megabyte-milliseconds taken by all reduce tasks=24851456
        Map-Reduce Framework
                Map input records=1
                Map output records=2
                Map output bytes=20
                Map output materialized bytes=120
                Input split bytes=106
                Combine input records=2
                Combine output records=2
                Reduce input groups=2
                Reduce shuffle bytes=120
                Reduce input records=2
                Reduce output records=2
                Spilled Records=4
                Shuffled Maps =6
                Failed Shuffles=0
                Merged Map outputs=6
                GC time elapsed (ms)=581
                CPU time spent (ms)=11830
                Physical memory (bytes) snapshot=1466945536
                Virtual memory (bytes) snapshot=19622957056
                Total committed heap usage (bytes)=1150287872
        Shuffle Errors
                BAD_ID=0
                CONNECTION=0
                IO_ERROR=0
                WRONG_LENGTH=0
                WRONG_MAP=0
                WRONG_REDUCE=0
        File Input Format Counters 
                Bytes Read=12
        File Output Format Counters 
                Bytes Written=16
You have new mail in /var/spool/mail/root
    [hdfs@node01 ~]$ hdfs dfs -ls /test/output
Found 7 items
-rw-r--r--   3 hdfs supergroup          0 2019-10-11 22:50 /test/output/_SUCCESS
-rw-r--r--   3 hdfs supergroup          0 2019-10-11 22:50 /test/output/part-r-00000
-rw-r--r--   3 hdfs supergroup          8 2019-10-11 22:50 /test/output/part-r-00001
-rw-r--r--   3 hdfs supergroup          0 2019-10-11 22:50 /test/output/part-r-00002
-rw-r--r--   3 hdfs supergroup          0 2019-10-11 22:50 /test/output/part-r-00003
-rw-r--r--   3 hdfs supergroup          0 2019-10-11 22:50 /test/output/part-r-00004
-rw-r--r--   3 hdfs supergroup          8 2019-10-11 22:50 /test/output/part-r-00005
[hdfs@node01 ~]$  hdfs dfs -text /test/output/part-r-00001
world   1
[hdfs@node01 ~]$  hdfs dfs -text /test/output/part-r-00005
hello   1
You have new mail in /var/spool/mail/root
[hdfs@node01 ~]$ 
```

# 13.手动添加Kafka服务

我们以安装kafka为例进行演示

## 13.1 检查kafka安装包

首先检查是否已经存在Kafka的parcel安装包，如下图提示远程提供，说明我们下载的parcel安装包中不包含Kafka的parcel安装包，这时需要我们手动到官网上下载

![1570850847758](assets/1570850847758.png)

## 13.2 检查Kafka安装包版本

首先查看搭建cdh版本 和kafka版本，是否是支持的：

登录如下网址：

```
https://www.cloudera.com/documentation/enterprise/release-notes/topics/rn_consolidated_pcm.html#pcm_kafka
```

我的CDH版本是cdh5.14.0 ，我想要的kafka版本是1.0.1

因此选择：

![img](assets/1633376-20190508130456986-1658024926.png)



## 13.3 下载Kafka parcel安装包

然后下载：<http://archive.cloudera.com/kafka/parcels/3.1.0/>

![img](assets/1633376-20190508130549214-56463812.png)

需要将下载的KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel.sha1 改成 KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel.sha

```shell
[root@node01 ~]# mv KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel.sha1 KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel.sha
You have new mail in /var/spool/mail/root

```

然后将这三个文件，拷贝到parcel-repo目录下。如果有相同的文件，即manifest.json，只需将之前的重命名备份即可。

```shell
[root@node01 ~] cd /opt/cloudera/parcel-repo/
[root@node01 parcel-repo]# mv manifest.json bak_manifest.json 
#拷贝到parcel-repo目录下
[root@node01 ~]# mv KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel* manifest.json /opt/cloudera/parcel-repo/
[root@node01 ~]# ll
total 989036
-rw-------. 1 root root      1260 Apr 16 01:35 anaconda-ks.cfg
-rw-r--r--. 1 root root 832469335 Oct 11 13:23 cloudera-manager-centos7-cm5.14.2_x86_64.tar.gz
-rw-r--r--. 1 root root 179439263 Oct 10 20:14 jdk-8u211-linux-x64.rpm
-rw-r--r--. 1 root root    848399 Oct 11 17:02 mysql-connector-java.jar
-rw-r--r--  1 root root        12 Oct 11 21:01 words
You have new mail in /var/spool/mail/root
[root@node01 ~]# ll
```



## 13.4 分配激活Kafka

如下图，在管理首页选择parcel

![1570859081770](assets/1570859081770.png)

如下图，检查更新多点击几次，就会出现分配按钮

![1570859361922](assets/1570859361922.png)

点击分配，等待分配按钮激活

![1570859491074](assets/1570859491074.png)

如下图，正在分配中...

![1570859524848](assets/1570859524848.png)

如下图按钮已经激活

![1570859672818](assets/1570859672818.png)

![1570859816399](assets/1570859816399.png)

如上两张图图，点击激活和确定，然后等待激活	

正在激活...

![1570859868869](assets/1570859868869.png)

如下图，分配并激活成功

![1570859891912](assets/1570859891912.png)



## 13.5 添加Kafka服务

点击cloudera manager回到主页

![1570860017451](assets/1570860017451.png)

页面中点击下拉操作按钮，点击添加服务

![1570849216563](assets/1570849216563.png)



如下图，点击选择kafka，点击继续

![1570849280747](assets/1570849280747.png)



如下图，选择Kakka Broker在三个节点上安装，Kafka MirrorMaker安装在node03上，Gateway安装在node02上（服务选择安装，需要自己根据每台机器上健康状态而定,这里只是作为参考）

![1570849433668](assets/1570849433668.png)



如下图，填写Destination Broker List和Source Broker List后点击继续

**注意:这里和上一步中选择的角色分配有关联,Kafka Broker选择的是三台机器Destination Broker List中就填写三台机器的主机名，中间使用逗号分开，如果选择的是一台机器那么久选择一台，一次类推.Source Broker List和Destination Broker List填写一样.**



![1570861737283](assets/1570861737283.png)

如下图，添加服务，最终状态为已完成，启动过程中会出现错误不用管，这时因为CDH给默认将kafka的内存设置为50M,太小了， 后续需要我们手动调整,点击继续

![1570862070794](assets/1570862070794.png)

如下图,点击完成.

![1570862239690](assets/1570862239690.png)

如下图，添加成功的Kafka服务

![1570862272925](assets/1570862272925.png)



## 13.6 配置Kafka的内存

如下图，点击Kafka服务

![1570862705339](assets/1570862705339.png)

如下图，点击实例，点击Kafka Broker（**我们先配置node01节点的内存大小,node02和node03内存配置方式相同，需要按照此方式进行修改**）

![1570862765551](assets/1570862765551.png)



如上图，点击Kafka Broker之后，如下图所示，点击配置

![1570862896070](assets/1570862896070.png)

右侧浏览器垂直滚动条往下找到broker_max_heap_size，修改值为256或者更大一些比如1G,点击保存更改

![1570863035798](assets/1570863035798.png)



**node02和node03按照上述步骤进行同样修改.**

## 13.7 重新启动kafka集群

点击启动

![1570863234060](assets/1570863234060.png)

点击启动

![1570863253382](assets/1570863253382.png)





然后kafka在启动中肯定会报错，如下图,因为默认broker最低内存是1G

![1574073350000](assets/1574073350000.png)

但是CDH给调节成50M了

![img](assets/1633376-20190508134454159-439863421.png)

因此调整过来

![img](https://img2018.cnblogs.com/blog/1633376/201905/1633376-20190508134600914-1525900032.png)

启动成功

![1570863458643](assets/1570863458643.png)



# 14.手动添加服务

请参考【13.手动添加Kafka服务】操作步骤.