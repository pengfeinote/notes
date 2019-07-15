## kafka设计与架构

### kafka为什么快
得益于kafka partition的设计.  
kafka中，一个topic被分为多个partition，每个partition又可以分为多个segment.  
1. <b>partition的结构</b>.  
<p>
一个topic可以有多个partition，创建时指定（可以修改），每个partition再broker集群中是一个目录，partition的命名规则：topic名称+有序序号，第一个序号从0开始。partition是物理概念，topic是逻辑概念
</p>
2. <b>为什么使用segment</b>.  
<p>
一个partition会被分割为多个segment文件，每个segment文件的大小固定（通过log.segment.bytes设置)。之所以要有segment，如果partition只有一个文件的话，可能存在文件无限增大的可能，不方便维护，也不利于文件的读写性能
</p>
3. <b>segment的组织方式，消息的定位方式</b>
<p>
一个partition在一台kafka节点上是一个文件夹的形式，文件夹中有很多文件叫做segment，producers生产消息的过程，本质上是写segment文件的过程，kafka之所以快，是因为kafka按照磁盘顺序写入文件，当然，读取消息也是按照磁盘顺序，对于计算机系统而言，按照磁盘顺序读写文件性能甚至高于随机读写内存(省去了内存管理，gc的问题)。
</p>
<p>
那么segment是以何种方式组织的呢？首先，segment是以其所在partition的offset命名的，例如：  
1. 00000000000000000000.index  
2. 00000000000000000000.log  
3. 00000000000000170410.index  
4. 00000000000000170410.log  
5. 00000000000000239430.index  
6. 00000000000000239430.log
.index文件存储元数据，.log存储实际的消息数据。.index文件的元数据存储了消息实体在.log文件中的实际物理偏移。其中以“.index”索引文件中的元数据[3, 348]为例，在“.log”数据文件表示第3个消息，即在全局partition中表示170410+3=170413个消息，该消息的物理偏移地址为348。
</p>
<p>
那么kafka是如何定位消息的磁盘位置呢？以上表为例，读取offset=170418的消息，首先查找segment文件，其中00000000000000000000.index为最开始的文件，第二个文件为00000000000000170410.index(起始偏移为170410+1=170411)，而第三个文件为00000000000000239430.index(起始偏移为239430+1=239431)，所以这个offset=170418就落到了第二个文件之中。其他后续文件可以依次类推，以其实偏移量命名并排列这些文件，然后根据二分查找法就可以快速定位到具体文件位置。其次根据00000000000000170410.index文件中的[8,1325]定位到00000000000000170410.log文件中的1325的位置进行读取。
</p>

<p>
要是读取offset=170418的消息，从00000000000000170410.log文件中的1325的位置进行读取，那么怎么知道何时读完本条消息，否则就读到下一条消息的内容了?
这个就需要联系到消息的物理结构了，消息都具有固定的物理结构，包括：offset(8 Bytes)、消息体的大小(4 Bytes)、crc32(4 Bytes)、magic(1 Byte)、attributes(1 Byte)、key length(4 Bytes)、key(K Bytes)、payload(N Bytes)等等字段，可以确定一条消息的大小，即读取到哪里截止。(各个字段待研究)
</p>
### 高可靠性的保证（主从同步机制）
<p>
在使用kafka创建topic的时候，可以指定复制因子的参数，参数指定了partition的副本数，默认为1，但为可靠性考虑，生产环境一般大于1，多个副本中有一个Leader副本，其他为Follower副本，Follower副本会从Leader副本拉取消息更新到本地，kafka会维护一个同步副本队列（ISR），如果某个follower落后太多，则从队列中剔除
</p>
名词解释：     
1. <b>AR(Assigned Replica)：副本合集.</b>
<p>
OR = ISR + OSR
</p>
2. <b>OSR(Out-Sync Replica)：因更新太慢而脱离更新的副本合集.</b>   
<p>
 kafka broker的配置中，有replica.lag.time.max.ms和replica.lag.max.messages两个参数（kafka 0.10.x版本后移除了replica.lag.max.messages参数,前一个参数表示Follower最多可以落后Leader多长时间，超过这个时间，Follower将会被从ISR中移除，加入OSR中.第二个参数表示，最大允许的消息条数.
