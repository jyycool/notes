## Docker概览

### 1. Overview

对于 Mac 版本的 Docker 来说, 提供基于 Mac 原生操作系统中的 Darwin 内核的 Docker 引擎无甚意义. 所以***在 Mac 版的 Docker 中, Docker daemon 是运行在一个轻量级的 Linux VM 之上的***.

***Mac 版本的 Docker 通过对外提供 daemon 和 API 的方式与 Mac 环境实现无缝集成***. 所以用户可以在 Mac 上打开终端并直接使用 Docker 命令.

尽管实现了在Mac上的无缝集成, 但 Mac 版本的 Docker 底层还是基于 Linux VM 运行的, 所以说 ***Mac 版本的 Docker 上只能运行基于 Linux 的 Docker 容器***.

### 2. Docker的基本配置及信息

***docker info*** : 查看docker信息, 例如 Linux内核版本, container总数, 存储驱动, docker root dir, registry镜像地址......

```shell
dockerHub docker info
Client:
 Debug Mode: false

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 1
 Server Version: 19.03.8
 Storage Driver: overlay2
  Backing Filesystem: <unknown>
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 7ad184331fa3e55e52b890ea95e65ba581ae3429
 runc version: dc9208a3303feef5b3839f4323d9beb36df0a9dd
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 4.19.76-linuxkit
 Operating System: Docker Desktop
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 1.945GiB
 Name: docker-desktop
 ID: 76UE:VFZQ:WZRY:DTKB:X46R:X5WK:WYQC:XQS3:7RWS:EYTG:26WJ:DHMB
 Docker Root Dir: /var/lib/docker
 Debug Mode: true
  File Descriptors: 39
  Goroutines: 50
  System Time: 2020-05-31T01:08:24.971360395Z
  EventsListeners: 3
 HTTP Proxy: gateway.docker.internal:3128
 HTTPS Proxy: gateway.docker.internal:3129
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Registry Mirrors:
  http://hub-mirror.c.163.com/
 Live Restore Enabled: false
 Product License: Community Engine
```

docker各个组件版本号, 例如docker, docker-compose, docker-machine, notary......

```shell
➜  dockerHub docker version
Client: Docker Engine - Community
 Version:           19.03.8
 API version:       1.40
 Go version:        go1.12.17
 Git commit:        afacb8b
 Built:             Wed Mar 11 01:21:11 2020
 OS/Arch:           darwin/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.8
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.17
  Git commit:       afacb8b
  Built:            Wed Mar 11 01:29:16 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.2.13
  GitCommit:        7ad184331fa3e55e52b890ea95e65ba581ae3429
 runc:
  Version:          1.0.0-rc10
  GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```

> 注意: 
>
> 1. Server 的OS/Arch 属性中显示的值是 linux/amd64. 这是因为 daemon 是基于前文提到过的 Linux VM 运行的.
>
> 2. Client 是原生的 Mac 组件, 运行于 Mac 操作系统的 Darwin 内核之上 (OS/Arch: darwin/amd64)
>
> 3. 当前 Docker 是稳定版 (Stable) 不是抢鲜版 (Edge) ,所以 Experimental 显示值为 false
>
> 4. Mac 版本的 Docker 安装了 Docker 引擎 (客户端及服务端守护进程)、Docker Compose、Docker machine 以及 Notary 命令行. 查看方式如下
>
>    ```sh
>    ➜  dockerHub docker -v
>    Docker version 19.03.8, build afacb8b
>    ➜  dockerHub docker-compose -v
>    docker-compose version 1.25.5, build 8a1c60f6
>    ➜  dockerHub docker-machine -v
>    zsh: command not found: docker-machine
>    ➜  dockerHub notary version
>    notary
>     Version:    0.6.1
>     Git commit: d6e1431f
>    ```
>
>    实际执行时 发现 docker-machine 并未默认安装, 需要手动安装.
>
>    ```sh
>    curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-`uname -s`-`uname -m` >/usr/local/bin/docker-machine && chmod +x /usr/local/bin/docker-machine
>    ```



### 3. Docker组件

Docker 的核心组件

1. Docker 客户端和服务器

2. 镜像 (Image)
3. 容器 (Container)
4. 仓库 (Registry)

#### 3.1 Docker 客户端和服务器

Docker 是一个 C/S 结构的程序, Docker 客户端只要向 Docker 服务端或守护进程发出请求, 服务端或守护进程将完成所有工作并返回结果. Docker 提供了一个命令行工具 docker 以及一整套 Restful API. 我们可以在同一台宿主机器上运行 Docker 守护进程和客户端, 也可以从本地的 Docker 客户端连接到运行在另一台宿主机上的远程 Docker 守护进程.



