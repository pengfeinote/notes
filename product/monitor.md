## 监控类目

从服务器的角度，将监控分为preview, 灰度, online。上线时先preview，再灰度，再online，让上线人员有足够的时间和机会发现问题。如果是容器部署，则需要分为容器和实体机监控

### 硬件级监控

* cpu负载监控: cpu.busy
* 内存占用百分比监控: mem.memused.percent
* 系统负载（1min, 5min, 15min）监控: load.1min, load.5min, load.15min
* 系统可用内存：mem.memfree
* 内存使用量：mem.memused.nocache
* 网卡入流量：net.if.in.busy/iface=bond0
* 网卡出流量：net.if.out.busy/iface=bond0
* 磁盘使用率：disk.io.util/device=sdb
* 磁盘读写速度：disk.io.write_bytes/device=/dev/sdb, disk.io.read_bytes/device=/dev/sdb
* 


### 进程级监控

#### gc监控

* jvm.mem.used.size: 服务使用内存绝对值，对上边的值在集群中取p995可得内存占用p995
* 按内存分区进行内存占用P995的统计（新声代/老年代/eden/元数据等）
* gc吞吐率(jvm.gc.throughput)：工作时间/(gc时间+工作时间,以分钟为单位)，正常应该大于99%
* gc耗时的绝对时间统计(jvm.gc.time，分区统计),正常应在100ms以内,
* 每分钟gc次数统计(jvm.gc.count，分区统计)，正常每分钟在10以内
* 每分钟old区内存增长(jvm.mem.promotion)，正常应该为0附近，否则可能有内存泄漏
* 每分钟eden区增长统计(振幅过大可能有不合理的内存申请)
* old区内存使用率统计(jvm.mem.usage)，占用率过高可能需要调整jvm参数，或内存泄漏
* 打开的fd统计(jvm.system.openFileDescriptorCount)，正常数值可以根据进程本身判断，一般情况下不会过多，比如20k以上
* jvm cpu使用率(jvm.system.processCpuPercent)
* jvm io任务排队数量(jvm.netty.pendingTasks)：正常不应该超过100
* jvm堆外内存数量(jvm.directMemory.reservedMemory): 


#### 本地缓存监控


#### 异常监控



### 中间件监控

#### rpc监控

#### redis监控

* 内存占用百分比监控
* 访问qps监控
* 访问时间统计监控
* large key监控
* 热点监控？
* 访问成功数监控
* 访问失败数监控


#### mysql监控


#### kafka监控


### 业务级监控

#### http服务通用监控

* 请求QPS/QPM监控，频率不高使用QPM
* 返回结果（业务错误码）监控
* 返回结果（http错误码）监控



### 报警与误报


### api自动化巡检监控
