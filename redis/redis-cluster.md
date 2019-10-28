### redis cluster

cluster与sentinel：redis 主从/Sentinel是为了解决redis HA(high available)的问题，redis cluster是解决单机内存不够用的问题

#### cluster槽分配

redis集群定义了16384个槽，redis集群会对key计算CRC16的值，余上16384得到hash值，然后均匀的分布到各个node上，新增或删除node时，会对数据进行迁移

**为什么是16384个槽呢**

首先，2的n次方有利于取余；其次，cluster nodes之间的心跳包需要包含所有的槽信息，以便于幂等的更新各个节点的配置，16k的槽配置大小约为2k，65k约为8k，一个redis集群的主node数一般不超过1000个，所以折中选取了16k

#### cluster中的主从

cluster模式相比于单机redis，提高了内存容量，如果需要提高高可用，则需要添加主从模式

主从模式并不能保证强一致性，因为redis采用了异步数据同步，在切换主从时可能会导致数据丢失；不过redis提供了WAIT命令来保证同步写，不过仍然无法保证强一致性。WAIT命令通知redis只有指定数量的节点接受写命令了，才会确认些成功。

通过配置node Timeout设置节点超时时间

After node timeout has elapsed, a master node is considered to be failing, and can be replaced by one of its replicas. Similarly after node timeout has elapsed without a master node to be able to sense the majority of the other master nodes, it enters an error state and stops accepting writes.
 
