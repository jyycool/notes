# 【Linux】/dev/random 和 /dev/urandom

### **1. 基本介绍**　　

　　/dev/random和/dev/urandom是Linux系统中提供的随机伪设备，这两个设备的任务，是提供永不为空的随机字节数据流。很多解密程序与安全应用程序（如SSH Keys,SSL Keys等）需要它们提供的随机数据流。

　　这两个设备的差异在于：/dev/random的random pool依赖于系统中断，因此在系统的中断数不足时，/dev/random设备会一直封锁，尝试读取的进程就会进入等待状态，直到系统的中断数充分够用, /dev/random设备可以保证数据的随机性。/dev/urandom不依赖系统的中断，也就不会造成进程忙等待，但是数据的随机性也不高。

　　使用cat 命令可以读取/dev/random 和/dev/urandom的数据流（二进制数据流,很难阅读），可以用od命令转换为十六进制后查看：

　　![img](https://images0.cnblogs.com/blog/679164/201410/222116048088229.png)

　　![img](https://images0.cnblogs.com/blog/679164/201410/222117125903821.png)

在cat的过程中发现，/dev/random产生的速度比较慢，有时候还会出现较大的停顿，而/dev/urandom的产生速度很快，基本没有任何停顿。

而使用dd命令从这些设备中copy数据流，发现速度差异很大：

从/dev/random中读取1KB的字节流：

　　![img](https://images0.cnblogs.com/blog/679164/201410/222118186686902.png)

从/dev/urandom 中读取1KB的字节流：

　　![img](https://images0.cnblogs.com/blog/679164/201410/222118465279123.png)

通过程序测试也发现：/dev/random设备被读取的越多，它的响应越慢.

使用PHP的加密扩展mcrypt时，mcrypt_create_iv()函数用于从随机源创建初始向量（initialization vector），该函数的签名为：

```
string mcrypt_create_iv ( int $size [, int $source = MCRYPT_DEV_URANDOM ] )
```

注意函数的第二个参数$source,在PHP 5.6.0以下的版本中，该参数默认是 MCRYPT_DEV_RANDOM，也就是说，mcrypt_create_iv默认从/dev/random设备获取随机数据源的。这在系统并发数较高时，系统无法提供足够的中断数，会导致访问进程挂起（锁住），从而无法正常响应。

一个简单的测试脚本如下：

```
1 <?php
2 define("MCRYPT_KEY","x90!-=zo2s");
3 $src = "test";
4 
5 $size = mcrypt_get_iv_size(MCRYPT_BLOWFISH,MCRYPT_MODE_ECB);
6 $iv = mcrypt_create_iv($size);
7 $encrypted = mcrypt_ecb(MCRYPT_BLOWFISH, MCRYPT_KEY, $src, MCRYPT_DECRYPT, $iv);//5.5+已废弃，请使用最新的API测试
```

我们之前在cat /dev/random的输出时已经发现，输出的随机数据流会出现较大的停顿。在并发数较大时，会造成读取进程的等待甚至无法响应。

幸好，我们可以指定第二个参数为**MCRYPT_DEV_URANDOM**使其强制使用/dev/urandom设备的随机数据流（PHP 5.6.0+版本中，已经默认使用/dev/urandom作为随机数据源）。

### **2.  /dev/random和/dev/random的其他用途**

  1. 这两个伪设备可用于代替mktemp产生随机临时文件名：

     ```sh
     cat /dev/urandom |od –x | tr –d  ' '| head –n 1
     ```

　　可以产生128位(bit)的临时文件名，具有较高的随机性和安全性。

2. 可以模拟生成SSH-keygen生成的footprint,脚本如下：

   ```sh
   #/bin/sh -
   cat /dev/urandom |
   od -x |
   head -n 1|
   cut -d ' ' -f 2- |
   awk -v ORS=":" 
   '{
       for(i=1; i<=NF; i++){
           if(i == NF){
               ORS = "\n";
           }
           print substr($i,1,2) ":" substr($i,3,2);
       }
   }'
   ```

   对该脚本的简单解释：

　　(1). cat /dev/urandom | od -x | head -n 1 用于从随机设备中读取一行数据流并转换为16进制。该段的输出类似于：　　　![img](https://images0.cnblogs.com/blog/679164/201410/222212381992464.png)

　　(2). 由于第一列实际上是数据的偏移量，并不是随机数据流，再次用cut取出后面的几个字段：cut -d ' ' -f 2-

　　(3). 利用awk程序输出。ORS是awk的内置变量，指输出记录分割符，默认为\n。

　　脚本的输出结果：

　　![img](data:image/png;base64,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)

　　对比用ssh-keygen生成的footprint，是不是挺像的？ :D

![img](data:image/png;base64,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)