</p> 
<p>
kafka之所以在配置中删除了消息条数这个配置，为了防止producers在负载较高的情况下，一次性写入大量消息，导致副本频繁失效的情况
</p>
3. <b>ISR(In-Sync Replica)： 在更新的副本合集.</b>   
4. <b>HW(HighWater)：consumer可见的最大offset.</b>
<p>
每个副本都有一个HW，整个partition的HW取ISR的最小HW
</p>
5. <b>LEO(LogEndOffset)：broker当前写的最大offset.</b>  
<p>
主从同步详细说明,producer生产消息到broker后,partition leader首先更新消息，然后将阻塞的followers解锁，通知ISR中的followers有新消息，然后ISR中的follower从leader取出最新消息更新，如果所有的follower复制完1条或者n条消息，则leader将整个partition的HW更新至n.  
由此可见，Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，同步复制要求所有能工作的follower都复制完，这条消息才会被commit，这种复制方式极大的影响了吞吐率。而异步复制方式下，follower异步的从leader复制数据，数据只要被leader写入log就被认为已经commit，这种情况下如果follower都还没有复制完，落后于leader时，突然leader宕机，则会丢失数据。而Kafka的这种使用ISR的方式则很好的均衡了确保数据不丢失以及吞吐率。
kafka的ISR管理最终都会落到zookeeper上,目前会有两个地方会对zk的节点进行维护：  
1. <b>Controller来维护</b>：Kafka集群中的其中一个Broker会被选举为Controller，主要负责Partition管理和副本状态管理，也会执行类似于重分配partition之类的管理任务。在符合某些特定条件下，Controller下的LeaderSelector会选举新的leader，ISR和新的leader_epoch及controller_epoch写入Zookeeper的相关节点中。同时发起LeaderAndIsrRequest通知所有的replicas。  
2. <b>leader来维护</b>：leader有单独的线程定期检测ISR中follower是否脱离ISR, 如果发现ISR变化，则会将新的ISR的信息返回到Zookeeper的相关节点中。

副本不同步的情况：
1. 受限于I/O，follower追加消息的速度慢于leader
2. follower卡住，因为GC或者进程失效的原因
3. 新启动副本：当用户给主题增加副本因子时，新的follower不在同步副本列表中，直到他们完全赶上了leader日志。
</p>

#### producers发送确认

request.required.acks=1  
这意味着producer在ISR中的leader已成功收到的数据并得到确认后发送下一条message。如果leader宕机了，则会丢失数据。
request.required.acks=0  
这意味着producer无需等待来自broker的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
request.required.acks=-1  
producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当ISR中只有leader时(前面ISR那一节讲到，ISR中的成员由于某些情况会增加也会减少，最少就只剩一个leader)，这样就变成了acks=1的情况。  
如果要提高数据的可靠性，在设置request.required.acks=-1的同时，也要min.insync.replicas这个参数(可以在broker或者topic层面进行设置)的配合，这样才能发挥最大的功效。min.insync.replicas这个参数设定ISR中的最小副本数是多少，默认值为1，当且仅当request.required.acks参数设置为-1时，此参数才生效。如果ISR中的副本数少于min.insync.replicas配置的数量时，客户端会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。

kafka还可以调整producer的发送模式，通过producer.type可以指定producer同步或是异步发送消息，默认是producer.type=sync同步发送消息，异步发送消息（producer.type=async）即以batch的方式发送数据，可以极大提高kafka的发送效率，减少网络请求和磁盘io次数，不过提高了消息丢失的风险。

以batch的方式推送数据，producer会在消息在内存中累计到一定数量后一次性发送给broker，batch的数量大小可以通过producer的参数(batch.num.messages)控制。通过增加batch的大小，可以减少网络请求和磁盘IO的次数，当然具体参数设置需要在效率和时效性方面做一个权衡。在比较新的版本中还有batch.size这个参数。

#### HW机制的使用
使用HW保证broker性能的同时，可以作为失效节点的checkPoint.  
如上图，某个topic的某partition有三个副本，分别为A、B、C。A作为leader肯定是LEO最高，B紧随其后，C机器由于配置比较低，网络比较差，故而同步最慢。这个时候A机器宕机，这时候如果B成为leader，假如没有HW，在A重新恢复之后会做同步(makeFollower)操作，在宕机时log文件之后直接做追加操作，而假如B的LEO已经达到了A的LEO，会产生数据不一致的情况，所以使用HW来避免这种情况。

A在做同步操作的时候，先将log文件截断到之前自己的HW的位置，即3，之后再从B中拉取消息进行同步。

