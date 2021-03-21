# 【Linux】各种执行shell脚本命令的区别

Linux 下如果存在一个脚本 demo.sh , 执行脚本的方式有以下几种

1. ***sh demo.sh***
2. ***source demo.sh***
3. ***. demo.sh***
4. ***./demo.sh***



新建一个脚本 ***echoPid.sh***, 脚本内容如下

```shell
#! /bin/bash
echo "======================"
echo "脚本[$0]的PID: $$"
echo "脚本[$0]的PPID: $PPID"
echo "======================"
```

当前shell环境的 ***PPID: 99441***

```shell
➜  lecture echo $PPID
99441
```

分别以上述4中方式执行脚本

### sh echoPid.sh

```sh
➜  lecture sh echoPid.sh
======================
脚本[echoPid.sh]的PID: 1771
脚本[echoPid.sh]的PPID: 99442
======================
```



### source echoPid.sh

```shell
➜  lecture source echoPid.sh
======================
脚本[echoPid.sh]的PID: 99442
脚本[echoPid.sh]的PPID: 99441
======================
```



### ./echoPid.sh

```sh
➜  lecture ./echoPid.sh
======================
脚本[./echoPid.sh]的PID: 1798
脚本[./echoPid.sh]的PPID: 99442
======================
```



### . echoPid.sh

```sh
➜  lecture . echoPid.sh
======================
脚本[echoPid.sh]的PID: 99442
脚本[echoPid.sh]的PPID: 99441
======================
```



### 结论

1. ***./echoPid.sh*** 和 ***sh echoPid.sh*** 执行脚本获得的 ***PPID 都是 99442***. 它们实际上都是启了一个subshell, ***然后在subshell环境中来执行echoPid.sh脚本***, 所以它们获取的***PPID***都是一样的, 执行它们的shell 并不是用户当前PID为99441的shell环境, 而是启动的新的PID为99442的shell的环境, 我们称这个新启动的shell为子***subshell(子shell)***

2. ***source echoPid.sh*** 和 ***. echoPid.sh*** 它们执行脚本获得的 ***PPID 都是 99441***,  所以这两种方式执行脚本的 shell环境就是 当前用户所在的shell 环境, 那个 ***$PPID 为99441*** 的 shell 环境

####  Tips: 

1. 用 ***sh xx.sh*** 和 ***source xx.sh*** 去执行时， 不要求 被执行脚本 有可执行权限， 但单独 ***./*** 这样去执行时，需要 被执行脚本 有可执行权限

2. 实际项目中初始化环境变量的脚本，都是用 ***source xx.sh 或 . xx.sh*** 执行脚本， 确保设置的环境变量在当前 shell环境 生效

3. 使用 ***( command )***, 也是在一个 subshell 中来执行 command 指令

