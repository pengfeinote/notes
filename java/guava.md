## guava特性

### guava cache

#### guava cache刷新机制

guava缓存的过期分为一下三种情况：

* expireAfterAccess: 访问一段时间后过期
* expireAfterWrite: 写入一段时间后过期
* refreshAfterWrite: 一段时间后刷新

**expireAfterWrite和refreshAfterWrite的区别:**

在LoadingCache中两者功能类似，区别是expireAfterWrite，当value expire时，cache会block住等待新值load，而refreshAfterWrite如果refresh nano已经过了，但是新值还没有load，还是会返回旧值

当一个key可以在一定时间(比如：2秒)内容忍旧值，但容忍时间有限(3秒)时，可以使用expireAfterWrite(3 Second).refreshAfterWrite(2 Second)。此时在两秒以内，返回可用值，2-3秒之间，如果refresh结束返回新值，否则返回旧值，3秒之后如果新值还没load成功等待新值load成功返回

>**注**：注意guava获取nano时间的方式：System.nanoTime()