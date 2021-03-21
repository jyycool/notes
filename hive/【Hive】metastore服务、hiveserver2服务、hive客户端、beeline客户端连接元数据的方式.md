# 【Hive】metastore服务、hiveserver2服务、hive客户端、beeline客户端连接元数据的方式

## 1. 前言

hive 是 Hadoop 的客户端，启动 hive 前必须启动hadoop，同时 hive 的元数据存储在mysql 中，是由于 hive 自带的 derby 数据库不支持多客户端访问。

## 2. 开启metastore服务的参数

hive-site.xml中打开metastore的连接地址。

```xml
<!-- 指定存储元数据要连接的地址 -->
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://hadoop102:9083</value>
    </property>
12345
```

## 3.hive 连接 mysql 中的元数据有2种方式

```
1）直接连接：直接去mysql中连接metastore库；
2）通过服务连：hive有2种服务分别是metastore和hiveserver2，hive通过metastore服务去连接mysql中的元数据。
12
```

## 4.如何控制hive连接元数据的方式？

### 4.1 通过metastore服务连接

如果 hive-site.xml 中有`hive.metastore.uris`这个参数，***就一定需要通过 metastore 服务去连接元数据***。

如果配置了`hive.metastore.uris`, 尝试使用 `bin/hive` 启动hive客户端，使用show databases 就会报异常 FAILED: HiveException  java.lang.RuntimeException:Unable to instantiate  org.apache.hadoop.hive.ql.metadata SessionHiveMetaStoreClient。

### 4.2 直接连接

如果hive-site.xml中没有配置hive.metastore.uris这个参数，但是配置了JDBC连接方式，就可以通过直接连接的方式连接元数据。 通过bin/hive命令就可以连接到 mysql 库中的元数据。

```xml
   <configuration>
       <!-- jdbc连接的URL -->
           <property>
               <name>javax.jdo.option.ConnectionURL</name>
               <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false</value>
           </property>

        <!-- jdbc连接的Driver-->
        <property>
            <name>javax.jdo.option.ConnectionDriverName</name>
            <value>com.mysql.jdbc.Driver</value>
        </property>

        <!-- jdbc连接的username-->
        <property>
            <name>javax.jdo.option.ConnectionUserName</name>
            <value>root</value>
        </property>

        <!-- jdbc连接的password -->
        <property>
            <name>javax.jdo.option.ConnectionPassword</name>
            <value>123456</value>
        </property>
    <configuration>
12345678910111213141516171819202122232425
```

## 5. 如何启动metastore服务？

```
1）通过命令bin/hive --service metastore 。 
启动metastore服务后，再使用bin/hive 启动hive客户端，使用show databases就不会报错。即hive客户端能够通过metastore服务找到元数据。
2）如果不想启动metastore服务才能启动hive客户端，只能将hive.metastore.uris参数注释掉。
3）通过bin/hive --service metastore命令启动metastore服务会占用窗口，可以通过如下命令推向后台。
    nohup hive --service metastore>log.txt 2>&1 &
4）命令解读：前台启动的方式导致需要打开多个shell窗口，可以使用如下方式后台方式启动
    nohup: 放在命令开头，表示不挂起,也就是关闭终端进程也继续保持运行状态
    2>&1 : 表示将错误重定向到标准输出上
    &: 放在命令结尾,表示后台运行
    一般会组合使用: nohup  [xxx命令操作]> file  2>&1 &  ， 表示将xxx命令运行的
    结果输出到file中，并保持命令启动的进程在后台运行。
1234567891011
```

## 6. hive.metastore.uris参数的意义？

对于分布式的hadoop集群，只需要在一个节点上安装hive即可。是因为hive既是客户端又是服务端，作为服务端的原因是有2个后台服务metastore服务和hiveserver2服务。这样对于集群的意义就是，可能会有多个节点安装hive，但是只有一个节点A是主服务端，其余节点BCD是客户端，而存放元数据的mysql节点E只对主服务端A暴露连接地址，其余节点BCD网络隔离根本访问不到mysql。这样节点BCD上hive要连接mysql，必须节点A先开启metastore服务，节点BCD先连接metastore服务再连接mysql。

