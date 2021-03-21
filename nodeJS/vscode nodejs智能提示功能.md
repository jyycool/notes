## vscode nodejs智能提示功能

最近想学学nodejs挑战一下全栈开发

在学习过程发现vscode没有对node智能代码提示，so在写代码中我也是懵得一逼



了解了一下 需要依赖一些第三方的插件

1. 配置npm镜像源 ( 这个如果之前已经安装过, 就不需要再安装了 )

   - 临时使用镜像源, 安装包的时候可以使用`--registry`这个参数

     ```shell
     // 临时配置淘宝的镜像源
     $ npm install express --registry https://registry.npm.taobao.org
     ```

   - 全局使用镜像源

     ```shell
     $ npm config set registry https://registry.npm.taobao.org
     // 配置完成后可以使用下面的方式来验证是否成功
     $ npm config get registry
     // 或
     $ npm info express
     ```

   - cnpm的使用 ( 不推荐 )

     ```sh
     // 安装cnpm
     $ npm install -g cnpm --registry=https://registry.npm.taobao.org
     
     // 使用 cnpm安装包
     $ cnpm install express
     ```

2. 安装typings这个包

   ```sh
   $ npm install -g typings
   
   // 验证typings是否安装成功
   $ typings --version
   ```

3. 项目根目录安装初始化typings

   ```sh
   $ typings init
   ```

   

