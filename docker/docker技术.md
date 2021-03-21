## Docker技术

### 1. Docker引擎



#### 1.1 Docker引擎--简介

***Docker 引擎是用来运行和管理容器的核心组件***. 即通常人们成为的 Docker 或Docker 平台. 

基于开放容器计划 (OCI) 相关标准要求, Docker 引擎采用了模块化的设计原则, 所以其组件是可以替换的.

- Docker 引擎由许多专用的工具协同工作, 从而可以创建和运行容器, 例如 API, 执行驱动, 运行时, shim进程等



Docker引擎由如下主要的组件构成: 

1. Docker客户端(Docker Client)
2. Docker守护进程(Docker deamon)
3. containerd
4. runc

它们共同负责容器的创建和运行

![Docker引擎](/Users/sherlock/Desktop/notes/allPics/Docker/Docker引擎.png)



#### 1.2 Docker引擎--详解

