# 【Linux】变量的作用域

环境变量和自定义变量的概念不多说.



脚本 var.sh

```shell
#! /bin/sh
name=sherlock
age=10
```

脚本 pub.sh

```sh
#! /bin/sh
echo $name
echo $age
```

1. 用subshell的方式 执行 var.sh 和 pub.sh

   ```sh
   ➜  shell ./var.sh      
   ➜  shell ./pub.sh
   
   
   ➜  shell 
   ```

   并未输出变量 name 和 age

2. 使用当前shell 执行 var.sh 和 pub.sh

   ```sh
   ➜  shell source var.sh  
   ➜  shell source pub.sh 
   sherlock
   10
   ➜  shell echo $name
   sherlock
   ➜  shell echo $age 
   10
   ➜  shell ./pub.sh
   
   
   ➜  shell 
   ```

   可见用 ***source var.sh 和 . var.sh*** 的方式执行脚本会将 变量作用域作用到当前 shell 环境,  但是在subshell 中变量还是无法获取.

3. 使用 ***expoet name 和 export age***, 可以将自定义变量 改为 环境变量,  这样不管是当前 shell 还是 subshell 都可以获取变量值 .

