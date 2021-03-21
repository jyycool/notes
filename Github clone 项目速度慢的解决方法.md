## Github clone 项目速度慢的解决方法

从github clone一些项目的时候速度极慢，有时候clone到一半还会失败，简直令人抓狂。

### 解决步骤

#### 1. 使用国内镜像网站

目前已知Github国内镜像网站有[github.com.cnpmjs.org](https://github.com.cnpmjs.org/)和[git.sdut.me/](https://git.sdut.me/)（亲测这个访问速度较快）。你可以根据你对这两个网站的访问速度快慢，选择其中一个即可。接下来只需要在clone某个项目的时候将github.com替换为github.com.cnpmjs.org即可。如下例：

```bash
git clone https://github.com/Hackergeek/architecture-samples
```

替换为

```bash
git clone https://github.com.cnpmjs.org/Hackergeek/architecture-samples
```

或者

```bash
git clone https://git.sdut.me/Hackergeek/architecture-samples
```

如果你仅仅只是想clone项目，不需要对clone下来的项目进行modify和push，那下面那个步骤就不需要看了。下面的步骤是为了解决使用镜像网站clone下来的项目无法进行push的问题的，因为国内的镜像网站是无法登录的

#### 2. 修改仓库push url

为了解决使用国内镜像网站clone下来的项目无法push的问题，我们需要将clone下来的仓库push url修改为github.com。如下例：
使用国内镜像网站clone下来的项目远程仓库地址如下图：

![exp1](/Users/sherlock/Desktop/notes/allPics/Git/exp1.png)

使用如下命令修改仓库的push url：

```bash
git remote set-url --push origin  https://github.com/Hackergeek/architectu
```

过程如下图：

![exp2](/Users/sherlock/Desktop/notes/allPics/Git/exp2.png)