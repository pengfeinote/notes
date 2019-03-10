## xargs

#### 参考文章
http://www.cnblogs.com/wangqiguo/p/6464234.html

#### 作用
xargs可以接受管道的输出，并将管道的输出通过空格分割（默认通过空格分割），作为下一个命令的参数

#### 示例

xargs与linux管道类似，但在使用上大不相同，管道将上一个命令的输出作为下一个命令的标准输入，xargs将上一个命令的输出作为下一个命令的输入参数（以空格做分隔）

例如：  
echo "--help" | cat   
结果是：  
--help  
而echo "--help" | xargs cat  
结果是cat的使用说明，相当于输入命令 cat --help

echo "test.txt test.cpp" | xargs cat  
则相当于打印出了test.txt和test.cpp的内容

#### 常用参数

* -d选项  
-d指定分隔符，例如：  
cat '11@22@33' |xargs -d @ echo  
输出11 22 33
* -p选项  
执行前询问  
执行前会将完整的命令打出来，用户输入y，则会继续执行
* -n选项  
-n 后面跟数字，表示一次传输几个参数给后面的命令.  
例：echo "11 22 33" | xargs -n1 -p echo
* -E选项  
-E指定的参数后面的参数忽略，-E只有在xargs不指定-d的时候有效，如果指定了-d则不起作用，而不管-d指定的是什么字符，空格也不行。
