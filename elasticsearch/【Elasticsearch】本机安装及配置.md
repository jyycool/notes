# 【Elasticsearch】本机安装及配置



## 准备

1. 解压 tar 包, 创建软连接及配置环境变量

   ```sh
   # 1. 解压 tar包,创建软连接
   ➜  ~ tar -zxvf /Users/sherlock/Downloads/softpack/es/elasticsearch-7.5.2-darwin-x86_64.tar.gz -C /Users/sherlock/devTools/opt
   
   # 2. 创建软连接
   ➜  ~ sudo ln -s /Users/sherlock/devTools/opt/elasticsearch-7.5.2 /usr/local/elasticsearch
   
   # 3. 修改环境变量(修改 ~/.bash_profile,添加如下内容)
   export ES_HOME=/usr/local/elasticsearch
   export PATH=$ES_HOME/bin:${PATH}
   ```



## 配置

- ***修改 JVM - config/jvm.options***

  默认配置设置是 1GB

  - 配置建议
    1. Xmx 和 Xmx 设置成一样
    2. Xmx 不要超过机器内存的 50%
    3. 不要超过 30GB - https://www.elastic.co/blog/a-heap-of-trouble



## 单节点集群的启动方式

ES 可以在单个节点 (单台机器) 启动多个 ES 实例

```sh
bin/elasticsearch -E node.name=node1 -E cluster.name=es-cluster -E path.data=data/es-cluster
bin/elasticsearch -E node.name=node2 -E cluster.name=es-cluster -E path.data=data/es-cluster
bin/elasticsearch -E node.name=node3 -E cluster.name=es-cluster -E path.data=data/es-cluster
```

