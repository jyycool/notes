# 【Linux】ps

运行在系统上的程序, 我们称之为进程(process)。想监测这些进程, 需要熟悉 ps 命令的用法。它能输出运行在系统上的所有程序的许多信息。

但是随着它功能强大而来的还有它的复杂性--极其多的参数, 但是在一般情况下, ps 命令并不会提供那么多的信息。

```sh
➜  ~ ps
  PID TTY           TIME CMD
 2350 ttys000    0:00.06 /Applications/iTerm.app/Contents/MacOS/iTerm2 --server login -fp sherlock
 2354 ttys000    0:00.36 -zsh
 2352 ttys001    0:00.06 /Applications/iTerm.app/Contents/MacOS/iTerm2 --server /usr/bin/login -fpl sherlock /Applicati
 2355 ttys001    0:00.12 -zsh
```

默认情况下, ps 命令只会显示运行在当前控制台下属于当前用户的进程。

上例中的基本输出显示了程序的进程ID(PID)、它们运行在哪个终端(TTY)、进程已使用 CPU 的时间(TIME)以及启动进程的目录(CMD)。

Linux 系统中使用的 GNU ps 命令支持 3 种不同类型的命令参数:

- Unix 风格的参数, 前面加单破折号("-"), e.g: ps -A
- BSD 风格的参数, 前面不加破折号号, e.g: ps aT
- GNU 风格的长参数, 前面加双破折号("--"), e.g: ps --format

## 参数

### 1. Unix 风格的参数

Unix 风格的参数是从贝尔实验室开发的 AT&T Unix 系统上的原有 ps 命令继承下来的。

下表: Unix风格的 ps命令参数

| 参数        | 描述                                               |
| ----------- | -------------------------------------------------- |
| -A          | 显示所有进程                                       |
| -N          | 显示与指定参数不符的所有进程                       |
| -a          | 显示除控制进程和无终端进程外的所有进程             |
| -d          | 显示除控制进程外的所有进程                         |
| -e          | 显示所有进程                                       |
| -C cmdlist  | 显示包含在 cmdlist 列表中的进程                    |
| -G grplist  | 显示组 ID 在 grplist 列表中的进程                  |
| -U userlist | 显示属主的用户 ID 在 userlist 列表中的进程         |
| -g grplist  | 显示会话或组 ID 在 grplist 列表中的进程            |
| -p pidlist  | 显示 PID 在 pidlist 列表中的进程                   |
| -s sesslist | 显示会话 ID 在 sesslist 列表中的进程               |
| -t ttylist  | 显示终端 ID 在 ttylist 列表中的进程                |
| -u userlist | 显示有效用户 ID 在 userlist 列表中的进程           |
| -F          | 显示更多额外输出(相对 -f 参数而言)                 |
| -O format   | 显示默认的输出列以及 format 列表指定的特定列       |
| -M          | 显示进程的安全信息                                 |
| -c          | 显示进程的额外调度器信息                           |
| -f          | 显示完整格式的输出信息                             |
| -j          | 显示任务信息                                       |
| -l          | 显示长列表                                         |
| -o format   | 仅显示由 format 指定的列                           |
| -Y          | 不要显示进程标记(process flag, 表名进程状态的标记) |
| -Z          | 显示安全标签(security context 信息)                |
| -H          | 用层级格式来显示进程(树状, 用来显示父进程)         |
| -n namelist | 定义了 WCHAN 列显示的值                            |
| -w          | 采用了宽输出模式, 不限宽度显示                     |
| -L          | 显示进程中的线程                                   |
| -V          | 显示 ps 命令的版本号                               |

使用 ps 命令不在于记住所有参数, 而在于记住最有用的参数。大多数时候我们都将一组参数配合使用来提取有用的进程信息。例如, 你想查看系统运行的所有进程, 可以用 -ef 的参数组合(ps 命令允许像这样把参数组合在一起)

```sh
➜  ~ ps -ef
UID   PID    PPID C  STIME   TTY          TIME     CMD
205   226     1   0  7:40上午 ??         0:00.10 /usr/sbin/distnoted agent
  205   227     1   0  7:40上午 ??         0:00.10 /usr/libexec/secinitd
  205   228     1   0  7:40上午 ??         0:00.02 /usr/sbin/cfprefsd agent
  205   229     1   0  7:40上午 ??         0:00.23 /usr/libexec/trustd --agent
  205   231     1   0  7:40上午 ??         0:00.14 /usr/libexec/containermanagerd --runmode=agent --bundle-container-mode=global --bundle-container-owner=_appinstalld --system-container-mode=none
    0   232     1   0  7:40上午 ??         0:09.06 /usr/libexec/mobileassetd
    0   233     1   0  7:40上午 ??         0:00.02 /usr/libexec/tzd
    0   237     1   0  7:40上午 ??         0:04.41 /usr/libexec/ApplicationFirewall/socketfilterfw
    0   292     1   0  7:40上午 ??         0:00.03 /usr/libexec/smd
```

