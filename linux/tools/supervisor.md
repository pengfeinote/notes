## Supervisor

Supervisor是python开发的一个通用的进程管理程序


### 安装

yum install supervisor

配置文件：/etc/supervisor/supervisord.conf
子进程默认位置：/etc/supervisord.d/

### 使用

#### 启动supervisord

supervisord -c /etc/supervisor/supervisord.conf

#### 添加新进程

在配置目录（默认为/etc/supervisord.d/）下新建一个进程的配置文件

```
#项目名
[program:blog]
#脚本目录
directory=/opt/bin
#脚本执行命令
command=/usr/bin/python /opt/bin/test.py

#supervisor启动的时候是否随着同时启动，默认True
autostart=true
#当程序exit的时候，这个program不会自动重启,默认unexpected，设置子进程挂掉后自动重启的情况，有三个选项，false,unexpected和true。如果为false的时候，无论什么情况下，都不会被重新启动，如果为unexpected，只有当进程的退出码不在下面的exitcodes里面定义的
autorestart=false
#这个选项是子进程启动多少秒之后，此时状态如果是running，则我们认为启动成功了。默认值为1
startsecs=1

#脚本运行的用户身份 
user = test

#日志输出 
stderr_logfile=/tmp/blog_stderr.log 
stdout_logfile=/tmp/blog_stdout.log 
#把stderr重定向到stdout，默认 false
redirect_stderr = true
#stdout日志文件大小，默认 50MB
stdout_logfile_maxbytes = 20M
#stdout日志文件备份数
stdout_logfile_backups = 20

```



#### 进程管理常用命令


* status: 查看所有进程状态
* stop: 终止指定进程
* start: 启动指定进程
* restart：重启指定进程
* update： 配置文件修改后使用该命令加载新配置
* reload：重新启动配置文件中的所有程序



