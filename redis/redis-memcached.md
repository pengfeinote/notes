1、数据类型
Redis支持丰富的数据类型，比如string、hash、list、set、zset。

Memcached只支持简单的key/value数据结构。

2、单线程/多线程/承载QPS
Redis是单线程请求，所有命令串行执行，并发情况下不需要考虑数据一致性问题；性能受限于CPU，单实例QPS在4-6w。

Memcached是多线程，可以利用多核优势，单实例在正常情况下，可以达到写入60-80w qps，读80-100w qps。

3、持久化/数据迁移
Redis支持持久化操作，可以进行aof及rdb数据持久化到磁盘。

Memcached无法进行持久化，数据不能备份，只能用于缓存使用，且重启后数据全部丢失。

4、对热点、bigkey支持的友好度
Redis的big key与热key类操作，如果qps较高则容易造成redis阻塞，影响整体请求。

Memcached因为是多线程，与redis相比，在big key与热key类操作上支持较好。

5、高可用/HA
Redis支持通过Replication进行数据复制，通过master-slave机制，可以实时进行数据的同步复制，

支持多级复制和增量复制，master-slave机制是Redis进行HA的重要手段。

Memcached无法进行数据同步，不能将实例中的数据迁移到其他MC实例中。

6、发布订阅机制
Redis支持pub/sub消息订阅机制，可以用来进行消息订阅与通知。

Memcached不支持发布订阅。

7、周边支持
Redis相对memcached，周边工具支持较好，比如迁移、数据分析等方便，目前KCC支持全量和指定前缀等数据分析和删除功能。

Memcched周边支持较少，且原生不支持key分析等操作，目前KCC自研实现针对中小memached集群的key分析和指定前缀数据删除功能。