如果失败的follower恢复过来，它首先将自己的log文件截断到上次checkpointed时刻的HW的位置，之后再从leader中同步消息。leader挂掉会重新选举，新的leader会发送“指令”让其余的follower截断至自身的HW的位置然后再拉取新的消息。

当ISR中的个副本的LEO不一致时，如果此时leader挂掉，选举新的leader时并不是按照LEO的高低进行选举，而是按照ISR中的顺序选举。

#### Leader选举    

<p>
如果leader宕机了，如何从follower中选举中新的leader，是一个非常重要的问题。该follower必须具有所有leader的消息，才能保证消息不丢失，如果leader等待所有follower确认才提交消息，又回极大的降低系统吞吐率。
</p>
<p>
一种非常常用的选举leader的方式是“少数服从多数”，Kafka并不是采用这种方式。这种模式下，如果我们有2f+1个副本，那么在commit之前必须保证有f+1个replica复制完消息，同时为了保证能正确选举出新的leader，失败的副本数不能超过f个。这种方式有个很大的优势，系统的延迟取决于最快的几台机器，也就是说比如副本数为3，那么延迟就取决于最快的那个follower而不是最慢的那个。“少数服从多数”的方式也有一些劣势，为了保证leader选举的正常进行，它所能容忍的失败的follower数比较少，如果要容忍1个follower挂掉，那么至少要3个以上的副本，如果要容忍2个follower挂掉，必须要有5个以上的副本。也就是说，在生产环境下为了保证较高的容错率，必须要有大量的副本，而大量的副本又会在大数据量下导致性能的急剧下降。这种算法更多用在Zookeeper这种共享集群配置的系统中而很少在需要大量数据的系统中使用的原因。HDFS的HA功能也是基于“少数服从多数”的方式，但是其数据存储并不是采用这样的方式。
</p>
<p>
实际上，leader选举的算法非常多，比如Zookeeper的Zab、Raft以及Viewstamped Replication。而Kafka所使用的leader选举算法更像是微软的PacificA算法。
</p>
<p>
Kafka在Zookeeper中为每一个partition动态的维护了一个ISR，这个ISR里的所有replica都跟上了leader，只有ISR里的成员才能有被选为leader的可能(unclean.leader.election.enable=false)。在这种模式下，对于f+1个副本，一个Kafka topic能在保证不丢失已经commit消息的前提下容忍f个副本的失败，在大多数使用场景下，这种模式是十分有利的。事实上，为了容忍f个副本的失败，“少数服从多数”的方式和ISR在commit前需要等待的副本的数量是一样的，但是ISR需要的总的副本的个数几乎是“少数服从多数”的方式的一半。
</p>
<p>
这就需要在可用性和一致性当中作出一个简单的抉择。如果一定要等待ISR中的replica“活”过来，那不可用的时间就可能会相对较长。而且如果ISR中所有的replica都无法“活”过来了，或者数据丢失了，这个partition将永远不可用。选择第一个“活”过来的replica作为leader,而这个replica不是ISR中的replica,那即使它并不保障已经包含了所有已commit的消息，它也会成为leader而作为consumer的数据源。默认情况下，Kafka采用第二种策略，即unclean.leader.election.enable=true，也可以将此参数设置为false来启用第一种策略。
</p>
<p>
unclean.leader.election.enable决定了不再ISR中的follower是否能被选举为leader
</p>
<p>
假设某个partition中的副本数为3，replica-0, replica-1, replica-2分别存放在broker0, broker1和broker2中。AR=(0,1,2)，ISR=(0,1)。</br>
设置request.required.acks=-1, min.insync.replicas=2，unclean.leader.election.enable=false。这里讲broker0中的副本也称之为broker0起初broker0为leader，broker1为follower。
</p>
<p>
当ISR中的replica-0出现crash的情况时，broker1选举为新的leader[ISR=(1)]，因为受min.insync.replicas=2影响，write不能服务，但是read能继续正常服务。此种情况恢复方案：
</br>
</p>
1. 尝试恢复(重启)replica-0，如果能起来，系统正常;   
2. 如果replica-0不能恢复，需要将min.insync.replicas设置为1，恢复write功能.

