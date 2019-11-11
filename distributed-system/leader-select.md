## 分布式系统leader选举协议

### zk

zk节点可以有以下四种状态：

* Looking

	在查找leader的过程中，如果集群已经有leader，则转为follower,如果没有，则开启一轮选举

* Leading

	节点本身处于Leader状态
	
* Following

	节点本身处于Follower状态

* Observing

	观察者状态，表明节点是Observer
	
zookeeper每次状态的变化都会接受一个zxid，如果zxid1 < zxid2，则zxid1的发生肯定早于zxid2。zxid是一个64位的数字，全局唯一，不断递增。

#### 启动时选举

系统启动时，所有节点都处于Looking状态，当looking节点找不到leader时，会进入第一轮选举过程。

以三台zk实例为例，id分别为(1, 2, 3)，首先启动实例(1, 2)，每个节点选取自己为leader，并同步给集群节点，节点1发出选取请求(1, 0)，节点2发出选取请求(2, 0)，每个节点进行以下步骤的对比，选出leader：

1. 查找zxid最大的选举
2. 如果zxid一样大，则选取id最大的zk节点

所以对于zk1而言，最终选择为节点2，zk2最终选择也为节点2。此时集群的服务器都统计出有两台机器接受了(2, 0)，大于集群数量的一半，此时便以选出ZK2为Leader

选出Leader后，各服务器会更新自己的状态，如果是Follower，则更新为Following，如果是Leader，则更新为Leading。当第三个节点启动时，发现已经选举出Leader，直接更新自己为Following

#### leader失效后选举

在zk集群运行期间，如果Leader节点挂了，那么zk暂停对外服务（所以zk是CP而非AP），进入leader选举：

1. 状态变更：非Observing更新状态为Looking，进入Leader选举
2. 发起投票：在运行期间，每个zk节点的zxid可能不同，比如发出投票依次为(1, 124), (3, 123)
3. 接收各服务器的投票
4. 处理投票：各节点都选出(1, 124)。（保证数据更新不丢失？因为最大zxid需要与leader同步）
5. 统计投票
6. 更新服务状态

### eureka


### kafka


### rabbitmq


### redis