## 7. 多节点hive客户端如何配置？配置文件hive-site.xml如何写？

```
只需要指定存储元数据要连接的地址即可，但是服务节点hadoop102要提前开启metastore服务，这样客户端节点才能通过bin/hive 命令启动hive客户端连接到元数据。
1
    <?xml version="1.0"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
        <!-- 指定存储元数据要连接的地址 -->
        <property>
            <name>hive.metastore.uris</name>
            <value>thrift://hadoop102:9083</value>
        </property>
    </configuration>
123456789
```

## 8. hiveserver2服务是什么？

- hive的客户端连接hive的需要开启hiveserver2服务，是先连hive，进而可以连元数据。
- hive是hadoop的客户端，同时hive也有自己的客户端。
- hive有2种客户端，hive客户端和beeline客户端。
- 启动beeline客户端，必须先启动hiveserver2服务。再通过命令bin/beeline -u jdbc:hive2://hadoop102:10000 -n atguigu(即用户名)

## 9.Hiveserver2服务的两种启动方式？

```
1）hive命令：bin/hive --service hiveserver2
2）hiveserver2命令:bin/hiveserver2 ,hiveserver2脚本其实也是通过bin/hive --service hiveserver2方式启动hiveserver2服务。
3）hiveserver2服务属于前台启动会占用窗口。
123
```

## 10.如何关闭hiveserver2服务和metastore服务？

```
通过jps -m 命令查看jps进程，通过kill -9 进程号关闭相应的服务。
![通过jps命令查看jps进程，发现metastore服务和hiveserver2服务都是RunJar，想关闭很麻烦](https://img-blog.csdnimg.cn/20201010202044526.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjcxNjIzNw==,size_16,color_FFFFFF,t_70#pic_center)
![jps -l 命令只能查看进程的全类名，也无法分辨2个服务](https://img-blog.csdnimg.cn/20201010202210764.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjcxNjIzNw==,size_16,color_FFFFFF,t_70#pic_center)
![jps -m 可以更加详细的查看进程信息](https://img-blog.csdnimg.cn/20201010202302740.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjcxNjIzNw==,size_16,color_FFFFFF,t_70#pic_center)
```

## 11.总结

```
1）hive有2种客户端：hive客户端和beeline客户端，beeline客户端是通过hiveserver2服务以JDBC的方式连接hive客户端。
2）hive作为服务端有2个后台服务：metastore服务，hiveserver2服务。
3）hive连接元数据有2种方式：直接连接和metastore服务连接。
4）如果配置了hive.metastore.uris参数，必须启动metastore服务才能连接元数据；如果没有配置可以直接连接元数据。
5）hive本身既是客户端，又是服务端。
```

## 12.hive-site.xml配置参数

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <!-- jdbc连接的URL -->
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false</value>
  </property>

  <!-- jdbc连接的Driver-->
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
  </property>

  <!-- jdbc连接的username-->
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
  </property>

  <!-- jdbc连接的password -->
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>123456</value>
  </property>
  <!-- Hive默认在HDFS的工作目录 -->
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
  </property>

  <!-- 指定hiveserver2连接的端口号 -->
  <property>
    <name>hive.server2.thrift.port</name>
    <value>10000</value>
  </property>
  <!-- 指定hiveserver2连接的host -->
  <property>
    <name>hive.server2.thrift.bind.host</name>
    <value>hadoop102</value>
  </property>

  <!-- 指定存储元数据要连接的地址 -->
  <property>
    <name>hive.metastore.uris</name>
    <value>thrift://hadoop102:9083</value>
  </property>
  <!-- 元数据存储授权  -->
  <property>
    <name>hive.metastore.event.db.notification.api.auth</name>
    <value>false</value>
  </property>
  <!-- Hive元数据存储版本的验证 -->
  <property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
  </property>

  <!-- hiveserver2的高可用参数，开启此参数可以提高hiveserver2的启动速度 -->
  <property>
    <name>hive.server2.active.passive.ha.enable</name>
    <value>true</value>
  </property>

  <!-- hive方式访问客户端：打印 当前库 和 表头 -->
  <property>
    <name>hive.cli.print.header</name>
    <value>true</value>
    <description>Whether to print the names of the columns in query output.               </description>
  </property>
  <property>
    <name>hive.cli.print.current.db</name>
    <value>true</value>
    <description>Whether to include the current database in the Hive prompt.         </description>
  </property>

