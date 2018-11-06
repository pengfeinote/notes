# zookeeper配置
## zoo.cfg
1. tickTime=2000 <br>
	the number of milliseconds of each tick
2. initLimit=10 <br>
	the number of ticks that initial synchronization phase can take
3. syncLimit <br>
	the number of ticks that can pass between sending a request and getting an acknowledgement
4. dataDir <br>
   the directory where the snapshot is stored. do not use /tmp for storage, /tmp here is just example
5. dataLogDir <brv>
	dataLogDir没设置的话，就使用dataDir,dataDir存放snapshot数据，dataLogDir存放日志数据
6. clientPort <br>
	供客户端连接的端口号
7. maxClientCnxns <br>
	the max number of client connections. increase this if you need to handle more clients
8. autopurge.snapRetainCount <br>
	autopurge只主要配置zookeeper删除磁盘上保留的snapshot
	the number of snapshots to retain in dataDir
9. autopurge.pergeInterbval=4 <br>
	purge task interval in hour, set to 0 to disable auto purge
10. 集群配置 <br>
	示例： <br>
	server.1 = 127.0.0.1:2888:3888 <br>
	server.2 = 127.0.0.2:2888:3888 <br>
	server.3 = 127.0.0.3:2888:3888 <br>
	第一个端口2888用于leader和follower或observer交换数据,后面一个是选举使用的端口;server后面是myid
11. minSessionTimeout与maxSessionTimeout <br>
	一般，客户端连接zookeeper的时候，都会设置一个session timeout，如果超过这个时间client没有与zookeeper server有联系，则这个session会被设置为过期(如果这个session上有临时节点，则会被全部删除，这就是实现集群感知的基础，后面的文章会介绍这一点)。但是这个时间不是客户端可以无限制设置的，服务器可以设置这两个参数来限制客户端设置的范围。
12. myid
	在dataDir里会放置一个myid文件，里面就一个数字，用来唯一标识这个服务。这个id是很重要的，一定要保证整个集群中唯一。zookeeper会根据这个id来取出server.x上的配置。比如当前id为1，则对应着zoo.cfg里的server.1的配置。