#### 3.2 Doker镜像

将镜像理解为一个包含了 OS 文件系统和应用的对象会很有帮助. 如果实际操作过, 就会认为与虚拟机模板类似. 虚拟机模板本质上是处于关机状态的虚拟机. 在 Docker 中, 镜像实际等价于未运行的容器. 在开发层面上, 可以将镜像类比为类(Class)

查看当前本地镜像 ***docker image ls / docker images***

```sh
➜  ~ docker image ls
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
zjffdu/zeppelin-blink   latest              41d3d93a261c        16 months ago       5.07GB
```

所以现在我们只要知道镜像包含了 基础操作系统, 以及应用程序运行所需的代码和依赖包, 例如拉取一个 Ubuntu

>拉取镜像: ***docker image pull <镜像1>*** 
>
>- ***docker image pull ubuntu*** (不加tag, 默认pull latest)
>
>删除镜像:  ***docker image rm [选项] <镜像1> [<镜像2> ...]***
>
>- ***docker image rm zjffdu/zeppelin-blink:latest***
>- ***docker image rm 41d***
>
>查看镜像分层:  ***docker image inspect <镜像1>*** 
>
>- ***docker image inspect ubuntu:latest***  (有4层)
>
>  ```json
>  "RootFS": {
>  	"Type": "layers",
>  	"Layers": [
>                  "sha256:7789f1a3d4e9258fbe5469a8d657deb6aba168d86967063e9b80ac3e1154333f",
>                  "sha256:9e53fd4895597d04f8871a68caea4c686011e1fbd0be32e57e89ada2ea5c24c4",
>                  "sha256:2a19bd70fcd4ce7fd73b37b1b2c710f8065817a9db821ff839fe0b4b4560e643",
>                  "sha256:8891751e0a1733c5c214d17ad2b0040deccbdea0acebb963679735964d516ac2"
>  	]
>  }
>  ```



#### 3.3 Registry

Docker 使用 Registry 来保存用户构建的镜像. Registry分为公共和私有两种. Docker 公司运营的公共的 Registry 是 Docker Hub, 用户可以在其上注册账号, 分享并保存自己的镜像



#### 3.4 容器

Docker 可以帮助构建和部署容器, 我们只需要把自己的应用程序或服务打包放在容器中即可, 而且容器是基于镜像启动起来的, 容器中可以运行一个或多个进程. 我们可以认为, 镜像是 Docke r生命周期中的构建和打包阶段, 而容器则是 Docker 生命周期中的打包和执行阶段

总结来容器就是:

	- 一个镜像格式
	- 一系列标准的操作
	- 一个执行环境



##### 3.4.1 运行容器

每个容器包含一个软件镜像, 可就是容器的"货物", 并且与真正的货物一样, 容器内的软件镜像可以被创建, 启动, 关闭, 重启及销毁

>容器运行: ***docker container [选项] run <镜像> [参数]***
>
>- ***docker container run -it Ubuntu:latest /bin/bash***
>
>  ```shell
>  ➜  dockerHub docker container -it ubuntu:latest /bin/bash
>  root@9ed6cd1e75a3:/#
>  ```
>
>  可以发现终端变了,  因为 -it 参数会将 shell 切换到容器终端, 已经进入到容器内部 ( ubuntu 的终端环境),

docker container run 会告诉 Docker deamon启动新的容器. ***其中 -it 参数告诉 docker 开启容器的交互模式并将用户当前的 shell 连接到容器终端***. 接下来, 命令告诉 Docker 用户想基于 ubuntu:latest 镜像启动容器. 最后, 命令告诉 Docker ,用户想要在容器内部运行哪个进程

> 在容器内部运行 ps 命令查看当前正在运行的全部进程
>
> ```shell
> root@9ed6cd1e75a3:/# ps -elf
> F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
> 4 S root         1     0  0  80   0 -  1028 -      02:36 pts/0    00:00:00 /bin/bash
> 4 R root        12     1  0  80   0 -  1471 -      02:43 pts/0    00:00:00 ps -elf
> ```

Linux容器内仅包含了两个进程:

1. ***PID 1***: 代表了***/bin/bash*** 进程, 该进程是通过docker container run 命令来通知容器运行的
2. ***PID 12***: 代表了 ***ps -elf*** 进程, 查看的那个钱运行中进程所使用的命令/程序



查看当前系统内全部处于运行状态的容器:

```shell
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
9ed6cd1e75a3        ubuntu:latest       "/bin/bash"         18 minutes ago      Up 18 minutes                           zen_tu
```



