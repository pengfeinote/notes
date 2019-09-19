1. redis需要设置过期时间(条件允许可以打散过期时间)
2. redis访问失败时需要做相应处理，不能仅仅打印日志
3. 例如hgetall、lrange、smembers、zrange、sinter等并非不能使用，但是需要明确N的值。有遍历的需求可以使用hscan、sscan、zscan代替。
4. 禁止线上使用keys、flushall、flushdb等，通过redis的rename机制禁掉命令，或者使用scan的方式渐进式处理。
5. 批量操作提高效率：mset mget pipeline(原生是原子操作，pipeline是非原子操作; pipeline可以打包不同的命令，原生做不到)
6. 多个服务不能使用同一个redis实例，导致相互影响，公用数据服务化
7. 使用带有连接池的数据库，可以有效控制连接，同时提高效率，标准使用方式
```
Jedis jedis = null;
try {
    jedis = jedisPool.getResource();
    //具体的命令
    jedis.executeCommand()
} catch (Exception e) {
    logger.error("op key {} error: " + e.getMessage(), key, e);
} finally {
    //注意这里不是关闭连接，在JedisPool模式下，Jedis会被归还给资源池。
    if (jedis != null) 
        jedis.close();
}
```
8. 内存淘汰策略
	* allkeys-lru：根据LRU算法删除键，不管数据有没有设置超时属性，直到腾出足够空间为止。
	* allkeys-random：随机删除所有键，直到腾出足够空间为止。
	* volatile-random:随机删除过期键，直到腾出足够空间为止。
	* volatile-ttl：根据键值对象的ttl属性，删除最近将要过期数据。如果没有，回退到noeviction策略。
	* noeviction：不会剔除任何数据，拒绝所有写入操作并返回客户端错误信息"(error) OOM command not allowed when used memory"，此时Redis只响应读操作。
9. redis间数据同步可以使用：redis-port
