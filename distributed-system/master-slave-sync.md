## 主从同步及可靠性保证

### mysql

master中有一个线程不停读取binlog中的日志，发送给slave，slave写入redolog，同时slave不听读取relay log的信息，将数据更新到本地实例

mysql有异步复制和半同步复制方式。异步复制，是指mysql将记录写入binlog即返回客户端，这样做效率最高，但是可能导致数据丢失，如果数据写入binlog后主库挂掉，此时slave还没来得及同步会导致数据丢失。

mysql还有半同步方式，如果master将日志写到binlog后，等到将binlog中的记录发送给slave relay-log之后才会返回ok

在进行主从复制时，如果主库已经有数据，则需要将主库锁定，然后导出数据快照，将数据快照写入从库中，然后再开启主从复制，保证主从一致

### redis

redis首先是全量复制，然后是增量复制；当有新的slave加入时，master首先bgsave将数据存为rdb文件，与此同时，记录这个过程中执行的写命令，当bgsave结束时，会将全量数据发送给slave，然后持续降后续写命令发送给slave，slave更新自身数据。

redis主从没有为数据提供强一致性，如果多个slave同时启动，将导致master发送多份儿全量数据，会使master的IO激增

### kafka

所有的副本集合称为ar(all replicas),kafka有in-sync replicas和out-of-sync replicas之分,kafka会定义一个阈值，延迟高于这个阈值的会从在同步集合转到不再同步集合。

在生产消息时，producer可以根据配置决定三种发送模式，发送不需要确认/发送需要leader确认/发送需要isr确认

kafka定义了“水位”的概念，“水位”可以看作是所有isr中消息的最小值，consumer只能消费水位以下的消息


### rabbitmq

rabbitmq的主从复制时队列复制(rabbitmq称之为mirror-queue)，如果开启confirm模式后，producer每次发送一个消息，都会异步等待rabbitmq的ack，收到ack后，producer才会认为发送成功，在主从模式下，rabbitmq master收到消息后，会将消息异步复制到mirror queue，然后才返回ack

### zookeeper 

leader在收到写请求时，会生成一个zxid，然后提交给所有follower，等到一半以上的follower发回响应时，leader会提交事务，并通知其他follower commit，当leader宕机时，系统会选择数据版本不低于leader版本的节点作为新的leader
