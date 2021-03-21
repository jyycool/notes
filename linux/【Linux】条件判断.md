# 【Linux】条件判断

## 一、文件表达式

1. `-e filename` 

   如果 filename存在，则为真

2. `-d filename` 

   如果 filename为目录，则为真 

3. `-f filename` 

   如果 filename为常规文件，则为真

4. `-L filename` 

   如果 filename为符号链接，则为真

5. `-r filename` 

   如果 filename可读，则为真 

6. `-w filename` 

   如果 filename可写，则为真 

7. `-x filename` 

   如果 filename可执行，则为真

8. `-s filename` 

   如果文件长度不为0，则为真

9. `-h filename` 

   如果文件是软链接，则为真

10. `filename1 -nt filename2` 

    如果 filename1比 filename2新，则为真。

11. `filename1 -ot filename2` 

    如果 filename1比 filename2旧，则为真。



## 二、整数变量表达式

1. `-eq` 等于
2. `-ne` 不等于
3. `-gt` 大于
4. `-ge` 大于等于
5. `-lt` 小于
6. `-le` 小于等于



## 三、字符串变量表达式

1. `if  [ $string1 = $string2 ]`         

   如果 string1 等于 string2，则为真;  字符串允许使用赋值号做等号

2. `if  [ $string1 != $string2 ]` 

    如果 string1 不等于 string2，则为真    

3. `if  [ -n $string ]`       

   如果string 非空(非0），返回0(true)  

4. `if  [ -z $string ]`       

   如果string 为空，则为真

5. `if  [ $sting ]`          

   如果string 非空，返回0 (和-n类似) 

6. `if [ ! -d $dir ]`       

    如果不存在目录 ${dir}

7. `if [ 表达式1  –a  表达式2 ]`

   逻辑与 –a; 条件表达式的并列

8. `if [ 表达式1  –o 表达式2 ]`  

   逻辑或 -o; 条件表达式的或

