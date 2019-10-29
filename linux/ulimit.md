### 系统限制ulimit

#### ulimit限制

core file size          (blocks, -c) 0
data seg size(最大内存大小)           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 31362
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files(最大文件句柄数)                      (-n) 65535
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size(栈大小)              (kbytes, -s) 10240
cpu time               (seconds, -t) unlimited
max user processes(最大进程数)              (-u) 65535
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited

#### 系统最大线程数

cat /proc/sys/kernel/threads-max


#### 系统最大进程/线程

cat /proc/sys/kernel/pid_max
