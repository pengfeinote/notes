## find命令

 Linux find命令用来在指定目录下查找文件。任何位于参数之前的字符串都将被视为欲查找的目录名。如果使用该命令时，不设置任何参数，则find命令将在当前目录下查找子目录与文件。并且将查找到的子目录和文件全部进行显示。

### find参数
find   path   -option   [   -print ]   [ -exec   -ok   command ]   {} 

常用选项：

1. -mount, -xdev : 只检查和指定目录在同一个文件系统下的文件，避免列出其它文件系统中的文件
2. -amin n : 在过去 n 分钟内被读取过
3. -anewer file : 比文件 file 更晚被读取过的文件
4. -atime n : 在过去n天内被读取过的文件
5. -cmin n : 在过去 n 分钟内被修改过
6. -cnewer file :比文件 file 更新的文件
7. -ctime n : 在过去n天内被修改过的文件
8. -empty : 空的文件-gid n or -group name : gid 是 n 或是 group 名称是 name
9. -ipath p, -path p : 路径名称符合 p 的文件，ipath 会忽略大小写
10.  <B>-name name, -iname name : 文件名称符合 name 的文件。iname 会忽略大小写</B>
11. -size n : 文件大小 是 n 单位，b 代表 512 位元组的区块，c 表示字元数，k 表示 kilo bytes，w 是二个位元组。-type c : 文件类型是 c 的文件。

### 实例

1. find . -name "*.c"
2. find . -type f
