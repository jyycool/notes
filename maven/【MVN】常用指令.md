## 【MVN】常用指令

1. ***mvn clean package***

   这是常见的项目打包指令, 后面可以跟 ***-DshipTests*** 或 ***-Dmaven.test.skip=true***

   ***mvn clean package -DskipTests*** : 不执行测试用例，但编译测试用例类生成相应的class文件至target/test-classes下。

   ***mvn clean package -Dmaven.test.skip=true*** : 不执行测试用例，也不编译测试用例类。

2. 