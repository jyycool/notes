# 【Homebrew】MacOS软件管理工具-Homebrew

## 介绍

经常使用Linux系统的人们，应该非常熟悉使用 ***yum*** 或 ***apt-get*** 来管理软件包。苹果MacOS 既拥有 Windows 一样易用的图形操作界面，也拥有 Linux 强大的命令行操作。

***Homebrew*** 是一款 MacOS 平台下的软件包管理工具，拥有安装、卸载、更新、查看、搜索等很多实用的功能。简单的一条指令，就可以实现包管理，而不用你关心各种依赖和文件路径的情况，十分方便快捷。

## Homebrew仓库源替换 

如果本地已经安装 homebrew, 则执行以下操作替换源地址

```sh
# 替换brew.git:
cd "$(brew --repo)"
git remote set-url origin https://mirrors.ustc.edu.cn/brew.git
# 核心软件仓库替换为 中科大 国内镜像
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git
# cask软件仓库替换为 中科大 国内镜像
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-cask"
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git
# Bottle源替换为 中科大 国内镜像
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.bash_profile
```

```sh
# 若上面的源不可用, 可以再次尝试调换为清华的源
➜  ~ git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git
➜  ~ git -C "$(brew --repo homebrew/cask)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask.git
```



## Homebrew基本用法

假设需要安装的软件是 wget

| 操作                   | 命令                     |
| ---------------------- | ------------------------ |
| 更新 Homebrew          | brew update              |
| 更新所有安装过的软件包 | brew upgrade             |
| 更新指定的软件包       | brew upgrade [soft_name] |
| 查找软件包             | brew search [soft_name]  |
| 安装软件包             | brew install [soft_name] |
| 卸载软件包             | brew remove [soft_name]  |
| 列出已经安装的软件包   | brew list                |
| 查看软件包信息         | brew info [soft_name]    |
| 列出软件包的依赖关系   | brew deps [soft_name]    |
| 列出可以更新的软件包   | brew outdated            |

### 卸载 Homebrew

```sh
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"
```



### 参考：

[Homebrew 中文主页](https://brew.sh/index_zh-cn.html)

[Homebrew Bottles 源使用帮助](http://mirrors.ustc.edu.cn/help/homebrew-bottles.html)

[Homebrew Cask 源使用帮助](http://mirrors.ustc.edu.cn/help/homebrew-cask.git.html)

[Homebrew Core 源使用帮助](http://mirrors.ustc.edu.cn/hel)



