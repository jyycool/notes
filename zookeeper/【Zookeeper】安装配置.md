# 【Zookeeper】安装配置

MacOS 下 安装和配置单节点的 zookeeper-3.5.8 

### 1. 解压 tar 包

```sh
➜ ~ tar -zxvf ~/Downloads/apache-zookeeper-3.5.8-bin.tar.gz -C ~/devTools/opt
➜ ~ cd ~/devTools/opt
➜ opt mv apache-zookeeper-3.5.8-bin zookeeper-3.5.8
```



### 2. 配置环境变量

修改 ~/.bash_profile

```sh
export ZOOKEEPER_HOME=/Users/sherlock/devTools/opt/zookeeper-3.5.8
export ZK_LOG_DIR=$ZOOKEEPER_HOME/logs
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```



### 3. 修改配置

修改 zoo.cfg

```sh
dataDir=/Users/sherlock/devTools/opt/zookeeper-3.5.8/data
```



### 4. 启动zk

```sh
➜ zookeeper-3.5.8 bin/zkServer.sh start
```



### 5. 常用指令

```sh
# 启动客户端
zkCli.sh -server localhost:2181 

help		    # 查看命令
quit		    # 退出
create 	    # -e 创建临时znode , -s 自动编号
get path	  # 查看信息
ls path 	  # 查看指定目录的列表
rmr path	  # 删除

ls /   									# 查看根目录
create -e /myapp  msg   # 创建目录
get /myapp 		       		# 查看myapp创建信息
ls / watch			   			# 添加关注事件
rmr	/myapp		          # 删除触发关注事件
quit
```

详细指令可参考[博客](https://www.cnblogs.com/jimcsharp/p/8358271.html)

