# 【Python】MacOS 下的 Python



Mac 上的 Python环境很混乱。Mac 系统有自己的 Python, 再加上用户安装的 Python, Anaconda 的 Python, 简直是一锅粥的乱。

## 一、Python 环境

### 1.1 MacOS 系统的 Python 环境

Mac 系统使用的 Python: `/System/Library/Frameworks/Python.framework/Versions`, 不建议删除或修改, 后面最好把这个环境和自己的 Python 环境隔离开。

这个 Python 的安装目录一般在:

```sh
➜  Versions pwd
/System/Library/Frameworks/Python.framework/Versions
➜  Versions ll
total 0
lrwxr-xr-x   1 root  wheel     3B  1  1  2020 2.3 -> 2.7
lrwxr-xr-x   1 root  wheel     3B  1  1  2020 2.5 -> 2.7
lrwxr-xr-x   1 root  wheel     3B  1  1  2020 2.6 -> 2.7
drwxr-xr-x  11 root  wheel   352B  1  1  2020 2.7
lrwxr-xr-x   1 root  wheel     3B  1  1  2020 Current -> 2.7
```

在 `/usr/bin` 下的 `python`, `python2`, `python2.7`, `pythonw`, `pythonw2.7` 都指向系统的 Python, 所以这些软连接都别去改动。

```sh
➜  bin pwd
/usr/bin
➜  bin ll
lrwxr-xr-x  1 root   wheel    75B  1  1  2020 python -> ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7
lrwxr-xr-x  1 root   wheel    75B  1  1  2020 python2 -> ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7
lrwxr-xr-x  1 root   wheel    75B  1  1  2020 python2.7 -> ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7
-rwxr-xr-x  1 root   wheel   134K  1  1  2020 python3
lrwxr-xr-x  1 root   wheel    76B  1  1  2020 pythonw -> ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/pythonw2.7
lrwxr-xr-x  1 root   wheel    76B  1  1  2020 pythonw2.7 -> ../../System/Library/Frameworks/Python.framework/Versions/2.7/bin/pythonw2.7
```

### 1.2 pkg 包安装的 Python 环境

Mac 用户安装的 Python: `/Library/Frameworks/Python.framework/Versions`, 这个是官网下载的 pkg 包安装的位置。

```sh
➜  Versions pwd
/Library/Frameworks/Python.framework/Versions
➜  Versions ll
total 0
drwxrwxr-x  12 root  admin   384B  2 23 10:22 2.7
drwxrwxr-x   2 root  admin    64B  6 24  2020 3.5
lrwxr-xr-x   1 root  wheel     3B  2 23 10:22 Current -> 2.7

```



```sh
➜  bin pwd
/usr/local/bin
➜  bin ll
lrwxr-xr-x  1 root      wheel    68B  2 23 10:22 python -> ../../../Library/Frameworks/Python.framework/Versions/2.7/bin/python
lrwxr-xr-x  1 root      wheel    69B  2 23 10:22 python2 -> ../../../Library/Frameworks/Python.framework/Versions/2.7/bin/python2
lrwxr-xr-x  1 root      wheel    71B  2 23 10:22 python2.7 -> ../../../Library/Frameworks/Python.framework/Versions/2.7/bin/python2.7
lrwxr-xr-x  1 root      wheel    69B  2 23 10:22 pythonw -> ../../../Library/Frameworks/Python.framework/Versions/2.7/bin/pythonw
lrwxr-xr-x  1 root      wheel    70B  2 23 10:22 pythonw2 -> ../../../Library/Frameworks/Python.framework/Versions/2.7/bin/pythonw2
lrwxr-xr-x  1 root      wheel    72B  2 23 10:22 pythonw2.7 -> ../../../Library/Frameworks/Python.framework/Versions/2.7/bin/pythonw2.7
```

在 `/usr/local/bin` 下的 `python`, `python2`, `python2.7`, `pythonw`, `pythonw2`, `pythonw2.7`, 都指向用户安装的 Python。

### 1.3 brew 安装的 Python 环境

Mac 上还有一种便捷的 Python 安装方式: 