参数 -e 指定显示所有运行在该系统上的进程; 参数 -f 扩展了这些输出, 这些扩展的列包含了有用的信息:

- UID 启动这些进程的用户
- PID 进程的进程 ID 号
- PPID 父进程的进程号(如果该进程由另一个进程启动)
- C 进程生命周期内的 CPU 利用率
- STIME 进程启动时的系统时间
- TTY 进程启动时的终端设备
- TIME 进程已经使用的 CPU 时间
- CMD 启动进程的名称

如果想要获得更多的信息, 可以采用 -l 参数, 它会产生一个长格式的输出。

```sh
➜  ~ ps -l
  UID   PID  PPID        F CPU PRI NI       SZ    RSS WCHAN     S             ADDR TTY           TIME CMD
  501  2350   709     4006   0  31  0  4306984   6048 -      Ss                  0 ttys000    0:00.06 /Applica
  501  2354  2351     4006   0  31  0  4298116   3032 -      S                   0 ttys000    0:00.85 -zsh
  501  2352   709     4006   0  31  0  4315176   6056 -      Ss                  0 ttys001    0:00.06 /Applica
  501  2355  2353     4006   0  31  0  4297496    948 -      S+                  0 ttys001    0:00.12 -zsh
```

多出来的那几列:

- F 内核分配给进程的系统标记
- S 进程的状态(O 代表正在运行; S 代表正在休眠; R 代表可运行, 正等待运行; Z 代表僵化, 进程已结束但父进程已不存在; T 代表停止)
- PRI 进程的优先级(数字越大代表优先级越低)
- NI 谦让度值用来参数决定优先级
- ADDR 进程的内存地址
- SZ  加入进程被换出, 所需交换空间的大致大小
- WCHAN 休眠进程的内核函数的地址

### 2. BSD 风格的参数

它是伯克利软件发行版本是加州大学伯克利分校开发的一个 Unix 版本。它和 AT&TUnix 系统有许多细小的不同。

BSD 风格的 `ps` 命令参数

| 参数         | 描述                                                    |
| :----------- | :------------------------------------------------------ |
| T            | 显示跟当前终端关联的所有进程                            |
| a            | 显示跟任意终端关联的所有进程                            |
| g            | 显示所有的进程，包括控制进程                            |
| r            | 仅显示运行中的进程                                      |
| x            | 显示所有的进程，甚至包括未分配任何终端的进程            |
| U *userlist* | 显示归 *`userlist`* 列表中某用户ID所有的进程            |
| p *pidlist*  | 显示PID在 *`pidlist`* 列表中的进程                      |
| t *ttylist*  | 显示所关联的终端在 *`ttylist`* 列表中的进程             |
| O *format*   | 除了默认输出的列之外，还输出由 *`format`* 指定的列      |
| X            | 按过去的Linux i386寄存器格式显示                        |
| Z            | 将安全信息添加到输出中                                  |
| j            | 显示任务信息                                            |
| l            | 采用长模式                                              |
| o *format*   | 仅显示由 *`format`* 指定的列                            |
| s            | 采用信号格式显示                                        |
| u            | 采用基于用户的格式显示                                  |
| v            | 采用虚拟内存格式显示                                    |
| N *namelist* | 定义在 `WCHAN` 列中使用的值                             |
| O *order*    | 定义显示信息列的顺序                                    |
| S            | 将数值信息从子进程加到父进程上，比如CPU和内存的使用情况 |
| c            | 显示真实的命令名称（用以启动进程的程序名称）            |
| e            | 显示命令使用的环境变量                                  |
| f            | 用分层格式来显示进程，表明哪些进程启动了哪些进程        |
| h            | 不显示头信息                                            |
| k *sort*     | 指定用以将输出排序的列                                  |
| n            | 和 `WCHAN` 信息一起显示出来，用数值来表示用户ID和组ID   |
| w            | 为较宽屏幕显示宽输出                                    |
| H            | 将线程按进程来显示                                      |
| m            | 在进程后显示线程                                        |
| L            | 列出所有格式指定符                                      |
| V            | 显示 `ps` 命令的版本号                                  |

大多数情况下，你只要选择自己所喜欢格式的参数类型就行了（比如你在使用 Linux 之前就已经习惯 BSD 环境了）。

在使用 BSD 参数时，`ps` 命令会自动改变输出以模仿 BSD 格式。

