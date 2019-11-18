## ThreadLocal


### 使用场景

每个线程保存一份副本

```java

ThreadLocal<String> threadLocal = new ThreadLocal();
threadLocal.set("wpf");

String name = threadLocal.get();

```


### 数据结构设计

每个Thread中保存了一个ThreadLocals的成员变量，变量本身是一个map结构，key是ThreadLocal对象的弱引用，value是实际存放的值

ThreadLocalMap并没有使用链表解决冲突，而是通过再hash(i + 1)解决冲突，再hash时，如果碰到key为null的，就将value清除，避免内存泄漏


### jvm中的引用(强引用/软引用/弱引用)

强引用：平常new对象时使用的就是强引用。

	String name = new String("wpf");

软引用SoftReference：

	当jvm内存不足时，会回收弱引用引用的对象内存。可以用于一些内存缓存，位图缓存中使用


弱引用WeakReference:

	jvm gc时，不会考虑弱引用对对象的引用，即如果对象被所有除弱引用之外的引用解引用，则会被gc回收。在回收之后，reference get的对象将会返回null

ThreadLocal中为什么使用弱引用：

	如果使用强引用，则线程存活时，ThreadLocalMap会一直存活，则对应的ThreadLocal对象会一直在内存中。

内存泄漏风险：

	如果ThreadLocal为key的弱引用包含的对象被回收，那么ThreadLocalMap中会存在key为null的entry，因为value在map中，所以value一直不会被回收，不过jdk为了避免这种情况，在get，set等碰到key为null的entry时，均会将value指向null避免内存泄漏，不过保险起见，还是建议不用时通过remove删除ThreadLocalMap中对应的entry
