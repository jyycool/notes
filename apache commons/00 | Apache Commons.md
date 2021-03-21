# 00 | Apache Commons

## 一、简介

Apache Commons 是一个成功而默默无闻的框架。如果你至少参与了一个中型规模的Java项目，那么有很大的机会会接触和使用到了 Apache Commons。

在 Java 生态中有两个优秀且功能强大的底层框架: Apache Commons 和 Google Guava。它们极大的丰富了 Java 基础库的功能。

本系列主要为了选择性的学习 Apache Commons, 具体请参考[官网](http://commons.apache.org)。

Apache Commons是一个 Apache 项目，专注于可重用的 Java 组件的各个方面。由三部分组成：

- Commons Proper - 可重用的Java组件的存储库。
- Commons Sandbox - Java组件开发的工作区。
- Commons Dormant - 当前处于非活动状态的组件的存储库。

## 二、Apache Commons Proper
Commons Proper 致力于一个主要目标：创建和维护可重用的Java组件。Commons Proper 是协作和共享的地方，来自Apache社区的开发人员可以在这些项目上共同努力，以供 Apache 项目和 Apache 用户共享。

Commons 开发人员将努力确保其组件对其他库的依赖性最小，以便可以轻松部署这些组件。另外，Commons 组件将保持其接口尽可能稳定，以便Apache用户（包括其他Apache项目）可以实现这些组件，而不必担心将来的更改。

下面给出了当前可以使用的组件的概览。

- becl: 字节码工程库-分析，创建和操作Java类文件。
- commons-beanutils: 围绕 Java reflection 和 introspection 的 APIs 构建的易于使用的包装器。
- bsf-api: Bean脚本框架-脚本语言（包括JSR-223）的接口。
- commons-chain: 责任链模式的实现。
- commons-cli: 命令行参数解析器。
- commons-codec: 通用编码/解码算法 (例如: 语音，base64，URL)。
- commons-collections4: 扩展或扩充Java Collections Framework。
- commons-compress: 定义用于处理 tar，zip 和 bzip2 文件的API。
- commons-configuration2: 读取各种格式的配置/首选项文件。
- commons-crypto: 使用 AES-NI 封装 Openssl 或 JCE 算法实现的优化的密码库。
- commons-csv: 用于读取和写入逗号分隔值文件的组件。
- commons-daemon: Java代码实现的类似于unix-daemon的调用机制。
- commons-dbcp2: 数据库连接池服务。
- commons-dbutils: JDBC 程序帮助库。
- commons-digester3: 实现XML到Java对象映射的功能。
- commons-email: 用于从Java发送电子邮件的库。
- commons-exec: 使用Java处理外部流程执行和环境管理的API。
- commons-fileupload: Servlet和Web应用程序的文件上传功能。
- commons-io: I/O 功能的集合库。
- jcs: Java缓存系统
- commons-jelly: 基于XML的脚本和处理引擎。
- commons-lang3: 为java.lang中的类提供额外的功能。
- commons-logging: 包装各种日志API实现。
- commons-math3: 轻巧的, 独立的数学和统计组件。
- commons-net: 实现网络和协议的功能库。
- commons-numbers-core: 数字类型(复数，四元数，分数)和实用程序(数组，组合数学)。
- commons-pool2: 通用对象池组件。
- commons-proxy: 用于创建动态代理的库。
- commons-rng-simple: 随机数生成器的实现。
- commons-text: 专注于处理字符串的算法的库。
- commons-validator: 在xml文件中定义验证器和验证规则的框架。
- commons-vfs2: 虚拟文件系统组件，用于将文件，FTP，SMB，ZIP等视为单个逻辑文件系统。
- commons-weaver-processor: 提供一种简单的方法来增强（编织）已编译的字节码。