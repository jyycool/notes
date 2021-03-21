## 【FLINK】命令行参数释义

```
./flink <ACTION> [OPTIONS] [ARGUMENTS]
```

可以使用以下操作： 

命令 "run" 编译并运行程序。

```
Syntax: run [OPTIONS] <jar-file> <arguments>
"run" action options:
     -c,--class <classname>               程序入口类
                                          ("main" 方法 或 "getPlan()" 方法)
                                          仅在 JAR 文件没有在 manifest 中指定类的时候使用
     -C,--classpath <url>                 在群集中的所有节点上向每个用户代码类加载器添加URL。
                                          路径必须指定协议（例如文件：//），并且可以在所有节点上访问（例如，通过NFS共享）。
                                          您可以多次使用此选项来指定多个URL。该协议必须由 {@link java.net.URLClassLoader} 支持。
     -d,--detached                        以独立模式运行任务
     -n,--allowNonRestoredState           允许跳过无法还原的保存点状态。
                                          当触发保存点的时候，
                                          你需要允许这个行为如果以从你的应用程序中移除一个算子 
     -p,--parallelism <parallelism>       运行程序的并行度。 可以选择覆盖配置中指定的默认值。
     -q,--sysoutLogging                   将日志输出到标准输出
     -s,--fromSavepoint <savepointPath>   从保存点的路径中恢复作业 (例如
                                          hdfs:///flink/savepoint-1537)
  Options for yarn-cluster mode:
     -d,--detached                        以独立模式运行任务
     -m,--jobmanager <arg>                连接 JobManager（主）的地址。
                                          使用此标志连接一个不同的 JobManager 在配置中指定的
     -yD <property=value>                 使用给定属性的值
     -yd,--yarndetached                   以独立模式运行任务（过期的；用 non-YARN 选项代替）
     -yh,--yarnhelp                       Yarn session CLI 的帮助信息
     -yid,--yarnapplicationId <arg>       用来运行 YARN Session 的 ID
     -yj,--yarnjar <arg>                  Flink jar 文件的路径
     -yjm,--yarnjobManagerMemory <arg>    JobManager 容器的内存可选单元（默认值: MB)
     -yn,--yarncontainer <arg>            分配 YARN 容器的数量(=TaskManager 的数量)
     -ynm,--yarnname <arg>                给应用程序一个自定义的名字显示在 YARN 上
     -yq,--yarnquery                      显示 YARN 的可用资源（内存，队列）
     -yqu,--yarnqueue <arg>               指定 YARN 队列
     -ys,--yarnslots <arg>                每个 TaskManager 的槽位数量
     -yst,--yarnstreaming                 以流式处理方式启动 Flink
     -yt,--yarnship <arg>                 在指定目录中传输文件
                                          (t for transfer)
     -ytm,--yarntaskManagerMemory <arg>   每个 TaskManager 容器的内存可选单元（默认值: MB）
     -yz,--yarnzookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
     -ynl,--yarnnodeLabel <arg>           指定 YARN 应用程序  YARN 节点标签
     -z,--zookeeperNamespace <arg>        用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
                                     使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
 
 
Action "info" 显示程序的优化执行计划(JSON).
 
  Syntax: info [OPTIONS] <jar-file> <arguments>
  "info" action options:
     -c,--class <classname>           具有程序入口的类
                                      ("main" 方法 或 "getPlan()" 方法)
                                      仅在如果 JAR 文件没有在 manifest 中指定类的时候使用
     -p,--parallelism <parallelism>   运行程序的并行度。 可以选择覆盖配置中指定的默认值。
 
 
Action "list" 罗列出正在运行和调度的作业
 
  Syntax: list [OPTIONS]
  "list" action options:
     -r,--running     只显示运行中的程序和他们的 JobID
     -s,--scheduled   只显示调度的程序和他们的 JobID
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
                                      使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
                                     使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
 
 
Action "stop" 停止正在运行的程序 （仅限流式处理作业）
 
  Syntax: stop [OPTIONS] <Job ID>
  "stop" action options:
 
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
                                      使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
                                     使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
 
 
Action "cancel" 取消正在运行的程序。
 
  Syntax: cancel [OPTIONS] <Job ID>
  "cancel" action options:
     -s,--withSavepoint <targetDirectory>   触发保存点和取消作业。
                                            目标目录是可选的。
                                            如果没有指定目录，使用默认配置
                                            (state.savepoints.dir)。
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
                                      使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
                                     使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
 
 
Action "savepoint" 触发运行作业的保存点，或处理现有作业。
 
  Syntax: savepoint [OPTIONS] <Job ID> [<target directory>]
  "savepoint" action options:
     -d,--dispose <arg>       保存点的处理路径。
     -j,--jarfile <jarfile>   Flink 程序的 JAR 文件。
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
                                      使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
  Options for default mode:
     -m,--jobmanager <arg>           连接 JobManager（主）的地址。
                                     使用此标志连接一个不同的 JobManager 在配置中指定的。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
 
 
Action "modify" 修改正在运行的作业 （例如：修改并行度）.
 
  Syntax: modify <Job ID> [OPTIONS]
  "modify" action options:
     -h,--help                           用来显示命令行的帮助信息。
     -p,--parallelism <newParallelism>   指定作业新的并行度。
     -v,--verbose                        这个选项过期了
  Options for yarn-cluster mode:
     -m,--jobmanager <arg>            连接 JobManager（主）的地址。
                                      使用此标志连接一个不同的 JobManager 在配置中指定的。
     -yid,--yarnapplicationId <arg>   用来运行 YARN Session 的 ID。
     -z,--zookeeperNamespace <arg>    用来创建高可用模式的 Zookeeper 的子路径的命名空间。
 
  Options for default mode:
     -m,--jobmanager <arg>           要连接的JobManager（主节点）的地址。
                                     使用此标志可连接到与配置中指定的不同的 JobManager。
     -z,--zookeeperNamespace <arg>   用来创建高可用模式的 Zookeeper 的子路径的命名空间。
```