</configuration>
```

## 13. 开启metastore服务和hiveserver2服务脚本

同时开启 hiveserver2 和 metastore 服务的脚本: hiveservices

```shell
#!/bin/bash

HIVE_LOG_DIR=$HIVE_HOME/logs
METASTORE_PORT=9083
HIVESERVER2_PORT=10001

if [ ! -d $HIVE_LOG_DIR ]
then
	mkdir -p $HIVE_LOG_DIR
fi

#检查进程是否运行正常，参数1为进程名，参数2为进程端口
function check_process()
{
	pid=$(ps -ef 2>/dev/null | grep -v grep | grep -i $1 | awk '{print $2}')
	ppid=$(netstat -nltp 2>/dev/null | grep $2 | awk '{print $7}' | cut -d '/' -f 1)
	echo $pid
	[[ "$pid" =~ "$ppid" ]] && [ "$ppid" ] && return 0 || return 1
}

function hive_start()
{
	metapid=$(check_process HiveMetastore $METASTORE_PORT)
	cmd="nohup hive --service metastore >$HIVE_LOG_DIR/metastore.log 2>&1 &"
	cmd=$cmd" sleep 4; hdfs dfsadmin -safemode wait >/dev/null 2>&1"
	[ -z "$metapid" ] && eval $cmd && echo "Starting Hive Metastore......" || echo "Hive Metastore is already running as PID: $metapid"
	sleep 3 
	server2pid=$(check_process HiveServer2 $HIVESERVER2_PORT)
	cmd="nohup hive --service hiveserver2 >$HIVE_LOG_DIR/hiveServer2.log 2>&1 &"
	[ -z "$server2pid" ] && eval $cmd && echo "Starting HiveServer2......" || echo "HiveServer2 is already running as PID: $server2pid"
	sleep 3
}

function hive_stop()
{
	metapid=$(check_process HiveMetastore $METASTORE_PORT)
[ -n "$metapid" ] && kill $metapid && echo "Stoping Hive Metastore......" || echo "Hive Metastore hasn't been started"
	sleep 3
	server2pid=$(check_process HiveServer2 $HIVESERVER2_PORT)
[ -n "$server2pid" ] && kill $server2pid && echo "Stoping HiveServer2......" || echo "HiveServer2 hasn't been started"
	sleep 3
}

case $1 in
	"start")
		hive_start
		;;
	"stop")
		hive_stop
		;;
	"restart")
		hive_stop
		hive_start
		;;
	"status")
		metapid=$(check_process HiveMetastore $METASTORE_PORT)
		server2pid=$(check_process HiveServer2 $HIVESERVER2_PORT)
		if [ -n "$metapid" ];
		then
			echo "Hive Metastore is running as PID: $metapid"
		else
			echo "Hive Metastore hasn't been started"
		fi
		if [ -n "$server2pid" ]; 
		then
			echo "HiveServer2 is running as PID: $server2pid"
		else
			echo "HiveServer2 hasn't been started"
		fi
		;;
	*)
		echo Invalid Args!
		echo 'Usage: '$(basename $0)' start|stop|restart|status'
		;;
esac
```

## 14. 查看各节点jps进程脚本

支持传入参数如 jps -m ; jps -l

```shell
vim /usr/local/bin/jpsall

#!/bin/bash
for i in hadoop102 hadoop103 hadoop104
	do
echo =========== $i ============
	ssh $i "jps $@ | grep -v Jps"
done
```