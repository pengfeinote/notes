innodb_file_per_table
每张表一个表空间，彼此独立

innodb_lock_wait_timeout

innodb_flush_log_at_trx_commit

innodb_buffer_pool_size:
	这个参数主要作用是缓存innodb表的索引，数据，插入数据时的缓冲

	默认值：128M

	专用mysql服务器设置的大小： 操作系统内存的70%-80%最佳。

	设置方法：

	my.cnf文件

	innodb_buffer_pool_size = 6G

	此外，这个参数是非动态的，要修改这个值，需要重启mysqld服务。

	所以设置的时候要非常谨慎。

	并不是设置的越大越好。设置的过大，会导致system的swap空间被占用，导致操作系统变慢，从而减低sql查询的效率。
innodb_buffer_pool_instances
	摘要：1 innodb_buffer_pool_instances可以开启多个内存缓冲池，把需要缓冲的数据hash到不同的缓冲池中，这样可以并行的内存读写。
	
	
	http://www.importnew.com/28256.html
	
	MySQL变量的概念

个人认为可以理解成MySQL在启动或者运行过程中读取的一些参数问题，利用这些参数来启动服务、响应或者支持用户的请求等

变量的配置
如果打算长期使用，应该写入配置文件，而不是在命中指定，因为在命中设置的变量会随着MySQL服务的重启而恢复默认值
另外要注意是设置的当前Session的变量还是全局的变量。

变量单位
不同的变量的单位不同，比如table_cache是指缓存的表的个数，而key_buffer_size则是以字节为单位
另外还有以页或者百分比为单位的变量
许多变量可以通过后缀制订单位，
比如1M表示一百万字节，在配置文件中或者在命令行下有效，
但是在使用set命令的时候，这些单位就无效，必须使用数字，单位为字节
比如：set @@session.sort_buffer_size = 1024*1024或者set @@session.sort_buffer_size = 1048576
但是配置文件中设置的时候就不能使用表达式

变量的作用域 
有些变量的作用是服务器级别的，有些是Session级别的，剩下的的一些是对象级别的。
许多回话的变量是全局变量相等，可以为是默认值
如果改变会话级的变量，它只影响当前Session，当前Session关闭后当前设置的参数会失效
举例：
query_cache_size是全局级的
sort_buffer_size可以在全局级设置，每个Session也可以独立设置
join_buffer_size可以在全局级设置，也可以在Session级设置，一个查询中如果有多个表关联，可以为每个关联分配一个join buffer
除了在配置文件中设置变量之外，（部分变量）也可以在运行时修改，MySQL称之为动态配置变量
比如：　set global sort_buffer_size = 1024*1024*1024
set sort_buffer_size = 1024*1024*1024
set @@sort_buffer_size = 1024*1024*1024
set @@session.sort_buffer_size = 1024*1024*1024
set @@global.sort_buffer_size = 1024*1024*1024

常见变量的设置与获取资源说明：
key_buffer_size
为键缓冲区（key buffer，也叫键缓存key cache）分配所有指定的空间，
操作系统不会为该设置立马分配内存，而是等到使用的时候才分配。
table_cache_size
当有线程打开表时，MySQL会检查这个标量的值，如果大于缓存中表的数量，线程可以把最先打开的表放入缓存，
如果该值比缓存中的表数小，MySQL将从缓存中删除不常用的表
thread_cache_size
当有连接关闭时，MySQL检查缓存中是否还有空间来缓存线程。
如果有:则缓存改线程已被下次连接重用
如果没有：他讲销毁改线程而不再缓存，
缓存中使用的线程数，不会立即减少，只有在新的连接删除缓存中的一个线程并使用后才会减少
MySQL只在关闭连接时候才在缓冲中增减线程，在创建新的连接的时候才从缓存中删除线程
query_cache_size
MySQL启动的时候，一次性分配并且初始化这块内存，如果修改这个变量（即使设置为与当前值一样）
MySQL会立刻删除所有缓存的查询，重新分配这片缓存到指定大小，并且重新初始化内存
read_buffer_size
MySQL只会在查询需要时才会为该缓存分配内存，并且会一次性分配改参数指定大小的全部内存
read_rnd_buffer_size
MySQL只会在查询需要时才会为该缓存分配内存，并且只分配需要的内存大小而不是全部指定的大小
应该叫做，max_read_rnd_buffer_size
sort_buffer_size
MySQL只会在查询需要做排序操作的时候拆毁为该缓存分配内存，
一旦需要排序，MySQL就会立刻分配给改参数指定大小的全部内存，而不管排序是否需要这么大的内存。

由此可见，不用的变量，设置之后的启用时间，启用原理，生效方式等都是有一定差异的。

innodb_locks_unsafe_for_binlog
innodb默认使用了next-key锁，它结合了record锁和gap锁,正因为这样的锁算法，innodb可以在可重复读的级别下一定程度的避免幻读,	该参数主要设置innodb是否对gap加锁
如果该参数enabled，则是unsafe，不会对gap加锁。当然对于一些和数据完整性相关的定义，如外键和唯一索引（含主键）需要对gap进行加锁，那么innodb_locks_unsafe_for_binlog的设置并不会影响gap是否加锁。innodb引入了一个概念叫做“semi-consistent”，这样会在innodb_locks_unsafe_for_binlog的状态为ennable时在一定程度上提高update并发性。

设置变量的潜在的影响
动态设置全局变量可能会导致意外的副作用。
某些变量改变后会立即生效，比如从缓冲中刷新赃块,从而引起服务器相应请求的一些不稳定甚至更严重的问题