##### 3.4.2 连接到运行中的容器

执行 docker container exec 命令, 可以将 shell  连接到一个运行中的 container 终端. 因为之前 Ubuntu 容器仍在运行, 所以下面会创建到该容器的新连接

```shell
➜  ~ docker container exec -it zen_tu bash
root@9ed6cd1e75a3:/# 
```

> 上述示例中 容器名为 zen_tu



##### 3.4.3 退出容器

只用 ***ctrl+PQ 或者 输入 exit*** 退出容器, 回到当前主机中.

再次使用 docker container ls 会显示 ubuntu 容器仍在运行中. 



##### 3.4.4 停止和结束容器

通过 ***docker container stop <container_name>***  和 ***docker container rm <container_name>*** 来停止和杀死容器

```shell
➜  ~ docker container stop confident_zhukovsky
confident_zhukovsky
➜  ~ docker container rm confident_zhukovsky
confident_zhukovsky
➜  ~ docker container ls -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
e9a6804ebdcd        ubuntu:latest       "/bin/bash"         2 minutes ago       Exited (0) 2 minutes ago                       wonderful_ellis
9ed6cd1e75a3        ubuntu:latest       "/bin/bash"         30 minutes ago      Exited (0) 3 minutes ago                       zen_tu
```



### 4. Docker 的技术组件

Docker可以运行在任何安装了现代 Linux 内核的 x64 主机上, 它包括以下及部分组成:

1. 一个原生的 Linux 容器格式( Docker 中称之为 libcontaoner )或者是很流行的容器平台 lxc, libcontainer格式现在是 Docker 的默认格式.
2. Linux 内核的命名空间( namespace), 用于隔离文件系统、进程和网络
   - 文件隔离系统: 每个容器都有自己的 root 文件系统
   - 进程隔离: 每个容器都运行在自己的进程环境中
   - 网络隔离: 容器间的虚拟网络接口和IP地址都是分开的
   - 资源隔离和分组: 使用 cgroups (即 control group, Linux内核特性之一)将 CPU 和内存之类的资源独立给每个 Docker 容器
   - 写时复制: 文件系统都是通过写时复制创建的. 这意味文件系统是分层的, 快速的, 而且占用的磁盘空间更小
   - 日志: 容器产生的 STDOUT 、STDERR 和 STDIN 这些IO流都会被收集器记录日志, 用来进行日志分析和故障排错
   - 交互式 shell: 用户可以创建一个伪 tty 终端, 将其连接到 STDIN, 为容器提供一个交互式的 shell



### 5. Docker存储驱动的选择

每个 Docker 容器都有一个本地存储空间, 用来保存镜像层 (Image Layer) 和挂载的容器文件. 默认情况下, 容器的所有读写操作都是发生在镜像上或挂载的文件系统中, 所以存储对于每个容器性能和稳定性至关重要

本地存储是通过存储驱动(Storage Driver) 进行管理的, 有时也被称为 GraphDriver, 虽然***存储驱动在设计上都采用了 栈式镜像层存储和写时复制 的设计思想***, 但Docker 在Linux 底层支持几种不同的存储驱动的实现方式, 每一种实现方式都采用了不同的方法实现了 镜像层和写时复制.

在Docker 上, Linux 可以选择的一些存储驱动包括 AUFS(最早的,最旧的), Overlay2(当下默认的最佳选择), Device Mapper, Btrfs 和 ZFS.

存储驱动是节点级别的,这意味着在每个Docker 主机上只能选择一种存储驱动. Linux上可通过修改 ***/etc/docker/deamon.json*** 文件来修改存储引擎, 例如将存储引擎修改为 overlay2

```json
{
	"storage-driver": "overlay2"
}
```

在 Mac上可以通过 Docker Desktop 图形版客户端来修改,  Mac 上 ***daemon.json*** 的存储路径是 ***~/.docker/daemon.json***



如果用户修改了正在运行的 Docker 主机的存储驱动, 则现有的 镜像和容器 都将无法运行, 这是因为每种存储驱动在主机上存储镜像的位置是不同的 (通常位于 ***/var/lib/docker/<storage-driver>/...*** 目录下), 修改了存储驱动类型, Docker 就无法找到原有的镜像和容器了, 切换回原来的驱动后, 之前的镜像和容器就可以继续使用了. 

如果希望切换存储驱动后还可以继续使用之前的镜像和容器, 则需要将镜像保存为 Docker 格式, 然后上传到仓库(Registry), 修改了本地存储驱动且重启后, 再将镜像从仓库拉取到本地, 之后重启容器