```sh 
➜  ~ brew search python
==> Formulae
app-engine-python                      micropython                            python@3.8
boost-python                           pr0d1r2/python2/python@2.7.17          python@3.9
boost-python3                          ptpython                               reorder-python-imports
bpython                                python-markdown                        wxpython
gst-python                             python-yq
ipython                                python@3.7
==> Casks
awips-python                           kk7ds-python-runtime                   mysql-connector-python

If you meant "python" specifically:
It was migrated from homebrew/cask to homebrew/core.

➜  ~ brew install python@3.8
Updating Homebrew...
==> Downloading https://mirrors.ustc.edu.cn/homebrew-bottles/bottles/python%403.8-3.8.7_2.big_sur.bottle.tar.gz
######################################################################## 100.0%
==> Pouring python@3.8-3.8.7_2.big_sur.bottle.tar.gz
==> /usr/local/Cellar/python@3.8/3.8.7_2/bin/python3 -s setup.py --no-user-cfg install --force --verbose --install-s
==> /usr/local/Cellar/python@3.8/3.8.7_2/bin/python3 -s setup.py --no-user-cfg install --force --verbose --install-s
==> /usr/local/Cellar/python@3.8/3.8.7_2/bin/python3 -s setup.py --no-user-cfg install --force --verbose --install-s
==> Caveats
Python has been installed as
  /usr/local/opt/python@3.8/bin/python3

Unversioned symlinks `python`, `python-config`, `pip` etc. pointing to
`python3`, `python3-config`, `pip3` etc., respectively, have been installed into
  /usr/local/opt/python@3.8/libexec/bin

You can install Python packages with
  /usr/local/opt/python@3.8/bin/pip3 install <package>
They will install into the site-package directory
  /usr/local/lib/python3.8/site-packages

See: https://docs.brew.sh/Homebrew-and-Python

python@3.8 is keg-only, which means it was not symlinked into /usr/local,
because this is an alternate version of another formula.

If you need to have python@3.8 first in your PATH, run:
  echo 'export PATH="/usr/local/opt/python@3.8/bin:$PATH"' >> ~/.zshrc

For compilers to find python@3.8 you may need to set:
  export LDFLAGS="-L/usr/local/opt/python@3.8/lib"

For pkg-config to find python@3.8 you may need to set:
  export PKG_CONFIG_PATH="/usr/local/opt/python@3.8/lib/pkgconfig"

==> Summary
🍺  /usr/local/Cellar/python@3.8/3.8.7_2: 4,498 files, 72.6MB
```

上面示例是使用 brew 安装 python@3.8, 最后提示了安装的目录: `/usr/local/Cellar/python@3.8/3.8.7_2`, 但是这还不是最优的安装管理 python 方式

### 1.4 pyenv 和 virtualenv 安装的 Python 环境

#### 1.4.1 为什么要使用 pyenv?

Python 有两大迭代版本 2.7 和 3, 一些旧的框架都是基于 Python-2.7.x 来开发的, 而新项目大都基于 Python-3.x.x 来开发的, 所以我们本地都需要至少两个版本的 Python 来管理项目。

pyenv 可以帮助你在一台开发机上建立多个版本的 Python 环境， 并提供方便的切换方法。

#### 1.4.2 为什么又需要 virtualenv?

virtualenv 可以搭建虚拟且独立的 Python 环境，可以使每个项目环境与其他项目独立开来，保持环境的干净，解决包冲突问题。举例说明。

首先我们可以用 pyenv 安装多个 Python 版本， 比如安装了 2.7, 3.7 版本。 用户可以随意切换当前默认的 Python 版本。此时，每个版本的环境仍是唯一且干净的。如果我们在环境中安装一些库的话，会导致这个版本的环境被污染。

但如果我们用 virtualenv 去建立虚拟环境，就可以完全保证 Python 系统路径的干净。无论你在虚拟环境中安装了什么程序， 都不会影响已安装版本的 Python 系统环境。

简单说 virtualenv 主要是用来管理不同项目的依赖冲突的问题。

## 二、pyenv

首先必须明白的是: pyenv 只会管理通过 pyenv 安装的 Python 版本，你自己在官网上下载直接安装的 Python, 系统自带的 Python, brew 安装的 Python 都不能被 pyenv 管理。

### 2.1 pyenv 安装配置 

```sh
➜  ~ brew install pyenv
Updating Homebrew...
Warning: pyenv 1.2.22 is already installed and up-to-date.
To reinstall 1.2.22, run:
  brew reinstall pyenv
```

使用 brew 安装 pyenv 的路径: `/usr/local/Cellar/pyenv`

```sh
➜  pyenv pwd
/usr/local/Cellar/pyenv
➜  pyenv ll
total 0
drwxr-xr-x  19 sherlock  staff   608B  2 23 10:52 1.2.22
```

配置 pyenv, 在 ~/.bash_profile中添加如下配置:

```sh
export PYENV_ROOT=/usr/local/var/pyenv
if which pyenv > /dev/null; then eval "$(pyenv init -)"; fi
```

> 特别说明下: 
>
> 使用 brew 方式安装的 pyenv 并不自带 plugin, 比如就没 pyenv-virtualenv。如果要使用 pyenv-virtualenv 还需要手动使用 brew install pyenv-virtualenv 安装。
>
> 使用官方推荐的安装包安装 pyenv: `curl -L https://github.com/pyenv/pyenv-installer/raw/master/bin/pyenv-installer | bash` 这种方式安装的 pyenv 会提供如下工具：
>
> - `pyenv`: pyenv 工具自身
> - `pyenv-virtualenv`: pyenv 的插件可以用来管理 vierual environments
> - `pyenv-update`: 用来更新 pyenv 的插件
> - `pyenv-doctor`: 验证 pyenv 和依赖是否安装的插件
> - `pyenv-which-ext`: 用来寻找相同命令的插件

### 2.2 pyenv 使用

