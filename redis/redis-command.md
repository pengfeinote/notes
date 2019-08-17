ZSET: containing a skip list and hash table

	ZADD: O(lgN)
		zadd key score element
	ZRANGE(WITHSCORES)(ZREVRANGE):O(log(N)+M)， N 为有序集的基数，而 M 为结果集的基数
		0 -1:返回全部元素
		
	ZRANGEBYSCORE: O(log(N) + M)  N 为有序集的基数， M 为被结果集的基数。
		ZRANGEBYSCORE zset (1 5: 返回所有符合条件 1 < score <= 5 的成员
		
	
	ZRANK(ZREVRANK): O(lgN)
		返回有序集 key 中成员 member 的排名。其中有序集成员按 score 值递增(从小到大)顺序排列。
		
	
	ZRANGEBYLEX
		当有序集合的所有成员都具有相同的分值时， 有序集合的元素会根据成员的字典序（lexicographical ordering）来进行排序， 而这个命令则可以返回给定的有序集合键 key 中， 值介于 min 和 max 之间的成员。
		合法的 min 和 max 参数必须包含 ( 或者 [ ， 其中 ( 表示开区间（指定的值不会被包含在范围之内）， 而 [ 则表示闭区间（指定的值会被包含在范围之内）。特殊值 + 和 - 在 min 参数以及 max 参数中具有特殊的意义， 其中 + 表示正无限， 而 - 表示负无限。 因此， 向一个所有成员的分值都相同的有序集合发送命令 ZRANGEBYLEX <zset> - + ， 命令将返回有序集合中的所有元素。
	
	ZCARD: O(1)
		查询zset元素个数
	ZCOUNT: O(lg(N))
		Returns the number of elements in the sorted set at key with a score between min and max.
	ZINCRBY
	
	ZPOPMAX(ZPOPMIN)


scan系列命令与range系列命令：
scan: scan hscan sscan zscan
range: lrange getrange hgetall keys等等
以keys命令为例，keys命令要对所有的redis key进行遍历，时间复杂度是O(N)，因为redis是单线程的，如果数据集比较大，keys操作可能会阻塞比较长的时间
scan是一个非阻塞的命令，是一个基于游标的迭代器，这意味着命令每次被调用都需要使用上一次这个调用返回的游标作为该次调用的游标参数，以此来延续之前的迭代过程。
scan语法：SCAN cursor [MATCH pattern] [COUNT count]The default COUNT value is 10
例： scan 0 key111* count 20 
查找前缀为key111的所有key,命令中的20指的不是结果个数，而是遍历redis字典槽位的数量
这个命令会返回两部分，第一部分是下一次便利的位置，第二部分是结果列表，比如第一部分返回1024，则下次命令应为scan 1024 key111* count 20，直到第一部分返回0，遍历结束

hscan: hashmap的key scan
zscan: 用于zset
sscan: 用于set

scan原理： https://www.jb51.net/article/148698.htm