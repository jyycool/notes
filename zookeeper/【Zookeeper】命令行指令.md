# 【Zookeeper】命令行指令

1. 创建节点

   ```sh
   # 创建节点，-s表示顺序节点，-e表示临时节点，默认是持久节点
   create [-s] [-e] path data acl
   # 示例
   create /zk-book 123
   ```

2. 查看节点

   ```sh
   ls path [watch]
   # 示例
   ls /zk-book
   ```

   

3. 获取节点数据和属性

   ```sh
   get path [watch]
   # 示例
   get /zk-book
   ```

   

4. 更新节点数据内容

   ```sh
   set path data [version]
   # 示例
   set /zk-book 456
   ```

   

5. 删除节点

   ```sh
   # 需要注意的是，要删除一个节点，其必须不包含任何子节点
   delete path [version]
   # 示例
   delete /zk-book
   ```

   

6. 权限设置

   ```sh
   # acl表示权限控制的方式
   create [-s] [-e] path data acl
   # 示例
   create -e /zk-book init digest:foo:FhzrsBhE8B6bwLiyO6Tn3pMvp6Y=:cdrwa
   ```

   

7. 设置权限

   ```sh
   setAcl path acl
   # 示例
   setAcl /zk-book digest:foo:FhzrsBhE8B6bwLiyO6Tn3pMvp6Y=:cdrwa
   ```

   

8. 获取权限

   ```sh
   getAcl path
   # 示例
   getAcl /zk-book
   ```

   

9. 超级管理员

   ```sh
   # 超级管理员的权限需要在zookeeper启动时以系统属性的方式进行配置，
   # 这里密码需要填写编码之后的形式，其是将一个用户密码通过BASE64(SHA-1(username:password))编码后的字符串，
   # 在进行权限访问时需要使用原始密码
   -Dzookeeper.DigestProviderAuthentication.superDigest=foo:kWN6aNSbjcKWPqjiV7cg0N24raU=
   ```

   

