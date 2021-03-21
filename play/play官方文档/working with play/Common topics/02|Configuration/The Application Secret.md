# The Application Secret

play 的秘钥应用于很多事:

- session cookies 的签名和 CSRF 令牌
- 内置加密工具

在 `application.conf` 中使用属性名称 `play.http.secret.key` 来配置，默认值为 `changeme`。如默认设置所示，应将其更改用于生产。

> **注意：**在产品模式下启动时，如果 Play 发现未设置 secret，或者如果设置为`changeme`，则Play会引发错误。



## 最佳实践

强烈建议您不要将应用程序密码配置在源代码管理中。而是应在生产服务器上对其进行配置。这意味着将生产应用程序秘钥放在 `application.conf` 中被认为是危险的做法。

一种方法是在生产服务器上配置应用程序密码, 将其作为系统属性传递给启动脚本。例如：

```sh
/path/to/yourapp/bin/yourapp -Dplay.http.secret.key='QCY?tAnfk?aZ?iwrNwnxIlR6CTf:G3gf:90Latabg@5241AB`R5W:1uDFN];Ik@n'
```

这种方法非常简单，我们会在Play文档中使用此方法在生产模式下运行您的应用程序，以提醒您需要设置应用程序密码。但是，在某些环境中，在命令行参数中放置机密不是一种好的做法。有两种解决方法。

### 环境变量

第一个是将应用程序秘密放在环境变量中。在这种情况下，建议您在`application.conf`文件中放置以下配置：

```properties
play.http.secret.key="changeme"
play.http.secret.key=${?APPLICATION_SECRET}
```

该配置的第二行将机密设置为来自环境变量（`APPLICATION_SECRET` 如果设置了该环境变量），否则，该机密与上一行保持不变。

这种方法在基于云的部署方案中特别有效，其中通常的做法可以通过该云提供商的 API 配置的环境变量来设置密码和其他机密。

### 生产配置文件

另一种方法是创建一个`production.conf` 驻留在服务器上的文件，该文件包括 `application.conf`，但也覆盖任何敏感配置，例如应用程序密钥和密码。

例如:

```properties
include "application"

play.http.secret.key="QCY?tAnfk?aZ?iwrNwnxIlR6CTf:G3gf:90Latabg@5241AB`R5W:1uDFN];Ik@n"
```

然后，您可以启动 Play 程序：

```sh
/path/to/yourapp/bin/yourapp -Dconfig.file=/path/to/production.conf
```



## 配置应用 secret 的需求

检查应用程序秘钥配置 `play.http.secret.key` 的最小生产长度。如果密钥不超过15个字符，将记录警告。如果密钥是八个字符或更少，则将引发错误并且配置无效。您可以通过将机密设置为至少32个字节的完全随机输入来解决此错误，例如使用`head -c 32 /dev/urandom | base64`或通过应用程序机密生成器，使用 `playGenerateSecret` 或 `playUpdateSecret` 如下所述。

而且，如上所述，应用程序秘密用作确保Play会话cookie有效的密钥，即已由服务器生成，而不是由攻击者欺骗。但是，密钥仅指定一个字符串，而不能确定该字符串中的熵量。无论如何，仅通过测量密钥的简短程度，便可以对密钥中的熵量设置上限：如果密钥长度为8个字符，则最多为64位熵，这在现代标准中是不够的。

## 生成应用程序密码

Play提供了可用于生成新机密的实用程序。在Play控制台中运行 `playGenerateSecret `。这将生成一个新的 `secret`，您可以在应用程序中使用它。例如：

```sh
[my-first-app] $ playGenerateSecret
[info] Generated new secret: QCYtAnfkaZiwrNwnxIlR6CTfG3gf90Latabg5241ABR5W1uDFNIkn
[success] Total time: 0 s, completed 28/03/2014 2:26:09 PM
```



## 在application.conf 中更新应用程序密钥

如果您要为开发或测试服务器配置特定的秘钥，Play 还提供了一个方便的实用程序来更新 `application.conf` 中的 `secret`。当您使用应用程序密钥对数据加密时，并且要确保每次在dev模式下运行应用程序时，都使用相同的密钥，这通常很有用。

要更新中的秘密 `application.conf`，请在Play控制台中运行 `playUpdateSecret` ：

```sh
[my-first-app] $ playUpdateSecret
[info] Generated new secret: B4FvQWnTp718vr6AHyvdGlrHBGNcvuM4y3jUeRCgXxIwBZIbt
[info] Updating application secret in /Users/jroper/tmp/my-first-app/conf/application.conf
[info] Replacing old application secret: play.http.secret.key="changeme"
[success] Total time: 0 s, completed 28/03/2014 2:36:54 PM
```

