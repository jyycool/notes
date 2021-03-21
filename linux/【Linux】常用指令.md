# 【Linux】常用指令

Linux提供了丰富的帮助手册，当你需要查看某个命令的参数时不必到处上网查找，只要man一下即可.

### cat

> The cat utility reads files sequentially, writing them to the standard output.
>
> cat 命令 顺序阅读文件内容, 并将内容输出到标准输出( 屏幕 )

```shell
➜  lecture cat << EOF >> date.txt
heredoc> i want to raise a rogdoll
heredoc> EOF
➜  lecture cat date.txt
this is a file test
this is a file test
i want to raise a rogdoll
```

### tee

> The tee utility copies standard input to standard output, making a copy in zero or more files.
>
> tee 命令拷贝 标准输入 的内容到 标准输出以及0个或多个文件中
>
> - -a 追加标准输出到指定文件中, ( 只是追加, 不会覆盖文件中原内容 )  

通常可以和 管道符| 配合使用

```shell
➜  lecture cat date.txt | tee -a tee.txt
this is a file test
this is a file test
i want to raise a rogdoll
➜  lecture cat tee.txt
this is a file test
this is a file test
i want to raise a rogdoll
```

### & , &>, &&

- ***command &*** : &位于命令最后, 表示command命令后台执行

- ***command &>/dev/null*** : &> 表示混合(标准输出和错误输出)重定向
- ***command1 && command2***: &&表示命令排序, 逻辑执行, command1成功才会执行command2 

```shell
➜  lecture ping -c1 www.baidu.com && echo "=====> is up...." || echo "=====> is down...."
PING www.a.shifen.com (36.152.44.96): 56 data bytes
64 bytes from 36.152.44.96: icmp_seq=0 ttl=58 time=8.068 ms

--- www.a.shifen.com ping statistics ---
1 packets transmitted, 1 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 8.068/8.068/8.068/0.000 ms
=====> is up....

➜  lecture ping -c1 www.baidu.com &>/dev/null && echo "=====> is up...." || echo "=====> is down...."
=====> is up....
```

### `$#、$0、$1、$@、$*、$$、$?`

- ***$#***：传入脚本的参数个数；

- ***$0***: 脚本自身的名称；　　

- ***$1***: 传入脚本的第一个参数；

- ***$2***: 传入脚本的第二个参数；

- ***$@***: 传入脚本的所有参数；

- ***$****：传入脚本的所有参数；

- ***$$***: 脚本执行的进程id；

- ***$?***: 上一条命令执行后的状态，结果为0表示执行正常，结果为1表示执行异常；

> 其中 ***$@*** 与 ***$\****正常情况下一样，当在脚本中将 ***$\**** 加上双引号作为 ***“$\*”*** 引用时，此时将输入的所有参数当做一个整体字符串对待.



### read

> read 内部命令被用来从标准输入读取单行数据。这个命令可以用来读取键盘输入，当使用重定向的时候，可以读取文件中的一行数据。
>
> - -a 后跟一个变量，该变量会被认为是个数组，然后给其赋值，默认是以空格为分割符。
> - -d 后面跟一个标志符，其实只有其后的第一个字符有用，作为结束的标志。
> - -p 后面跟提示信息，即在输入前打印提示信息。
> - -e 在输入的时候可以使用命令补全功能。
> - -n 后跟一个数字，定义输入文本的长度，很实用。
> - -r 屏蔽\，如果没有该选项，则\作为一个转义字符，有的话 \就是个正常的字符了。
> - -s 安静模式，在输入字符时不再屏幕上显示，例如login时输入密码。
> - -t 后面跟秒数，定义输入字符的等待时间。
> - -u 后面跟fd，从文件描述符中读入，该文件描述符可以是exec新开启的。

