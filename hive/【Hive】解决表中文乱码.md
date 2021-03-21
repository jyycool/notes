# 【Hive】解决表中文乱码



## 表注释中文乱码

创建表的时候，comment说明字段包含中文，表成功创建成功之后，中文说明显示乱码.

```sql
create table test_tb(
id int comment '测试 ID',
name string comment '测试名字',
hobby array<string> comment '测试 array',
add map<String,string> comment '测试 map'
)
row format delimited
fields terminated by '\t'
collection items terminated by '-'
map keys terminated by ':'
;
```

beeline 连接 hive 后, 使用命令 desc formatted test_tb, 中文注释显示乱码.

![hive乱码1](/Users/sherlock/Desktop/notes/allPics/Hive/hive乱码1.png)

这是因为在MySQL中的元数据出现乱码

## 针对元数据库中的表,分区,视图的编码设置

那么我们只需要把相应注释的地方的字符集由 latin1 改成 utf-8，就可以了。用到注释的就三个地方，表、分区、视图。如下修改分为两个步骤：

#### 1. 在元数据数据库中执行以下SQL

##### （1）修改表字段注解和表注解

```sql
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8;
alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
```

##### （2）修改分区字段注解

```sql
alter table PARTITION_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
alter table PARTITION_KEYS modify column PKEY_COMMENT varchar(4000) character set utf8;
```

##### （3）修改索引注解

```sql
alter table INDEX_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
```

#### 2、修改 metastore 的连接 URL

 修改hive-site.xml配置文件

```xml
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost:3306/metastore_hive?createDatabaseIfNotExist=true&amp;useUnicode=true&amp;characterEncoding=UTF-8&amp;useSSL=false</value>
</property>
```

## 验证

做完重启 hive 服务, 之后在查看可以解决乱码问题

![hive乱码2](/Users/sherlock/Desktop/notes/allPics/Hive/hive乱码2.png)

