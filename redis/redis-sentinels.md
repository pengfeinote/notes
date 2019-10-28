https://redis.io/topics/sentinel
http://www.cnblogs.com/zhoujinyi/p/5569462.html

## redis-sentinel

### 简介

sentinel主要做以下几件事：  

* monitoring: 不断监控主服务器和从服务器是否正常运作
* Notification: 当被监控的某个redis服务器出现问题时，Sentinel可以通过API向管理员或者其他应用<B>发送通知</b>
* Automatic failover: 当一个主服务器不能正常工作时，Sentinel会开始一次自动故障迁移操作，它会将失效主服务器的其中一个从服务器升级为新的主服务，并将失效主服务器的其他从服务器改为复制新的主服务器，当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

* Configuration Provider: sentinel向连接它的客户端提供可用的主redis实例的地址，如果发生故障迁移，则sentinel提供新的地址

Redis Sentinel 是一个分布式系统， 你可以在一个架构中运行多个 Sentinel 进程（progress）， 这些进程使用流言协议（gossip protocols)来接收关于主服务器是否下线的信息， 并使用投票协议（agreement protocols）来决定是否执行自动故障迁移， 以及选择哪个从服务器作为新的主服务器。

虽然 Redis Sentinel 释出为一个单独的可执行文件 redis-sentinel ， 但实际上它只是一个运行在特殊模式下的 Redis 服务器， 你可以在启动一个普通 Redis 服务器时通过给定 –sentinel 选项来启动 Redis Sentinel 。

### 使用

使用src/redis-sentinel /path/to/sentinel.conf启动。
sentinel是单独的进程实例，默认端口是26379，或者使用src/redis-server /path/to/sentinel.conf --sentinel
使用规则:

* 为保证鲁棒性，至少需要3个sentinel实例，因为启动故障转移你需要至少多数sentinel的同意(since Sentinels always need to talk with the majority in order to start a failover.)(in order to prevent a permanent split brain condition.)
* 3个实例最好运行在不同的机器上
* Sentinel + Redis distributed system does not guarantee that acknowledged writes are retained during failures, since Redis uses asynchronous replication. However there are ways to deploy Sentinel that make the window to lose writes limited to certain moments, while there are other less secure ways to deploy it.
* 客户端需要支持sentinel，并不是所有客户端都支持sentinel
* 使用docker等容器部署sentnel要需要小心

### 配置

sentinel monitor mymaster 127.0.0.1 6379 2 <br>
sentinel down-after-milliseconds mymaster 60000<br>
sentinel failover-timeout mymaster 180000<br>
sentinel parallel-syncs mymaster 1<br>

sentinel monitor resque 192.168.1.3 6380 4<br>
sentinel down-after-milliseconds resque 10000<br>
sentinel failover-timeout resque 180000<br>
sentinel parallel-syncs resque 5<br>

* 上面的配置监控了两套redis实例，一个master名叫mymaster，一个master名叫resque
* 第一行配置: sentinel monitor <master-group-name> <ip> <port> <quorum>
	* 表示监控的master名称，ip地址和端口号
	* quorum表示标记master不可用或者启动故障修复，至少需要几个sentinel的同意
	* quorum仅代表detect failover的sentinel数量，failover发生需要majority数量的同意。如果集群有5个sentinel，quorum设置的是2，If two Sentinels agree at the same time about the master being unreachable, one of the two will try to start a failover.If there are at least a total of three Sentinels reachable, the failover will be authorized and will actually start.In practical terms this means during failures Sentinel never starts a failover if the majority of Sentinel processes are unable to talk (aka no failover in the minority partition).
* down-after-milliseconds: 多长时间不可达，则sentinel认为该redis失效（down-after-milliseconds is the time in milliseconds an instance should not be reachable (either does not reply to our PINGs or it is replying with an error) for a Sentinel starting to think it is down.)
* parallel-syncs: 在进行故障转移时，最多有多少个slave可以从新的master同步数据，这个数字越小，故障转移的时间越长，但数字越大，就意味着越多的从服务器不可用
* 所有的sentinel参数可以通过SENTINEL SET命令动态修改
* 避免写在失效的master实例上：如果有3个redis实例，3个sentinel分别部署在3个redis实例上，如果master redis与其他两个机器失去连接，sentinel启动故障转移，设置slave为新的master，此时如果客户端继续写在旧master上，写的数据将会丢失，因为如果连接回复，旧的master将会变为slave与新的master同步，丢失自有的数据。此时可以使用下面的配置
	* 	min-slaves-to-write: 3<br>
		min-slaves-max-lag: 10<br>
		如果slave小于3个，或者3个slave的延迟都大于10s，则master redis将拒绝写操作
* slave prioriy<br>
  redis实例有slave-priority选项，
  * 该值为0时，该slave不会被选为master
  * 该值越小，sentinel越可能会把该实例作为master备选
* redis实例与sentinel实例均可以设置密码（使用requirepass配置），如果redis实例设置了密码，则在sentinel里需要通过sentinel auth-pass <master-group-name> <pass>来设置master的密码，同时因为master和slave可能相互转变，需要命令masterauth设置master密码。同理sentinel如果设置密码，所有sentinel应该设置同样的密码

### 原理
* sdown与odown状态:Redis Sentinel has two different concepts of being down, one is called a Subjectively Down condition (SDOWN) and is a down condition that is local to a given Sentinel instance. Another is called Objectively Down condition (ODOWN) and is reached when enough Sentinels (at least the number configured as the quorum parameter of the monitored master) have an SDOWN condition, and get feedback from other Sentinels using the SENTINEL is-master-down-by-addr command.
* sentinels互连：sentinel实例不需要通过sentinel.conf配置进行互联，sentinel通过redis实例的pub/sub功能发现其他的sentinels(Sentinel uses the Redis instances Pub/Sub capabilities in order to discover the other Sentinels that are monitoring the same masters and slaves.)
	* 每个连接到redis（master/slave）实例的sentinel向channel \_\_sentinel\_\_:hello发送一条消息
	* 每个sentinel订阅该实例，有新的未知sentinel加入时，加入到master的sentinel集群中
	* hello message也会包含最新的配置信息
	* 添加新的sentinel时，会检查同一机器/端口是否已经有sentinel实例了，如果有，会把旧的删除，加入新的sentinel实例
* sentinel总会将最新配置加到监控的实例中 (Sentinel is a system where each process will always try to impose the last logical configuration to the set of monitored instances.)
* slave选举优先级
	* Disconnection time from the master.
	* slave priority
	* Replication offset processed.
	* Run ID.
	

### ha(高可用)搭建
