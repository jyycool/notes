# JetBrains Runtime（jbr）的介绍和更改

## 1. JetBrains Runtime

JetBrains Runtime （即 JetBrains 运行时）是一个运行时环境，用于在 Windows，Mac OS X 和 Linux 上 **运行** IntelliJ 平台的各种产品。JetBrains Runtime 基于 OpenJDK 项目，并进行了一些修改。这些修改包括：抗锯齿，Linux 上增强的字体渲染，HiDPI 支持，连字，一些官方版本中未提供的针对产品崩溃的修复程序以及其他小的增强功能。

JetBrains Runtime 不是 OpenJDK 的认证版本。请自己承担风险使用。

可以通过 IDE 的 Help About 在弹出的对话框中的 "Runtime Version" 来验证当前的 JetBrains Runtime 版本。



![img](https://pic1.zhimg.com/80/v2-46bb81e1d30fe82f6d72e293bedb6ec0_1440w.jpg)



所有 IntelliJ 产品都已绑定 JetBrains Runtime，默认情况下将使用此运行时。如果需要将运行时更改为其他版本，请参见下文。

默认绑定的 JetBrains Runtime 路径在 IDE 安装路径下的：

- 2020.1 及以上版本为：`jbr` 目录
- 2019.3.x 及以下版本为：`jre32/64` 目录

## 2. 更改 JetBrains Runtime

> 注意：一般不建议更改

### 2.1 通过 Choose Runtime 插件进行切换

- 先安装此插件
- 打开此插件：通过菜单 Help Find Action 输入 `Choose Runtime` 并回车
- 选择要安装的版本，`sdk` 后面的数字越大，则版本越新。 安装最新版本或 JetBrains 工作人员要求您尝试的版本很有意义。 安装旧的运行时版本可能会使您的 IDE 无法使用或引入新的问题；比如在 Linux中 的字体显示、输入法不能跟随光标等。

![img](https://pic1.zhimg.com/80/v2-925dd1b6b82e4dd14fb9bac91c93af84_1440w.jpg)



- 单击 "下载" 按钮，下载 JetBrains Runtime 文件。
- 下载完成后，单击 "安装" 按钮，文件将解压到 `idea.config.path\jdks` 位置，并会修改 `idea.config.path\<product>.jdk` 文件内容为此 JDK 的完整路径。然后 IDE 将自动重新启动。（这里的 `idea.config.path` 路径 和 `<product>.jdk`文件将在下文讲解）
- Help About 在弹出的对话框中验证当前的 JetBrains Runtime 版本
- 如果对话框中的版本并没有改变，那是因为你配置了配置了某个环境变量，比如 `IDEA_JDK_64` ，它的优先级高于 `<product>.jdk`文件 中配置的 JetBrains Runtime

> 这里 `<product>.exe` 中的 `product` 表示 IntelliJ 平台中的某个产品。

### 2.2 通过环境变量或配置文件修改

> **不同系统配置不同**

对于 Windows 系统：

`<product>.exe` 对 JDK 的查找顺序为：

1. `IDEA_JDK / PHPSTORM_JDK / WEBIDE_JDK / PYCHARM_JDK / RUBYMINE_JDK / CL_JDK / DATAGRIP_JDK / GOLAND_JDK` 环境变量。如果是 `<product>64.exe` 则在环境变量末尾添加 `_64` ，比如 `IDEA_JDK_64` 。
2. `idea.config.path\<product>.jdk` 配置文件 。或 `<product>64.jdk`
3. `..\jre` 目录 ；或 `..\jre64` 目录
4. `JDK_HOME` 环境变量
5. `JAVA_HOME` 环境变量

注意：这里的环境变量必须指向 JDK 安装主目录，例如 `c:\Program Files (x86)\Java\jdk1.8.0_112`



**Linux：**

- 从 IntelliJ IDEA 2016 到 最新版本的轻量级 IDE，我们将自定义 JRE 与 Linux 发行版捆绑在一起，就像我们为 Mac 所做的一样。 我们的自定义 JRE 基于 OpenJDK，并包括最新的修复程序，以在 Linux 上提供更好的用户体验（例如字体渲染改进和 HiDPI 支持）。
- 引导 JDK 路径存储在 config 文件夹 中的`<product>.jdk`文件中。它可以通过 `Change IDE boot JDK` 操作修改（Help Find Action 输入 `Change IDE boot JDK`），也可以通过手动编辑此 `.jdk` 文件（如果您不能通过`Change IDE boot JDK`来更改时）
- 建议使用捆绑的 JRE（如果有）。 如果捆绑版本有任何问题，则可以切换到系统上可用的最新 Oracle JDK 或 OpenJDK 构建（建议使用 JDK 11+）。
- 如果更改后你的字体显示比较丑，则可参考 [this thread comments](https://link.zhihu.com/?target=http%3A//youtrack.jetbrains.com/issue/IDEA-57233)
- Help About 在弹出的对话框中验证当前的 JetBrains Runtime 版本



**Mac OS：**

- 默认情况下，IDE 使用捆绑 JetBrains 运行时。
- 如果重写 IDE JDK 版本（通过 Choose Runtime 插件），则其路径保存在 config 文件夹中的`<product>.jdk`文件中。 如果 IDE 不再启动并且您无法通过菜单更改它，请删除此文件或手动更改文件内部的路径。
- 如果 IDE 无法启动且该文件不存在，请手动创建它并指定要使用的 Java 路径（Java home 路径）

## 3. config 配置文件夹

即前面说的 `idea.config.path` ，该路径是 IDE 用来存储设置、缓存、插件和日志的目录。具体位置取决于操作系统、产品和版本

**2020.1 及以上版本：**

**Windows ：**

Syntax

```
%USERPROFILE%\AppData\Roaming\JetBrains\<product><version>
```

Example

```
C:\Users\JohnS\AppData\Roaming\JetBrains\IntelliJIdea2020.1
```



**MacOS ：**

Syntax

```
~/Library/Application Support/JetBrains/<product><version>
```

Example

```
~/Library/Application Support/JetBrains/IntelliJIdea2020.1
```



**Linux：**

Syntax

```
~/.config/JetBrains/<product><version>
```

Example

```
~/.config/JetBrains/IntelliJIdea2020.1
```



**2019.3.x 及以下版本：**

**Windows ：**

Syntax

```text
 %HOMEPATH%\.<product><version>\config
```

Example

```text
 C:\Users\JohnS\.IntelliJIdea2019.3\config
```



**MacOS ：**

Syntax

```
~/Library/Preferences/<product><version>
```

Example

```
~/Library/Preferences/IntelliJIdea2019.3
```



**Linux：**

Syntax

```
~/.<product><version>/config
```

Example

```
~/.IntelliJIdea2019.3/config
```



## 4. 参考

[https://confluence.jetbrains.com/display/JBR/JetBrains+Runtime](https://link.zhihu.com/?target=https%3A//confluence.jetbrains.com/display/JBR/JetBrains%2BRuntime)

[https://intellij-support.jetbrains.com/hc/en-us/articles/206544519](https://link.zhihu.com/?target=https%3A//intellij-support.jetbrains.com/hc/en-us/articles/206544519)

[https://intellij-support.jetbrains.com/hc/en-us/articles/206544879-Selecting-the-JDK-version-the-IDE-will-run-under](https://link.zhihu.com/?target=https%3A//intellij-support.jetbrains.com/hc/en-us/articles/206544879-Selecting-the-JDK-version-the-IDE-will-run-under)