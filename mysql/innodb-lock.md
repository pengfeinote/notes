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

mysql有两种机制来避免和处理死锁，一种是lock_wait_timeout机制，当等待锁超过一定时间时，有一个事务会回滚；另外，mysql有wait-for graph的机制，

**死锁成因**

死锁成因主要还是循环等待，以一下sql为例：

事务t1

```java
select * from t1 for update;

update t2 set name='xxx';

```

事务t2

```java
select * from t2 for update;

update t2 set name = 'xxx';

```

t1执行第一句之后（表t1上锁），t2执行第一句之后(表t2上锁)，其后t1执行第二句发现与表t2锁冲突开始等待，t2执行第二句发现与表t1锁冲突开始等待，导致循环等待，形成死锁。

上面是更新不同表导致的死锁问题，对不同索引进行更新，也会导致比较隐晦的死锁：

事务t1

```java
update msg set message='订单' where token >= 'aaa';

```

事务t2

```java
delete from msg where id >= 1;

```

t1查询会在token列上加gap锁，并且会在对应的聚簇索引上加record锁（为什么？防止被删掉），顺序可能是(1,4, 3),而t2会在聚簇索引上加锁，顺序可能是(1,3,4)，形成死锁

除了以上两种情况外，gap锁也有可能造成死锁

如表中token有('aaa', 'axd', 'cvs', 'asd', 'cvs')，排序后('aaa', 'asd', 'axd', 'cvs')

事务t1

```java
update msg set message='订单' where token='asd';

insert into msg values(null, 'aad', 'hello');
```

事务t2

```java

update msg set message='订单' where token='aaa';

insert into msg values(null, 'bsa', 'hello');

```

事务1会加[aaa, axd] gap锁，事务2会加[aaa, asd] gap锁(todo 加锁范围待验证)，形成死锁


**死锁检测**


**innodb死锁处理**


https://www.cnblogs.com/tartis/p/9366574.html

