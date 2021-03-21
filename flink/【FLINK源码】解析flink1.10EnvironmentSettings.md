# 【FLINK源码】解析 flink1.10 EnvironmentSettings

```java
EnvironmentSettings settings = EnvironmentSettings.newInstance()
      .useBlinkPlanner()
      .inStreamingMode()
      .withBuiltInCatalogName("default_catalog")
      .withBuiltInDatabaseName("default_database")
      .build();
```

在创建TableEnvironment之前, 会使用实例化EnvironmentSettings对象

EnvironmentSettings主要做了哪些事: 

1. ***useBlinkPlanner()***

   ```java
   public Builder useBlinkPlanner() {
     // org.apache.flink.table.planner.delegation.BlinkPlannerFactory
     this.plannerClass = BLINK_PLANNER_FACTORY;
     //org.apache.flink.table.planner.delegation.BlinkExecutorFactory
     this.executorClass = BLINK_EXECUTOR_FACTORY;
     return this;
   }
   ```

2. ***inStreamingMode()***

   ```java
   public Builder inStreamingMode() {
     this.isStreamingMode = true;
     return this;
   }
   ```

3. ***withBuiltInCatalogName()和 withBuiltInDatabaseName()***

   ```java
   public Builder withBuiltInCatalogName(String builtInCatalogName) {
   	this.builtInCatalogName = builtInCatalogName;
   	return this;
   }
   ```

   > 指定实例化 TableEnvironment 时要创建的初始目录的名称。此目录将用于存储所有*不可序列化的对象.*
   >
   > *例如通过例如* `TableEnvironment＃registerTableSink(String, TableSink)`或 `TableEnvironment＃registerFunction(String, ScalarFunction)`. 它也是当前目录的初始值，可以通过 `TableEnvironment＃useCatalog(String)` 进行更改。
   >
   > 默认值:  "default_catalog" 。

   ```java
   public Builder withBuiltInDatabaseName(String builtInDatabaseName) {
   	this.builtInDatabaseName = builtInDatabaseName;
   	return this;
   }
   ```

   > 指定实例化TableEnvironment时要创建的初始目录中默认数据库的名称。该数据库将用于*存储所有不可序列化的对象，*
   >
   > 例如通过注册的表和函数. ***TableEnvironment＃registerTableSink(String，TableSink)*** 或 ***TableEnvironment＃registerFunction(String, ScalarFunction)***。 还是当前数据库的初始值，可以通过 ***TableEnvironment＃useDatabase(String)*** 进行更改。
   >
   > 默认值：***“ default_database”***。
   >
   > > 默认目录集鉴于结构 ***default_catalog*** 
   > >
   > > 默认数据库设置为 ***default_database*** 。
   > >
   > > ```java
   > > root:
   > > 	|- default_catalog
   > > 		|- default_database
   > > 			|- tab1
   > > 		|- db1
   > > 			|- tab1
   > > 	|- cat1
   > > 		|- db1
   > > 			|- tab1
   > > ```
   > >
   > > 寻找请求的对象在以下的顺序的路径：
   > >
   > > ```java
   > > [current-catalog].[current-database].[requested-path]
   > > [current-catalog].[requested-path]
   > > [requested-path]
   > > ```

   

   