<p>    
当ISR中的replica-0出现crash，紧接着replica-1也出现了crash, 此时[ISR=(1),leader=-1],不能对外提供服务，此种情况恢复方案：
</p>
1. 尝试恢复replica-0和replica-1，如果都能起来，则系统恢复正常;   
2. 如果replica-0起来，而replica-1不能起来，这时候仍然不能选出leader，因为当设置unclean.leader.election.enable=false时，leader只能从ISR中选举，当ISR中所有副本都失效之后，需要ISR中最后失效的那个副本能恢复之后才能选举leader, 即replica-0先失效，replica-1后失效，需要replica-1恢复后才能选举leader。保守的方案建议把unclean.leader.election.enable设置为true,但是这样会有丢失数据的情况发生，这样可以恢复read服务。同样需要将min.insync.replicas设置为1，恢复write功能;
3. replica-1恢复，replica-0不能恢复，这个情况上面遇到过，read服务可用，需要将min.insync.replicas设置为1，恢复write功能;
4. replica-0和replica-1都不能恢复，这种情况可以参考情形

<p>
当ISR中的replica-0, replica-1同时宕机,此时[ISR=(0,1)],不能对外提供服务，此种情况恢复方案：尝试恢复replica-0和replica-1，当其中任意一个副本恢复正常时，对外可以提供read服务。直到2个副本恢复正常，write功能才能恢复，或者将将min.insync.replicas设置为1.</p>
### 高可靠性分析

#### 消息传输保障
<p>
前面已经介绍了Kafka如何进行有效的存储，以及了解了producer和consumer如何工作。接下来讨论的是Kafka如何确保消息在producer和consumer之间传输。有以下三种可能的传输保障(delivery guarantee):
</p>
* At most once: 消息可能会丢，但绝不会重复传输
* At least once：消息绝不会丢，但可能会重复传输
* Exactly once：每条消息肯定会被传输一次且仅传输一次

Kafka的消息传输保障机制非常直观。当producer向broker发送消息时，一旦这条消息被commit，由于副本机制(replication)的存在，它就不会丢失。但是如果producer发送数据给broker后，遇到的网络问题而造成通信中断，那producer就无法判断该条消息是否已经提交(commit)。虽然Kafka无法确定网络故障期间发生了什么，但是producer可以retry多次，确保消息已经正确传输到broker中，所以目前Kafka实现的是at least once。

consumer从broker中读取消息后，可以选择commit，该操作会在Zookeeper中存下该consumer在该partition下读取的消息的offset。该consumer下一次再读该partition时会从下一条开始读取。如未commit，下一次读取的开始位置会跟上一次commit之后的开始位置相同。当然也可以将consumer设置为autocommit，即consumer一旦读取到数据立即自动commit。如果只讨论这一读取消息的过程，那Kafka是确保了exactly once, 但是如果由于前面producer与broker之间的某种原因导致消息的重复，那么这里就是at least once。

考虑这样一种情况，当consumer读完消息之后先commit再处理消息，在这种模式下，如果consumer在commit后还没来得及处理消息就crash了，下次重新开始工作后就无法读到刚刚已提交而未处理的消息，这就对应于at most once了。

读完消息先处理再commit。这种模式下，如果处理完了消息在commit之前consumer crash了，下次重新开始工作时还会处理刚刚未commit的消息，实际上该消息已经被处理过了，这就对应于at least once。

要做到exactly once就需要引入消息去重机制。

kafka支持的更多是at least once

#### 重复消息

如上一节所述，Kafka在producer端和consumer端都会出现消息的重复，这就需要去重处理。

Kafka文档中提及GUID(Globally Unique Identifier)的概念，通过客户端生成算法得到每个消息的unique id，同时可映射至broker上存储的地址，即通过GUID便可查询提取消息内容，也便于发送方的幂等性保证，需要在broker上提供此去重处理模块，目前版本尚不支持。

针对GUID, 如果从客户端的角度去重，那么需要引入集中式缓存，必然会增加依赖复杂度，另外缓存的大小难以界定。

不只是Kafka, 类似RabbitMQ以及RocketMQ这类商业级中间件也只保障at least once, 且也无法从自身去进行消息去重。所以我们建议业务方根据自身的业务特点进行去重，比如业务消息本身具备幂等性，或者借助Redis等其他产品进行去重处理。

#### 高可靠性配置
要保证数据写入到Kafka是安全的，高可靠的，需要如下的配置：  

* topic的配置：replication.factor>=3,即副本数至少是3个;2<=min.insync.replicas<=replication.factor
* broker的配置：leader的选举条件unclean.leader.election.enable=false
* producer的配置：request.required.acks=-1(all)，producer.type=sync
