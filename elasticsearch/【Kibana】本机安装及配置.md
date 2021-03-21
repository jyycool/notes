# 【Kibana】本机安装及配置

## 准备

1. 解压 tar 包, 创建软连接及配置环境变量

   ```sh
   # 1. 解压 tar包,创建软连接
   ➜  ~ tar -zxvf /Users/sherlock/Downloads/softpack/es/kibana-7.5.2-darwin-x86_64.tar.gz -C /Users/sherlock/devTools/opt
   
   # 2. 创建软连接
   ➜  ~ sudo ln -s /Users/sherlock/devTools/opt/kibana-7.5.2 /usr/local/kibana
   
   # 3. 修改环境变量(修改 ~/.bash_profile,添加如下内容)
   export KIBANA_HOME=/usr/local/kibana
   export PATH=$KIBANA_HOME/bin:${PATH}
   ```

   

