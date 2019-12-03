## mysql 多版本控制协议

多版本控制协议(Multi-version Concurrency Control)是数据库系统为实现事务隔离性而经常采用的一种协议，协议简称为MVCC。Mysql、Oracle等主流数据库系统一般都采用该协议来实现事务的隔离性。主要思想是对于一条数据，根据版本号生成多条记录，通过记录的版本来控制可见性等问题。

在mysql的innodb中，在RR、RC级别会使用mvcc来提升并发。

### 数据记录的版本链

一般情况下，mysql一行记录默认会有以下四个字段：

* ROW_ID:  记录ID，如果一行记录没有聚簇索引，mysql会默认生成一列
* TRX_ID:  最后更新/插入/删除这个记录的事务ID
* ROLL_PTR: 指向更新前记录在undo log中的位置
* Delete flag: 指示记录是否被删除的flag

其中RoW_ID与MVCC无关，仅讨论剩余三个字段。

当Insert一条记录时，新记录的TRX_ID为插入事务的ID，ROLL_PTR为空，删除flag为0。当有事务更新该记录时，mysql首先会将更新前的记录写进Undo log，然后将新纪录加入到数据表中，新纪录的TRX_ID为插入新纪录的事务ID，ROLL_PTR指向undo log中更新前记录的位置，Delete Flag为0。当有事务删除记录时，mysql会将该记录写入undo log中，复制一份新纪录在数据表中，新纪录的TRX_ID为删除记录的事务ID，ROLL_PTR指向删除前记录在Undo log中的位置，新纪录的删除标志位设为1。

mysql后台有purge线程，当mysql发现系统中最老的事务已经比被删除记录的事务大的时候，就会清理被删除的记录，同理，也会适时清理undo log中的数据。

### ReadView

从上面的内容可知，mvcc中mysql会为每一个记录生成一个版本链，那么，mysql如何判断一个事务能否能访问版本链中的一条数据呢？对于每一个事务，mysql会生成一个ReadView用于快照读，数据库根据ReadView来判断一条记录对一个事务的可见性。

ReadView包含一下几方面信息：

* m\_trx\_ids: 当前数据库活跃的事务id
* low\_min\_id: 当前活跃事务id的最小值
* up\_limit\_id: ReadView生成时刻系统尚未分配的下一个事务ID(这里为什么是下一个事务ID？因为更老的事务可能还在活跃，更新的事务可能已经提交，所以不能根据当前活跃事务的最大ID来判断)
* m\_creator\_id: 创建该ReadView的Id

当当前事务select时，根据以下规则判断可见性：

* 如果记录的trx\_id < low\_min\_id，则表明创建ReadView时，此记录已经Commit，可见
* 如果记录的trx\_id = m_creator\_id，则表明该记录由创建事务本身创建，可见
* 如果记录的trx\_id >= up\_limit\_id，则表明该记录是创建Readview之后的新事务创建的，不可见
* 如果trx\_id在m\_trx\_ids中，则记录不可见（因为相关事务还在活跃中），否则可见

### Undo-log

mysql中undo log除了rollback时使用，mvcc也使用了rollback。undo log分为两种:

* insert undo log

事务在insert时产生的undo log，只有在事务回滚时需要，事务提交后可以立即丢弃

* update/delete undo log

事务在进行update或delete时产生的undo log，版本链中旧的记录会存放在undo log中，在快照读中会用到。mysql后台有专门的purge线程，在适当的时机清楚版本链中已经不需要的记录

>注： Undo log用于rollback+mvcc，redo log用于持久性保障，当事务执行中未落盘宕机时，通过redo log做落盘保证

### RR与RC下的MVCC

RR(可重复度)与RC(读一提交)在MVCC上的主要区别是readView生成时机不同。

* RR隔离级别下，会在事务开启时创建ReadView，这个ReadView会一直持续到事务结束，因此，事务不会看到开启后其他事务的修改结果
* RC隔离级别下，每个select会单独生成一个ReadView，第一个select时事务可能在活跃事务列表中，第二个select时可能已经不在了，所以存在不可重复读（即两次读结果不一致）的问题。

