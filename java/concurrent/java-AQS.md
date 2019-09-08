## java AQS

AQS(AbstractQueuedSynchronizer)是jdk concurrent包下的基础类，它为jdk同步包中的各种基础组件（如Lock, Latch, Semaphore, ThreadPool）等提供了同步支持。如名称所示，AQS为线程对资源竞争使用提供了一种队列及同步机制，简单而言，各线程按照对资源的竞争顺序进入队列，先进入队列的线程将优先使用资源，使用完毕后释放资源，唤醒后面的线程。AQS本质上是对资源竞争使用的抽象实现。

### AQS设计

AQS在实现上采用了双向队列的方式，队列中一个节点为一个等待线程，节点分为独占节点和共享节点两种，每个节点可以有一下状态：

1. CANCELLED(1): 节点由于超时或中断被取消，是节点的最终状态，节点一旦被取消，将不再会被阻塞
2. SIGNAL(-1): 节点的后继节点将被阻塞，因此当前节点释放或者被取消时需要unpark后继节点，
3. CONDITION(-2): 节点在条件队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()方法后，该节点将会从条件队列中转移到同步队列中，加入到对同步状态的获取中
4. PROPAGATE(-3): releaseShared()需要传播下去，在共享模式下保证release传播

AQS中的核心方法：

```java

//获取资源的顶层方法，获取失败则阻塞等待
public final void acquire(int arg);

//释放资源的顶层方法，释放成功则唤醒下一节点
public final boolean release(int arg);

//尝试获取资源，需子类实现
protected boolean tryAcquire(int arg);

//尝试释放资源，需子类实现
protected boolean tryRelease(int arg);

```

以上是独占资源的主要接口，与之对应还有一套Shared接口，为共享资源的主要接口。

try类接口需要子类实现，接口的调用线程会是顶层接口的调用线程，如果tryAcquire()返回失败，则线程将被加入队列并阻塞，直到前驱节点的线程唤醒


<small><b>注1：AQS并没有将try类接口设置为abstract，而是在函数中抛出了Unsupported异常，因为子类不需要同时实现Exclusive和Shared接口，日常工作中可以借鉴该做法</b></small>

<small><b>注2：AQS中的线程同步完全采用了volatile+CAS的方式</b></small>

### AQS源码

#### Node结构

```java
	/** Marker to indicate a node is waiting in shared mode */
    static final Node SHARED = new Node();

	/** Marker to indicate a node is waiting in exclusive mode */
    static final Node EXCLUSIVE = null;

	volatile int waitStatus;

	volatile Node prev;

	volatile Node next;

	volatile Thread thread;

	Node nextWaiter;
```

注意volatile的使用，因为多数字段需要对其他线程可见。注意:

1. nextWaiter是指条件队列的下一个节点
2. thread只有在构造node节点时赋值，在使用之后赋值为null

#### 成员变量

```java

private transient volatile Node head;

private transient volatile Node tail;

private volatile int state;

```

注意所有变量均为volatile，state可以看做同步的辅助状态。例如在Sync的实现中，state代表了当前线程的对资源的acquire次数，
在Semaphore中则代表资源数量

#### acquire（获取资源）函数

```java

	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    
```

可以看出，先尝试获取资源，如果获取失败，则将节点加入队尾(addWaiter)，并调用park等待(acquireQueued)，实现情况如下:
    
```java    
	private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
    
	private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
     		  }
        }
    }    

	final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

addWaiter将新节点加入队尾，如果队列是空的或者有竞争线程，则进入enq函数，继续入队。

入队成功之后，调用函数acquireQueued，如果当前节点是头节点且获取资源成功，则设置头节点，重置同节点状态，否则
调用shouldParkAfterFailedAcquire和parkAndCheckInterrupt将线程block。

<small><b>
注意enq函数的实现，通过volatile+循环+cas实现了线程安全的入队操作
</b></small>
    
shouldParkAfterFailedAcquire和parkAndCheckInterrupt的实现如下：

```java    
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