```sh
➜  ~ ps l
  UID   PID  PPID CPU PRI NI      VSZ    RSS WCHAN  STAT   TT       TIME COMMAND
  501  2350   709   0  31  0  4306984   6048 -      Ss   s000    0:00.06 /Applications/iTerm.app/Contents/MacO
  501  2354  2351   0  31  0  4298116   3104 -      S    s000    0:00.89 -zsh
  501  2352   709   0  31  0  4315176   6056 -      Ss   s001    0:00.06 /Applications/iTerm.app/Contents/MacO
  501  2355  2353   0  31  0  4297496    948 -      S+   s001    0:00.12 -zsh
```

其中大部分的输出列跟使用 Unix 风格参数是一样的，只有一小部分不同。

- VSZ 进程在内存中的大小，以千字节（KB）为单位。
- RSS 进程在未换出时占用的物理内存。
- STAT 代表当前进程状态的双字符状态码。

许多系统管理员都喜欢 BSD 风格的 `l` 参数。它能输出更详细的进程状态码（STAT列）。双字符状态码能比 Unix 风格输出的单字符状态码更清楚地表示进程的当前状态。

第一个字符采用了和 Unix 风格 `S` 列相同的值，表明进程是在休眠、运行还是等待。

- `O` 代表正在运行
- `S` 代表正在休眠
- `R` 代表可运行, 正等待运行
- `Z` 代表僵化, 进程已结束但父进程已不存在
- `T` 代表停止

第二个参数进一步说明进程的状态。

- `<` ：该进程运行在高优先级上。
- `N` ：该进程运行在低优先级上。
- `L` ：该进程有页面锁定在内存中。
- `s` ：该进程是控制进程。
- `l` ：该进程是多线程的。
- `+` ：该进程运行在前台。

从前面的例子可以看出，`bash` 命令处于休眠状态，但同时它也是一个控制进程（在我的会话中，它是主要进程），而 `ps` 命令则运行在系统的前台。

### 3. GNU 风格的长参数

GNU 开发人员在新改进过的 `ps` 命令中加入了另外一些参数。其中一些 GNU 长参数复制了现有的 Unix 或 BSD 类型的参数，而另一些则提供了新功能。下表列出了现有的 GNU 长参数。

| 参数              | 描述                                       |
| :---------------- | :----------------------------------------- |
| --deselect        | 显示所有进程，命令行中列出的进程           |
| --Group *grplist* | 显示组ID在 *`grplist`* 列表中的进程        |
| --User *userlist* | 显示用户ID在 *`userlist`* 列表中的进程     |
| --group *grplist* | 显示有效组ID在 *`grplist`* 列表中的进程    |
| --pid *pidlist*   | 显示PID在 *`pidlist`* 列表中的进程         |
| --ppid *pidlist*  | 显示父PID在 *`pidlist`* 列表中的进程       |
| --sid *sidlist*   | 显示会话ID在 *`sidlist`* 列表中的进程      |
| --tty *ttylist*   | 显示终端设备号在 *`ttylist`* 列表中的进程  |
| --user *userlist* | 显示有效用户ID在 *`userlist`* 列表中的进程 |
| --format *format* | 仅显示由 *`format`* 指定的列               |
| --context         | 显示额外的安全信息                         |
| --cols *n*        | 将屏幕宽度设置为 *`n`* 列                  |
| --columns *n*     | 将屏幕宽度设置为 *`n`* 列                  |
| --cumulative      | 包含已停止的子进程的信息                   |
| --forest          | 用层级结构显示出进程和父进程之间的关系     |
| --headers         | 在每页输出中都显示列的头                   |
| --no-headers      | 不显示列的头                               |
| --lines *n*       | 将屏幕高度设为 *`n`* 行                    |
| --rows *n*        | 将屏幕高度设为 *`n`* 排                    |
| --sort *order*    | 指定将输出按哪列排序                       |
| --width *n*       | 将屏幕宽度设为 *`n`* 列                    |
| --help            | 显示帮助信息                               |
| --info            | 显示调试信息                               |
| --version         | 显示 `ps` 命令的版本号                     |

可以将 GNU 长参数和 Unix 或 BSD 风格的参数混用来定制输出。GNU 长参数中一个着实让人喜爱的功能就是 `--forest` 参数。它会显示进程的层级信息，并用 ASCII 字符绘出可爱的图表。

```sh
1981 ?        00:00:00 sshd
3078 ?        00:00:00  \_ sshd
3080 ?        00:00:00      \_ sshd
3081 pts/0    00:00:00          \_ bash
16676 pts/0    00:00:00              \_ ps
```

这种格式让跟踪子进程和父进程变得十分容易。

