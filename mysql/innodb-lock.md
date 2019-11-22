## innodb锁

innodb默认事务隔离级别为RR(Repeatable read)，innodb在实现RR时，主要使用到了四种粒度锁：Record lock, Gap lock, Next-key lock, Insert-intention lock,从排他性上来分，可以分为s-lock(shared)和x-lock(exclude)
可以使用select * from innodb_locks查看innodb实例的锁情况

### record lock

记录锁，record lock锁住的是索引本身，而非记录，如果表中没有索引，则会在系统默认生成的聚簇索引上加锁

例如：select * from t where t.cid = 10 for update; update t set t.cid = 5 where t.cid = 10;

这两条会为t.cid为10的记录都加上排他的record lock，如果改为:
select * from t where t.cid = 10 lock in share mode.
则为t.cid为10的记录加上共享锁

### gap lock

gap锁锁住的是一个范围，当我们在查询select * from t where t.cid between 10 and 20时，gap锁会锁住区间(10,20)

### next-key lock

记录锁和间隙锁的结合，gap锁是全开区间，next-key锁则是半开半闭区间，在RR隔离级别下，mysql使用next-key来避免幻读

### insert-intention lock?

insert意向锁，是一种特殊的gap锁

### auto-incr lock

auto-increment是一个表级锁,当一个事务向带有自增主键的表插入记录时,其他事务必须等待第一个事务插入完成，以便能得到连续的auto-incr-pk值

### 加锁方式

* select * from ...

不加锁，不阻塞，通过mvcc读取快照数据，虽然解决了幻读问题，但是读取到的是历史数据

* select * from ... lock in share mode

加共享锁，读取的是当前数据，通过添加next-key锁解决幻读问题

* select * from ... for update

加排他锁，读取的是当前数据，通过添加next-key锁解决幻读问题,使用for update可以避免其他事务读取select的记录,举一个例子,如果两个事务都要对一个值加1，如果加共享锁，最终结果可能只加了1(读取的初始值相同)

* update ... where

加排他next-key锁，

* delete ... where

加排他next-key锁

* insert into ...

简单的insert会在主键上加入record lock排他锁，在insert之前，会加入一个意向gap锁，我理解这个意向gap锁允许其他意向gap锁插入数据（如果不冲突的话），但是不允许其他类似update的gap锁获取到锁


mysql中，mvcc本质上是乐观锁的实现，以上是悲观锁

### mysql死锁

mysql中存在三种锁,Record锁、gap锁以及next-key锁。在RC和RR隔离级别下，mysql会对事务加锁，在mysql commit或者rollback之后，会释放锁。

mysql有两种机制来避免和处理死锁，一种是

**死锁成因**


**死锁检测**


**innodb死锁处理**

https://www.cnblogs.com/tartis/p/9366574.html

