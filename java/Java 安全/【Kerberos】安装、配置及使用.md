# 【Kerberos】安装、配置及使用

在 Mac 上面使用 kerberos 比较麻烦，Mac 本身已经安装了 kerberos，在命令行界面敲下 kinit, 会得到如下提示:

```sh
# sherlock @ mbp in ~ [13:27:42]
$ kadmin
kadmin: kadm5_init_with_password: Configuration file does not specify default realm
```

提示: 配置文件中未指定 default realm, 那么配置文件在哪? Google 的说法不一, 有的说是 /etc/krb5.conf, 有的说是 /Library/Preferences/edu.mit.Kerberos, 还有说在 ~/Library/Preferences/edu.mit.Kerberos。

但是在 Mac OS BugSur 上我在上面三个路径下都没找到配置文件, 但可以确定的 Mac 确实安装了 Kerberos, 因为它对外提供了 kerberos 的服务, 并且服务在 /System/Library/LaunchDaemons/ 下:

```sh
# sherlock @ mbp in /System/Library/LaunchDaemons [13:47:27]
$ ll
total 2992
......
-rw-r--r--  1 root  wheel   526B  1  1  2020 com.apple.Kerberos.digest-service.plist
-rw-r--r--  1 root  wheel   586B  1  1  2020 com.apple.Kerberos.kadmind.plist
-rw-r--r--  1 root  wheel   578B  1  1  2020 com.apple.Kerberos.kcm.plist
-rw-r--r--  1 root  wheel   894B  1  1  2020 com.apple.Kerberos.kdc.plist
-rw-r--r--  1 root  wheel   588B  1  1  2020 com.apple.Kerberos.kpasswdd.plist
......
```

于是, 先试着配置 /etc/krb5.conf