1. 列出可用 Python 版本, 包含了当下所有市面上的 Python

   ```sh
   ➜  ~ pyenv install --list
   Available versions:
     ...
     2.7.18
     ...
     3.10-dev
     ...
     activepython-3.6.0
     ...
     anaconda3-2020.11
    	...
     jython-2.7.2
     ...
     micropython-1.13
     ...
     miniconda3-4.7.12
     ...
     pypy3.6-7.3.1
     ...
     pyston-0.6.1
    	...
     stackless-3.7.5
   ```

2. 安装指定版本的 python, -v 参数可选

   ```sh
   ➜  ~ pyenv install [-v] 2.7-dev
   ......
   Installed Python-2.7-dev to /usr/local/var/pyenv/versions/2.7-dev
   ```

3. 查看当前使用的 Python 版本, 带有 * 号的就是当前正在使用的 Python 版本

   ```sh
   ➜  ~ pyenv versions
    * system (set by /usr/local/var/pyenv/version)
       2.7.8
       2.7.10
   ```

4. 卸载指定版本的 python

   ```sh
   ➜  ~ pyenv uninstall 2.7-dev
   ```

5. 指定全局 python 环境, 不推荐这么做, 对 MacOS 可能会有影响

   通过将版本号写入 `${PYENV_ROOT}/version` 文件的方式。

   ```sh
   ➜  ~ pyenv global 2.7-dev
   ```

6. 指定目录级 python 环境, 仅对指定文件夹有效(推荐使用)

   通过将版本号写入当前目录下的 `.python-version` 文件的方式设置。该方式设置的 Python 版本优先级比 global 高。

   ```sh
   ➜  test pyenv local 2.7-dev
   ➜  test la
   total 8
   drwxr-xr-x    3 sherlock  staff    96B  2 23 23:20 .
   drwxr-xr-x+ 121 sherlock  staff   3.8K  2 23 23:20 ..
   -rw-r--r--    1 sherlock  staff     8B  2 23 23:20 .python-version
   ```

7. 指定当前 shell 的 python 环境(编译项目时可以这么做)

   ```sh
   ➜  ~ pyenv shell 2.7-dev 
   ```

8. 取消 pyenv 设定的 Python 环境, 使用 --unset 参数

   ```sh
   ➜  ~ pyenv global --unset 
   ➜  ~ pyenv local --unset
   ➜  ~ pyenv shell --unset
   ```

更详细请查看[官网](https://github.com/pyenv/pyenv/blob/master/COMMANDS.md#command-reference)

## 三、pyenv-virtualenv

### 3.1 pyenv-virtualenv 安装配置

```sh
➜  ~ brew install pyenv-virtualenv
Updating Homebrew...
Warning: pyenv-virtualenv 1.1.5 is already installed and up-to-date.
To reinstall 1.1.5, run:
  brew reinstall pyenv-virtualenv
```

使用 brew 安装 pyenv-virtualenv 的路径: `/usr/local/Cellar/pyenv-virtualenv`

```sh
➜  pyenv-virtualenv pwd
/usr/local/Cellar/pyenv-virtualenv
➜  pyenv-virtualenv ll
total 0
drwxr-xr-x  11 sherlock  admin   352B  2 23 21:45 1.1.5
```

配置 pyenv-virtualenv, 在 ~/.bash_profile中添加如下配置:

```sh
if which pyenv-virtualenv-init > /dev/null; then eval "$(pyenv virtualenv-init -)"; fi
```

为这个命令取别名:

```sh
alias pye='/usr/local/bin/pyenv'
alias pye-vt='/usr/local/bin/pyenv-virtualenv'
```

### 3.2 pyenv-virtualenv 使用

1. 创建虚拟环境

   `pyenv virtualenv <version> <folder_name>`

   ```sh
   ➜  ~ pyenv virtualenv 2.7-dev hue
   ```

   >Tip：
   >1. 创建虚拟环境的时候，所指定的 Python 版本号必须是已经安装过的 Python
   >2. 'folder_name'这个是根据个人的习惯而命名的。我看很多资料发现大多数人都是'venv-2.7.10'这种方式，我个人习惯是把文件名这样命名'project_name-venv'

2. 列出当前所有的虚拟环境

   `pyenv virtualenvs`

   ```sh
   ➜  ~ pyenv virtualenvs
     2.7-dev/envs/hue (created from /usr/local/var/pyenv/versions/2.7-dev)
     hue (created from /usr/local/var/pyenv/versions/2.7-dev)
   ```

3. 切换虚拟环境：

   `pyenv activate <folder_name>`

   ```sh
   ➜  test pyenv activate hue
   pyenv-virtualenv: prompt changing will be removed from future release. configure `export PYENV_VIRTUALENV_DISABLE_PROMPT=1' to simulate the behavior.
   (hue) ➜  test 
   ```

   接下来我们的 Python 环境就已经切换到名为 hue 的虚拟环境了

4. 退出虚拟环境

   `pyenv deactivate`

5. 删除虚拟环境

   `pyenv virtualenv-delete <folder_name>`

   ```sh
   ➜  test pyenv virtualenv-delete hue
   ```

   