```

shouldParkAfterFailedAcquire方法通过对当前节点的前一个节点的状态进行判断，对当前节点做出不同的操作

parkAndCheckInterrupt让线程去休息，真正进入等待状态。park()会让当前线程进入waiting状态。在此状态下，有两种途径可以唤醒该线程：1）被unpark()；2）被interrupt()。需要注意的是，Thread.interrupted()会清除当前线程的中断标记位。

acquireQueued的流程可以总结为：

1. 结点进入队尾后，检查状态，找到安全休息点；
2. 调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
3. 被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。

acquire的流程可以概括为：

1. 调用tryAcquire尝试获取资源
2. 没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

<small><b>注意acquire在执行过程中不响应中断，只有在线程获取到资源时才将中断补上，如果需要响应中断，可以使用acquireInterruptibly</b></small>

#### release

release方式是释放独占资源的顶层入口，如果线程释放资源成功，会唤醒后续线程来获取资源

```java

	public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

```

如代码所示，使用tryRelease释放资源，如果释放成功使用unparkSuccessor来唤醒后续线程

```java

	private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }

```

在unparkSuccessor中，首先重置了头节点的状态(state = 0)，然后唤醒后继节点中第一个不为null切没有取消的后继节点

<small><b>注意代码中是从tail向前遍历而非从head向后遍历，是防止同步问题导致队列卡住</b></small>

#### acqureShared

acquireShared用于获取共享资源，注意获取资源失败的判断方法:tryAcquireShared(arg) < 0，表示没有足够的资源供线程使用

```

 	public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

	private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // doAcquireShared help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

```

与acquire不同的地方在于，head获取资源之后，会执行函数setHeadAndPropagate，将剩余的资源用于释放队列中剩下的线程

```java

private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

```


#### releaseShared

```java

	public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
    
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }

```

注意这里release传递（progate）的方式，线程A释放资源之后，调用doReleaseShared唤醒A的后继节点线程B，线程B执行doAcquireShared获取到
资源，如果B获取资源之后资源数大于0(if r > 0)，则调用setHeadAndPropagate，setHeadAndPropagate中又使用了
doReleaseShared唤醒后继节点C

### AQS中的ConditionObject

AQS中实现了两种队列，同步队列和COndition队列，一个AQS只有一个同步队列，表示获取资源的队列，一个AQS可以有多个Condition队列，可以理解为等待某种资源而挂起的队列。一个节点必须先在同步队列中，调用ConditionObject的await接口，从同步队列移动到等待队列，所以在使用锁的Condition时，调用Condition的接口前线程必须先获取锁，然后调用await接口，释放锁(从同步队列移除)，等待其他线程调用signal(移入到Condition队列中)，当有其他线程调用signal接口时，将节点从COndition队列移除并加入到同步队列中

```java

	public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

```
注意，因为await之前，当前线程已经获取锁，它在同步队列里的node已经被移除，所以这里添加了新节点

```java
	private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

	final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }

```

signal操作时，将节点从condition队列转移到同步队列

注意，ConditionObject中没有同步操作，因为调用接口的时候，线程已经获取到锁了，所以不需要同步

https://www.jianshu.com/p/a82228c8c0d5

### AQS在jdk中的典型应用

#### ReentrantLock中的FairSync与UnfairSync

```java

	//nonfair Sync
	final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
   
    //fair sync
    protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
        
    final void lock() {
            acquire(1);
        }

```

注意fair与unfair的区别：
FairSync在尝试获取资源时，会先查看AQS中是否有前驱节点，如果有，则直接返回失败，NonfairSync则直接尝试获取资源，
获取失败时返回失败。联想到AQS acquire的实现，非公平锁在资源可用时，直接尝试获取资源，竞争失败则入队，等待队列
中前驱节点唤醒

总结一下ReentrantLock中lock的过程：

1. 使用sync的lock,调用了AQS中的acquire
2. AQS通过acquire调用sync的tryAcquire
3. 如果是公平锁，则入队等待调度，如果是非公平锁则直接尝试获取资源，失败后等待调度

#### Semaphore中共享模式的使用

```java

	final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
        
    protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

```


此时AQS的state代表信号量最大允许数量，remaining<0时直接返回，代表try失败，否则重新设置state状态，获取成功

通过这两个例子可以看到，同步的主要过程都放在了AQS中处理，AQS极大的减少了JDK中同步工具实现的复杂度。
