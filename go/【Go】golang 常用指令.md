# 【Go】golang 常用指令

在项目 code 完成后, 编译 -> 打包 -> 安装, 这个任何一个程序不可避免的生命周期, Go 程序也不例外, 但是 Go 程序没有 Java 项目的管理那样复杂



## 1. go build

该命名分为两种情况, 在项目根目录执行和在任意目录执行

1. 在项目根目录执行 `go build`

   ```sh
   # 若我们当前在 go 项目 go_lecture 根目录下, -o 可以重命名编译后的文件
   ➜  go_lecture go build -o hello   
   ➜  go_lecture ./hello
   Hello World !
   ```

   该命令会在当前目录下生成一个二进制的可执行脚本文件 `hello`

   我们可以直接使用 `./hello` 执行该脚本文件

2. 在任意目录下执行`go build`

   ```sh
   # 若我们在任意目录(这里选择 HOME 目录)下执行编译, 则我们需要在命令后面补全 $GOPATH/src 之后到项目根目录的路径
   ➜  ~ go build github.com/sherlock/go_lecture -o hello
   ➜  ~ ./hello
   Hello World !
   ```

   因为我们当前配置的 `GOPATH=/Users/sherlock/VSworkspace/Go`

   而项目的全路径为 `/Users/sherlock/VSworkspace/Go/src/github.com/sherlock/go_lecture`

   所以可以看出 `go build` 后面的路径就是项目相对于 `$GOPATH` 的相对路径, ***这种方式不推荐, 因为它会将编译后的文件放在当前目录下, 不便于管理.***



## 2. go run

它在项目根目录下使用, 可以在不编译的前提下直接执行 `main` 

```sh
➜  go_lecture go run main.go
Hello World !
```



## 3. go install

这个命令做了两件事:

1. 使用 `go build` 编译源文件.
2. 将编译后的二进制文件 copy 到 `$GOPATH/bin` 目录下, 因此该脚本就会被添加到环境变量中, 到处可执行.



## 4. go get

该命令可以借助代码管理工具通过远程拉取或更新代码包及其依赖包，并自动完成编译和安装。整个过程就像安装一个 App 一样简单。

这个命令在内部实际上分成了两步操作

1. 第一步是下载源码包到 `$GOPATH/pkg`
2. 第二步是执行 go install。下载源码包的 go 工具会自动根据不同的域名调用不同的源码工具，对应关系如下：

```sh
# 使用示例, 远程包名格式是: 域名/机构(作者)/项目名, 这个格式同样用于 src 下go项目的结构
$ go get -v github.com/davyxu/cellnet
```

go get 使用时的附加参数附加参数备

| 附加参数  | 备  注                                 |
| --------- | -------------------------------------- |
| -v        | 显示操作流程的日志及信息，方便检查错误 |
| -u        | 下载丢失的包，但不会更新已经存在的包   |
| -d        | 只下载，不安装                         |
| -insecure | 允许使用不安全的 HTTP 方式进行下载操作 |



## 5. 跨平台编译

有时因为开发平台的生成平台大都不一样, 所以跨平台编译是为了, 编译后的文件能完美运行于生产平台

指定目标操作系统的平台和处理器架构

```sh
# SET 是在 windows cmd 中使用
1. SET CGO_ENABLED=0 	# 禁用CGO
2. SET GOOS=linux			# 目标是Linux平台
3. SET GOARCH=amd64 	# 目标处理器架构是amd64
```

之后在执行 `go build` 就可以得到能在 linux 平台下的可执行文件了

1. Mac 下编译运行于 windows(64位) 和 linux(64位) 平台的可执行文件

   ```sh
   # 编译运行于 Linux 的可执行文件
   CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build
   
   # 编译运行于 Windows 的可执行文件
   CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build
   ```

2. Linux 下编译运行于 windows(64位) 和 Mac(64位) 平台的可执行文件

   ```sh
   # 编译运行于 Mac 的可执行文件
   CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build
   
   # 编译运行于 Windows 的可执行文件
   CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build
   ```

3. Windows 下编译运行于  Mac (64位) 平台的可执行文件

   ```sh
   # 编译运行于 Mac 的可执行文件
   SET CGO_ENABLED=0 
   SET GOOS=darwin
   SET GOARCH=amd64
   go build
   ```

   

