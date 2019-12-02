## mysql 多版本控制协议

多版本控制协议(Multi-version Concurrency Control)是数据库系统为实现事务隔离性而经常采用的一种协议，协议简称为MVCC。Mysql、Oracle等主流数据库系统一般都采用该协议来实现事务的隔离性。主要思想是对于一条数据，根据版本号生成多条记录，通过记录的版本来控制可见性等问题。

在mysql的innodb中，在RR、RC级别会使用mvcc来提升并发。

### 记录链

一般情况下，mysql一行记录默认会有以下四个字段：

* ROW_ID:  记录ID，如果一行记录没有聚簇索引，mysql会默认生成一列
* TRX_ID:  最后更新/插入/删除这个记录的事务ID
* ROLL_PTR: 指向更新前记录在undo log中的位置
* Delete flag: 指示记录是否被删除的flag

其中RoW_ID与MVCC无关，仅讨论剩余三个字段。

当Insert一条记录时，新记录的TRX_ID为插入事务的ID，ROLL_PTR为空，删除flag为0。当有事务更新该记录时，mysql首先会将更新前的记录写进Undo log，然后将新纪录加入到数据表中，新纪录的TRX_ID为插入新纪录的事务ID，ROLL_PTR指向undo log中更新前记录的位置，Delete Flag为0。当有事务删除记录时，mysql会将该记录写入undo log中，复制一份新纪录在数据表中，新纪录的TRX_ID为删除记录的事务ID，ROLL_PTR指向删除前记录在Undo log中的位置，新纪录的删除标志位设为1。

mysql后台有purge线程，当mysql发现系统中最老的事务已经比被删除记录的事务大的时候，就会清理被删除的记录，同理，也会适时清理undo log中的数据。

### ReadView


### Undo-log与Redo-log


### 更新记录的主流程



### RR与RC下的MVCC
