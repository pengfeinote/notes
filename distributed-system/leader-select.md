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

eureka是无中心的分布式系统，如果某个节点宕机(节点间通过心跳来探测可用性)，client仍然可以尝试连接其他的server，只不过不能保证数据的一致性。因此Eureka相对zookeeper是AP，而ZK是CP

### kafka

使用的不是少数服从多数的原则，如何保证不脑裂，AP/CP?，一致性的保障

#### kafka partition leader的选举

Kafka的Brokers会选举出某些Broker作为Controller，Controller负责topic replica的重新分配、partition leader的选举等（Controller的高可用如何保证）。

当分区中的leader副本不可用时，kafka controller会根据分区选择算法从ISR中选择某个分区作为新的leader，通过rpc的方式将新leader及isr的信息写入zk

#### kafka controller leader的选举

kafka启动后，会选出一个controller leader，controller leader会在zk指定路径上创建一个临时节点，其他节点则会watch该临时节点，当controller失效后，临时节点被删除，其他broker会尝试创建新的临时节点，但只有一个broker会创建成功，创建成功的成为新的controller leader，其他节点继续watch新的临时节点。

#### kafka controller的脑裂

如果controller leader假死（比如gc导致长时间STW,zk session timeout），此时zk删除临时节点，其他节点竞选leader成功，而此时旧的leader又苏醒，则会同时存在两个controller），此时zk删除临时节点，其他节点竞选leader成功，而此时旧的leader又苏醒，则会同时存在两个controller leader。

每当新的controller产生的时候就会在zk中生成一个全新的、数值更大的controller epoch的标识，并同步给其他的broker进行保存，这样当第二个controller发送指令时，其他的broker就会自动忽略。

#### kafka中的CAP

AP/CP: kafka通过可配置的方式选择AP或是CP，当某分区ISR中所有的Follower都宕机时，可以有两种选项：

* 等待ISR中某个follower活过来，选举它为leader
* 选举任一或者的Follower作为Leader

第一种情况选择了一致性而忽略了可用性；第二种情况选择了可用性，但数据有可能不一致。使用参数unclean.leader.election.enable控制选择

### rabbitmq

rabbitmq的master queue失效后，会将时间最长的镜像队列提升为master，这里rabbitmq基于一个假设，运行时间最长的镜像队列跟旧master同步的可能行最高。

新特生的镜像队列会将所有未被consumer确认消费的消息重新入队，如果消费者的ack在到达旧master前丢失，或是在旧master同步到mirrors前丢失，新的master只能将未收到ack的消息重新入队，因此，consumer要假设可能收到重复消息，消费消息时进行幂等处理。

当一个镜像队列变成新的master之后，新master上的消息都会得以保存，同时，新master上的消息也会同步到其他mirror。

如果producer开启了确认模式，master失效后，新master收到消息后仍然会发送确认消息给producer

如果开启自动确认模式，consumer可能面临丢失数据的问题，master发送消息之后即认为发送成功，如果此时master失效，consumer同master的连接断开，新的master并不会重新发送消息。可以设置consumer cancellation notification，当连接断开时，consumer可以收到提醒。如果吞吐量要求大于数据一致性，开启自动确认是个不错的选择

### redis
