# 【Zeppelin】安装

## 1. 版本说明

Zeppelin的安装和启动，其实还是比较容易的，关键在于版本兼容问题，当版本选对了其实启动Zeppelin很顺利。(巨坑的是 Interpreter 的安装, 因为官网并没有说要安装插件包......)

建议按照下面的版本安装，以防因版本兼容问题出现异常，我刚开始使用的是jdk11，启动zeppelin时就会出现Zeppelin内部的Lucene，因jdk版本问题而报异常日志.在各自官网下载对应的版本

| 软件     | 版本                       |
| -------- | -------------------------- |
| Zeppelin | zeppelin-0.9.0-bin-all.tgz |
| JDK      | jdk1.8.0_271               |

## 2. 安装包

下载地址 http://zeppelin.apache.org/download.html，例如我们下载最新版本 zeppelin-0.9.0-bin-all.tgz

因为我是安装在本地 Mac, 所以直接解压到安装目录

```sh
tar -zxvf ./zeppelin-0.9.0-bin-all.tgz -C ~/devTools/opt/
```

配置环境变量，便于访问zeppelin的命令, 因为我在 /usr/local 使用了软连接, 这样方便当版本变化时不需要修改 ~/.bash_profile

```sh
➜  ~ cd /usr/local
➜  local sudo ln -s ~/devTools/opt/zeppelin-0.9.0 zeppelin

➜  local vi ~/.bash_profile

# 将 ZEPPELIN_HOME 添加到环境变量,以及 PATH 中
export ZEPPELIN_HOME=/usr/local/zeppelin
export PATH=$ZEPPELIN_HOME/bin:$PATH
# 之后就算 zeppelin 版本变化, 只需要修改软连接,不需要修改 bash_profile 文件了

➜  local source ~/.bash_profile
```

## 3. 修改配置

### 3.1 修改 zeppelin-site.xml

在 zeppelin/conf 目录下,复制出 zeppelin-site.xml 和 zeppelin-env.sh, 来修改配置参数

```sh
➜  ~ cd /usr/local/zeppelin
➜  zeppelin cp conf/zeppelin-site.xml.template conf/zeppelin-site.xml
➜  zeppelin cp conf/zeppelin-env.sh.template conf/zeppelin-env.sh
```

zeppelin-site.xml 该文件中修改启动监听的**ip**和**port**，修改的内容如下：

```xml
/*默认ip是127.0.0.1，表示只能在本机访问zeppelin*/
<property>
	<name>zeppelin.server.addr</name>
	<value>0.0.0.0</value>
	<description>Server binding address</description>
</property>

/*默认是8080，我们修改为18080*/
<property>
	<name>zeppelin.server.port</name>
	<value>18080</value>
	<description>Server port.</description>
</property>
```

### 3.2 修改zeppelin的内存大小

修改 zeppelin-env.sh 文件

- 修改zeppelin的内存有2个方面：(这里可以不修改,默认都给了1G内存,本地改太大,电脑吃力)

  通过修改 zeppelin-env.sh里的 ZEPPELIN_MEM 来修改zeppelin server 的内存

  通过修改 zeppelin-env.sh里的 ZEPPELIN_INTP_MEM 来修改 interpreter 进程的内存

- 配置zeppelin-env.sh中的**JAVA_HOME**

  配置zeppelin-env.sh中的JAVA_HOME参数，使其指向JDK路径，（若使用其他版本，例如jdk11、jdk15可能会有版本兼容问题，因此我们直接使用下列jdk版本）

  ```sh
  export JAVA_HOME=${path_to_your_jdk1.8.0_271}
  ```

**重启zeppelin使得配置生效**

## 4. 启动访问

因为 zeppelin 的启动命令有些长, zeppelin-daemon.sh,......,我比较懒,所以写了一个启动脚本

zeppelin. 并将其放在了 ~/devTools/bin 目录下, 该目录已经配置在我环境变量里

```sh
#! /bin/sh

# 因为 zeppelin pid和日志文件, 都是用了 user-hostname 来拼接
HOSTNAME=sherlock-mbp
ZP_PATH=/usr/local/zeppelin
PIDFILE=${ZP_PATH}/run/zeppelin-${HOSTNAME}.pid

case "$1" in
	start)
		if [ ! -f ${PIDFILE} ] 
		then
			${ZP_PATH}/bin/zeppelin-daemon.sh start 
		else
			PID=$(cat ${PIDFILE})
		echo "Zeppelin Server has bee running as PID: ${PID}"      
		fi
		;;
	stop)
		if [ -f ${PIDFILE} ] 
		then
			${ZP_PATH}/bin/zeppelin-daemon.sh stop 
		else
			echo "Zeppelin Server is not running"
		fi
		;;
	status)
		pid=`ps ax | grep -i 'zeppelin.server' | grep -v grep | awk '{print $1}'`
		[ -n "${pid}" ] && echo "Zeppelin Server is Running as PID: ${pid}" || echo "Zeppelin Server hasn't been started"
		;;
	*)
		echo "Please use [start|stop|status] as first argument"
		;;
esac
exit 0

```

执行启动命令：

```sh
# zeppelin 自带的启动命令
zeppelin-daemon.sh start|stop|restart

# 自定义zeppelin脚本的启动命令
zeppelin start|stop|status
```

如下启动日志，表示成功

![img](https://img-blog.csdnimg.cn/20201218223101502.png)

页面访问 http:localhost:18080

![img](https://img-blog.csdnimg.cn/20201218223143167.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpdGxpdDAyMw==,size_16,color_FFFFFF,t_70) 

## 5. 参考资料

https://www.yuque.com/jeffzhangjianfeng/gldg8w/bam5y1

http://zeppelin.apache.org/

http://zeppelin.apache.org/docs/0.9.0-preview2/quickstart/explore_ui.html

https://blog.csdn.net/xianpanjia4616/article/details/107438077