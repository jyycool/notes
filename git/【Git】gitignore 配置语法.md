# 【Git】gitignore 配置语法

## 前言

`.gitignore` 用来忽略git项目中一些文件，使得这些文件不被 git 识别和跟踪；

简单的说就是在 `.gitignore` 添加了某个文件之后，这个文件就不会上传到 github 上被别人看见；



## 二、语法规范

1、空行或是以 # 开头的行即注释行将被忽略。

2、可以在前面添加 正斜杠/ 来避免递归,下面的例子中可以很明白的看出来与下一条的区别。

3、可以在后面添加 正斜杠/ 来忽略文件夹，例如 build/ 即忽略 build 文件夹，/doc/build/ 这样的目录也会忽略。

4、可以使用 ! 来否定忽略，即比如在前面用了*.apk，然后使用!a.apk，则这个a.apk不会被忽略。

5、* 用来匹配零个或多个字符，如*.[oa]忽略所有以".o"或".a"结尾；

6、[] 用来匹配括号内的任一字符，如 [abc]，也可以在括号内加连接符，如 [0-9] 匹配0至9的数；

7、? 用来匹配单个字符。



## 三、示例规则说明

```sh
# 1. 不存在 / 的情况下, 表示递归过滤文件
#		递归过滤所有 readme 文件, a/readme,a/b/readme,都会被过滤
readme

# 2. 以 / 开头, 表示非递归, 仅过滤项目根目录下文件
#		过滤项目根目录下 readme 文件, 但 a/readme,a/b/readme,不会被过滤
/readme

# 3. 以 / 结尾, 表示递归过滤文件夹及其所有内容
#		递归过滤 target 文件夹, a/target,a/b/target 文件夹都会被过滤
target/

# 4. 以 / 开头, 并以 / 结尾, 仅过滤项目根目录下目标文件夹
#		过滤项目根目录下 target 文件夹, 但 a/target,a/b/target,不会被过滤
/target/

# * 表示任意
# 	不以 / 开头, 所以会递归过滤所有 .pdf 文件, a/m.pdf,a/b/n.pdf,都会被过滤
*.pdf

# ** 表示递归子目录
# 	以 / 开头, 非递归, 所以会过滤 target 目录及其任意深度子目录下 json 文件. 
/target/**/*.json
```



## 四、添加.gitignore前push项目

（为避免冲突可以先同步下远程仓库 $ git pull）

1. 在本地项目目录下删除暂存区内容： $ git rm -r --cached .
2. 新建.gitignore文件，并添加过滤规则（用文本编辑器如Notepad++）
3. 再次add文件，添加到暂存区
4. 再次commit提交文件
5. $ git commit -m “add .gitignore”
6. 最后push即可

## 五、push文件后修改过滤规则再次使其生效(同上)

```sh
修改完.gitignore 在本地项目目录下

$ git rm -r --cached .
$ git add .
$ git commit -m".gitignore update"
```



## 注意事项

- 命令和注释别在同一行，如`*.txt #注释txt`这样会导致这一行无法被识别
- git add .之前如果有改动.gitignore一定要 执行 git rm -r --cached .
- 合理使用.gitignore可以避免无用文件的上传，也可以防止重要配置信息的泄露