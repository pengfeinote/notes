## linux文件对比工具

### diff
逐行比较文件，输出文件之间的差异
对比文件夹
diff -urNa dir1 dir2
　　-a Treat all files as text and compare them line-by-line, even if they do not seem to be text.
　　-N, --new-file
　　　　In directory comparison, if a file is found in only one directory, treat it as present but empty in the other directory.
　　-r When comparing directories, recursively compare any subdirectories found.

　　-u Use the unified output format.

### colordiff
diff的扩展版本

### vimdiff
同时打开两个文件，对比文件差异

### comm
一行一行对两个已经排序的文件进行比较，在第三列中显示同一行是否相同，需要两个文件已经排序，
