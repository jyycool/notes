# 【Linux】strace



## strace

strace 是 Linux 中常用来跟踪进程执行时的系统调用和所接收的信号。在 Linux 世界，进程不能直接访问硬件设备，当进程需要访问硬件设备( 比如读取磁盘文件，接收网络数据等等)时，必须由用户态模式切换至内核态(kernel)模式，通过系统调用访问硬件设备。

strace可以跟踪到一个进程产生的系统调用, 包括参数，返回值，执行消耗的时间。

### 参数

```java
-c 统计每一系统调用的所执行的时间,次数和出错的次数等. 
-d 输出strace关于标准错误的调试信息. 
-f 跟踪由fork调用所产生的子进程. 
-ff 如果提供-o filename,则所有进程的跟踪结果输出到相应的filename.pid中,pid是各进程的进程号. 
-F 尝试跟踪vfork调用.在-f时,vfork不被跟踪. 
-h 输出简要的帮助信息. 
-i 输出系统调用的入口指针. 
-q 禁止输出关于脱离的消息. 
-r 打印出相对时间关于,,每一个系统调用. 
-t 在输出中的每一行前加上时间信息. 
-tt 在输出中的每一行前加上时间信息,微秒级. 
-ttt 微秒级输出,以秒了表示时间. 
-T 显示每一调用所耗的时间. 
-v 输出所有的系统调用.一些调用关于环境变量,状态,输入输出等调用由于使用频繁,默认不输出. 
-V 输出strace的版本信息. 
-x 以十六进制形式输出非标准字符串 
-xx 所有字符串以十六进制形式输出. 
-a column 
设置返回值的输出位置.默认 为40. 
-e expr 
指定一个表达式,用来控制如何跟踪.格式如下: 
[qualifier=][!]value1[,value2]... 
qualifier只能是 trace,abbrev,verbose,raw,signal,read,write其中之一.value是用来限定的符号或数字.默认的 qualifier是 trace.感叹号是否定符号.例如: 
-eopen等价于 -e trace=open,表示只跟踪open调用.而-etrace!=open表示跟踪除了open以外的其他调用.有两个特殊的符号 all 和 none. 
注意有些shell使用!来执行历史记录里的命令,所以要使用\\. 
-e trace=set 
只跟踪指定的系统 调用.例如:-e trace=open,close,rean,write表示只跟踪这四个系统调用.默认的为set=all. 
-e trace=file 
只跟踪有关文件操作的系统调用. 
-e trace=process 
只跟踪有关进程控制的系统调用. 
-e trace=network 
跟踪与网络有关的所有系统调用. 
-e strace=signal 
跟踪所有与系统信号有关的 系统调用 
-e trace=ipc 
跟踪所有与进程通讯有关的系统调用 
-e abbrev=set 
设定 strace输出的系统调用的结果集.-v 等与 abbrev=none.默认为abbrev=all. 
-e raw=set 
将指 定的系统调用的参数以十六进制显示. 
-e signal=set 
指定跟踪的系统信号.默认为all.如 signal=!SIGIO(或者signal=!io),表示不跟踪SIGIO信号. 
-e read=set 
输出从指定文件中读出 的数据.例如: 
-e read=3,5 
-e write=set 
输出写入到指定文件中的数据. 
-o filename 
将strace的输出写入文件filename 
-p pid 
跟踪指定的进程pid. 
-s strsize 
指定输出的字符串的最大长度.默认为32.文件名一直全部输出. 
-u username 
以username 的UID和GID执行被跟踪的命令
```

**通用的完整用法：**

1. 跟踪某个进程[pid], 并将结果输出到 output.txt

   ```sh
   strace -o output.txt -T -tt -e trace=all -p 28979
   ```

2. 直接跟踪某个启动命令[java], 输出结果到 output

   ```sh
   strace -ff -o output ${JAVA_HOME}/java TestDemo
   ```

## dtruss

Mac中的strace命令

### 参数

```sh
USAGE: dtruss [-acdefholLs] [-t syscall] { -p PID | -n name | command | -W name }

          -p PID          # examine this PID
          -n name         # examine this process name
          -t syscall      # examine this syscall only
          -W name         # wait for a process matching this name
          -a              # print all details
          -c              # print syscall counts
          -d              # print relative times (us)
          -e              # print elapsed times (us)
          -f              # follow children
          -l              # force printing pid/lwpid
          -o              # print on cpu times
          -s              # print stack backtraces
          -L              # don't print pid/lwpid
          -b bufsize      # dynamic variable buf size
   eg,
       dtruss df -h       # run and examine "df -h"
       dtruss -p 1871     # examine PID 1871
       dtruss -n tar      # examine all processes called "tar"
       dtruss -f test.sh  # run test.sh and follow children
```

