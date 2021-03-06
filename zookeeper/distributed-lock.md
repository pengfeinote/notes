## zk实现分布式锁

>**转：http://www.importnew.com/23025.html**

###锁服务

分布式锁用来为一组程序提供互斥机制。任意一个时刻仅有一个进程能够获得锁。分布式锁可以用来实现大型分布式系统的leader选举算法，即leader就是获取到锁的那个进程。

>不要把ZooKeeper的原生leader选举算法和我们这里所说的通用leader选举服务搞混淆了。ZooKeeper的原生leader选举算法并不是公开的算法，并不能向我们这里所说的通用leader选举服务那样，为一个分布式系统提供主进程选举服务。

为了使用ZooKeeper实现分布式锁，我们使用可排序的znode来实现进程对锁的竞争。思路其实很简单：首先，我们需要一个表示锁的znode，获得锁的进程就表示被这把锁给锁定了（命名为，/leader）。然后，client为了获得锁，就需要在锁的znode下创建ephemeral类型的子znode。在任何时间点上，只有排序序号最小的znode的client获得锁，即被锁定。例如，如果两个client同时创建znode /leader/lock-1和/leader/lock-2，所以创建/leader/lock-1的client获得锁，因为他的排序序号最小。ZooKeeper服务被看作是排序的权威管理者，因为是由他来安排排序的序号的。

锁可能因为删除了/leader/lock-1znode而被简单的释放。另外，如果相应的客户端死掉，使用ephemeral znode的价值就在这里，znode可以被自动删除掉。创建/leader/lock-2的client就获得了锁，因为他的序号现在最小。当然客户端需要启动观察模式，在znode被删除时才能获得通知：此时他已经获得了锁。

在lock的znode下创建名字为lock-的ephemeral类型znode，并记录下创建的znode的path（会在创建函数中返回）。
获取lock znode的子节点列表，并开启对lock的子节点的watch模式。
如果创建的子节点的序号最小，则再执行一次第2步，那么就表示已经获得锁了。退出。
等待第2步的观察模式的通知，如果获得通知，则再执行第2步。

###羊群效应

虽然这个算法是正确的，但是还是有一些问题。第一个问题是羊群效应。试想一下，当有成千成百的client正在试图获得锁。每一个client都对lock节点开启了观察模式，等待lock的子节点的变化通知。每次锁的释放和获取，观察模式将被触发，每个client都会得到消息。那么羊群效应就是指像这样，大量的client都会获得相同的事件通知，而只有很小的一部分client会对事件通知有响应。我们这里，只有一个client将获得锁，但是所有的client都得到了通知。那么这就像在网络公路上撒了把钉子，增加了ZooKeeper服务器的压力。

为了避免羊群效应，通知的范围需要更精准。我们通过观察发现，只有当序号排在当前znode之前一个znode离开时，才有必要通知创建当前znode的client，而不必在任意一个znode删除或者创建时都通知client。在我们的例子中，如果client1、client2和client3创建了znode/leader/lock-1、/leader/lock-2和leader/lock-3，client3仅在/leader/lock-2消失时，才获得通知。而不需要在/leader/lock-1消失时，或者新建/leader/lock-4时，获得通知。

###可恢复异常 Recoverable Exception

这个锁算法的另一个问题是没有处理当连接中断造成的创建失败。在这种情况下，我们根本就不知道之前的创建是否成功了。创建一个可排序的znode是一个非等幂操作，所以我们不能简单重试，因为如果第一次我们创建成功了，那么第一次创建的znode就成了一个孤立的znode了，将永远不会被删除直到会话结束。

那么问题的关键在于，在重新连接以后，client不能确定是否之前创建过lock节点的子节点。我们在znode的名字中间嵌入一个client的ID，那么在重新连接后，就可以通过检查lock znode的子节点znode中是否有名字包含client ID的节点。如果有这样的节点，说明之前创建节点操作成功了，就不需要再创建了。如果没有这样的节点，那就重新创建一个。

Client的会话ID是一个长整型数据，并且在ZooKeeper中是唯一的。我们可以使用会话的ID在处理连接丢失事件过程中作为client的id。在ZooKeeper的JAVA API中，我们可以调用getSessionId()方法来获得会话的ID。

那么Ephemeral类型的可排序znode不要命名为lock-sessionId-，所以当加上序号后就变成了lock-sessionId-sequenceNumber。那么序号虽然针对上一级名字是唯一的，但是上一级名字本身就是唯一的，所以这个方法既可以标记znode的创建者，也可以实现创建的顺序排序。

###不能恢复异常 Unrecoverable Exception

如果client的会话过期，那么他创建的ephemeral znode将被删除，client将立即失去锁（或者至少放弃获得锁的机会）。应用需要意识到他不再拥有锁，然后清理一切状态，重新创建一个锁对象，并尝试再次获得锁。注意，应用必须在得到通知的第一时间进行处理，因为应用不知道如何在znode被删除事后判断是否需要清理他的状态。

###实现 Implementation

考虑到所有的失败模式的处理的繁琐，所以实现一个正确的分布式锁是需要做很多细微的设计工作。好在ZooKeeper为我们提供了一个产品级质量保证的锁的实现，我们叫做WriteLock。我们可以轻松的在client中应用。
