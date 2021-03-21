## 【Hue】MacOS安装Hue

想在本地安装 HUE, 可视化 Hadoop, Hive, HBase 等工具

踩了好多的坑

## 准备

根据[官网](https://gethue.com/start-developing-hue-on-a-mac-in-a-few-minutes/)的安装教程, 需要准备一下几个组件

1. homebrew - Mac已自带, 升级换下国内镜像源

2. maven - 依赖管理, 安装很 easy

3. MySQL - 已安装

4. gmp -  使用命令 `brew install gmp` 安装

5. python2.7 - 强烈建议使用 pyenv-virtualenv 安装 Python-2.7-dev 这个版本的虚拟环境 (先提前安装配置好 pyenv-virtualenv)来编译 Hue, 这样不会污染系统中的 Python环境。


## Hue 编译

1. clone 项目不多说了, 速度慢的直接下载 zip 包

   `git clone https://github.com/cloudera/hue.git`

2. 在 hue 根目录下设置 python 环境并验证(使用虚拟环境 hue)

   `pyenv virtualenvs`

   ```sh
   ➜  hue git:(master) ✗ pyenv virtualenvs
     2.7-dev/envs/hue (created from /usr/local/var/pyenv/versions/2.7-dev)
   * hue (created from /usr/local/var/pyenv/versions/2.7-dev)
   ```
   
3. 开始编译

   `make apps`

   > 踩坑开始, 所有的问题都在这个过程里:
   >
   > 1. 编译报错: "Error: must have python development packages for 2.6 or 2.7. Could not find Python.h. Please install python2.6-devel or python2.7-devel"。 Stop。
   >
   >    这是 python 版本的原因, 不建议使用 Mac 自带的 python2.7, 而且最近 homebrew 也移除了 python@2 的安装, 所以最靠谱的方式是使用 pyenv 来安装管理 python。
   >
   > 2. 编译报错: build/temp.macosx-11.2-x86_64-2.7/_openssl.c:575:10: fatal error: 'openssl/opensslv.h' file not found
   >
   >    首先查看是否安装了 openssl, 如果没有安装: `brew install openssl` 安装一下。
   >
   >    ```sh
   >    # 查看是否安装了 openssl
   >    $ brew list
   >    
   >    # 安装 openssl
   >    $ brew install openssl
   >    ```
   >
   >    如果安装了 openssl 还是报错, 那么就是环境变量的问题了，需要重新指定`openssl`的路径安装：
   >
   >    ```sh
   >    ➜  hue git:(master) ✗ brew link openssl --force
   >    Warning: Refusing to link macOS provided/shadowed software: openssl@1.1
   >    If you need to have openssl@1.1 first in your PATH, run:
   >      echo 'export PATH="/usr/local/opt/openssl@1.1/bin:$PATH"' >> ~/.zshrc
   >    
   >    For compilers to find openssl@1.1 you may need to set:
   >      export LDFLAGS="-L/usr/local/opt/openssl@1.1/lib"
   >      export CPPFLAGS="-I/usr/local/opt/openssl@1.1/include"
   >    
   >    For pkg-config to find openssl@1.1 you may need to set:
   >      export PKG_CONFIG_PATH="/usr/local/opt/openssl@1.1/lib/pkgconfig"
   >    
   >    # 然后根据命令行的提示设置环境变量, 在 ~/.bash_profile 中配置
   >    #======================
   >    # Openssl
   >    #======================
   >    export PATH="/usr/local/opt/openssl/bin:$PATH"
   >    export LDFLAGS="-L/usr/local/opt/openssl/lib"
   >    export CPPFLAGS="-I/usr/local/opt/openssl/include"
   >    export PKG_CONFIG_PATH="/usr/local/opt/openssl/lib/pkgconfig"
   >    
   >    # 保存退出 vim
   >    ➜  hue git:(master) ✗ source ~/.zshrc
   >    
   >    # 然后验证下当前 openssl, 这就 OK 了。
   >    ➜  hue git:(master) ✗ openssl version
   >    OpenSSL 1.1.1i  8 Dec 2020
   >    ```
   >
   > 3. 编译报错: Modules/LDAPObject.c:18:10: fatal error: 'sasl.h' file not found
   >    #include <sasl.h>
   >             ^~~~~~~~
   >    1 error generated.
   >    
   >    大致知道是缺少 'sasl.h', google 后找到的解决方法:
   >    
   >```sh
   >    # 查看 sdk的路径
   >➜  ~ xcrun --show-sdk-path
   >    /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk
   >
   >    # 将这个路径 $(xcrun --show-sdk-path)/usr/include 添加到上一步的环境变量 CPPFLAGS 中
   >    export CPPFLAGS="-I/usr/local/opt/openssl/include -I/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/sasl"
   >    ```
   >    
   

至此，`make apps` 操作已经全部完成。 

## Hue 配置

