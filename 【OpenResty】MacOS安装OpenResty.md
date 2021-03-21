# 【OpenResty】MacOS安装OpenResty

#openresty

## OpenResty 介绍

OpenResty(又称：ngx_openresty) 是一个基于 NGINX 的可伸缩的 Web 平台，由中国人章亦春发起，提供了很多高质量的第三方模块。

OpenResty 是一个强大的 Web 应用服务器，Web 开发人员可以使用 Lua 脚本语言调动 Nginx 支持的各种 C 以及 Lua 模块,更主要的是在性能方面，OpenResty可以 快速构造出足以胜任 10K 以上并发连接响应的超高性能 Web 应用系统。

360，UPYUN，阿里云，新浪，腾讯网，去哪儿网，酷狗音乐等都是 OpenResty 的深度用户。

## MacOSX(macOS)用户安装

对于 MacOSX 或 macOS 用户，强烈推荐您使用 [homebrew](https://brew.sh/) 包管理工具安装 OpenResty。可以直接使用下面 这一条命令：

```bash
brew install openresty/brew/openresty
```

如果你之前是从 `homebrew/nginx` 安装的 OpenResty，请先执行：

```bash
brew untap homebrew/nginx
```

推荐您使用一些软件管理工具先安装PCRE, 比如说 [Homebrew](http://mxcl.github.com/homebrew/):

```bash
brew update
brew install pcre openssl
```

当然了，您也可以直接通过代码安装 PCRE 和 OpenSSL.

安装好 PCRE 和 OpenSSL 之后，可以使用下面的命令进行安装：

```bash
$ ./configure \
   --with-cc-opt="-I/usr/local/opt/openssl/include/ -I/usr/local/opt/pcre/include/" \
   --with-ld-opt="-L/usr/local/opt/openssl/lib/ -L/usr/local/opt/pcre/lib/" \
   -j8
```

在 MacOS 上 OpenResty 默认安装位于 `/usr/local/Cellar/openresty/1.17.8.2_1`

## 准备nginx.conf配置文件

创建一个简单的纯文本文件，`~/devTools/conf/nginx.conf`其中包含以下内容：

```conf
worker_processes  1;
error_log /Users/sherlock/devTools/log/error.log;
events {
    worker_connections 1024;
}
http {
    server {
        listen 9090;
        location / {
            default_type text/html;
            content_by_lua_block {
                ngx.say("<p>hello, world</p>")
            }
        }
    }
}
```

### 创建 nginx 软连接

```shell
ln -s /usr/local/Cellar/openresty/1.17.8.2_1/nginx/sbin/nginx /usr/local/bin/nginx
```

### 启动 Nginx 服务器

我们以这种方式使用配置文件启动nginx服务器：

```
nginx -p `pwd`/ -c ~/devTools/conf/nginx.conf
```

错误消息将发送到stderr设备或 `~/devTools/log/error.log`当前工作目录中的默认错误日志文件。

### 访问我们的HelloWorld Web服务

我们可以使用curl访问名为HelloWorld的新Web服务：

```
curl http://localhost:9090/
```

如果一切正常，我们应该得到输出

```
<p>hello, world</p>
```

您一定可以将自己喜欢的Web浏览器指向该位置`http://localhost:9090/`。

### 停止 Nginx 服务器

1. 快速停止

   ```sh
   nginx -s stop
   ```

2. 完整有序停止

   ```sh
   nginx -s quit
   ```

3. 重启

   ```sh
   nginx -s reload
   ```

   

