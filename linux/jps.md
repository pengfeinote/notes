## jps命令

### 简介

jps全称Java Virtual Machine Process Status Tool，主要用于查看目标主机上jvm进程的运行状况。

jps命令格式

```
jps [options] [hostid]
```

jps会列出目标机器上有权限的jvm进程列表，如果没有指定host，将会列出本机的进程列表，进程列表中，用户可以看到jvm进程的进程id、进程main函数、启动参数以及jvm参数等等

### 参数

* -q: 只打印进程id
* -m: 打印出main函数及传入main函数的参数
* -l: 打印出main方法的完整包路径
* -v: 打印jvm参数
* -V: 打印通过flag文件传输到jvm的jvm参数



### 命令实例

输入示例：

```
jps -lvm
```


输出示例:

```
1135 com.xk72.charles.macos.gui.Main -Djava.library.path=/Applications/Charles.app/Contents/MacOS -DLibraryDirectory=/Users/wenpengfei/Library -DDocumentsDirectory=/Users/wenpengfei/Documents -DApplicationSupportDirectory=/Users/wenpengfei/Library/Application Support -DCachesDirectory=/Users/wenpengfei/Library/Caches -DApplicationDirectory=/Users/wenpengfei/Applications -DAutosavedInformationDirectory=/Users/wenpengfei/Library/Autosave Information -DDesktopDirectory=/Users/wenpengfei/Desktop -DDownloadsDirectory=/Users/wenpengfei/Downloads -DMoviesDirectory=/Users/wenpengfei/Movies -DMusicDirectory=/Users/wenpengfei/Music -DPicturesDirectory=/Users/wenpengfei/Pictures -DSharedPublicDirectory=/Users/wenpengfei/Public -DSystemLibraryDirectory=/Library -DSystemApplicationSupportDirectory=/Library/Application Support -DSystemCachesDirectory=/Library/Caches -DSystemApplicationDirectory=/Applications -DSystemUserDirectory=/Users -DUserHome=/Users/wenpengfei -DSandboxEnabled=true -DDarkMode=false -DLaunchModifierFlags=0 -DLaunchModifierFlagCapsLock=fal
```
