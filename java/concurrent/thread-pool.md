## java线程池

线程池技术是程序设计中经常用到的基础技术，JDK为线程池提供了丰富的类和接口。

1. Executor: 只提供一个execute接口，接收一个Runnable参数
2. ExecutorService: 提供了更丰富的管理task的接口
3. ScheduledExecutorService: 提供了周期性执行task的接口
4. AbstractExecutorService: 实现了部分task的执行机制
5. ThreadPoolExecutor: 线程池Executor的核心实现
6. ScheduledThreadPoolExecutor: 可以周期执行任务的线程池Executor
7. Executors: jdk线程池的工具类

下面将以ThreadPoolExecutor为出发点，详解java线程池的用法及原理

### ThreadPoolExecutor原理及使用

ThreadPoolExecutor是java线程池家族中比较核心的类，类构造函数如下；

ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)

#### 参数含义

1. corePoolSize: 线程池核心线程数量，
2. maximumPoolSize: 线程池最大线程数量，如果有新的task到来时，
3. keepAliveTime: 线程池中非核心线程存活时间
4. unit: 存活时间的时间单位
5. workQueue: task队列
6. threadFactory: 新建线程的工厂，可以为线程创建有意义的名称，方便排查问题
7. handler: 线程池的饱和策略事件

#### 执行流程

当有新的task到来时

1. 如果curThreadSize < corePoolSize，则新建线程加入线程池，并执行当前task
2. 如果curThreadSize >= corePoolSize
	3. 如果workQueue未满，则加入到等待队列中
	4. 如果workQueue已满
		* 如果curThreadSize < maxPoolSize，则新建线程
		* 否则拒绝task

#### 线程池状态

* RUNNING: 在运行中.值为 -1 << (Integer.SIZE - 3)；结果为负
* SHUTDOWN: 被终止，值为0 << (Integer.SIZE - 3)；结果为0，shutdown后不接受新任务，但已经submit的任务将会执行完成
* STOP: 1 << COUNT_BITS。调用ShutdownNow()，不接受新任务，忽略队列中的任务，在执行的任务会尝试中断，如果未中断成功，则等待线程执行完成
* TIDYING: 2 << COUNT_BITS;所有任务都执行完（包含阻塞队列里面任务）当前线程池活动线程为0，将要调用terminated方法
* TERMINATED: 3 << COUNT_BITS;终止状态。terminated方法调用完成以后的状态

#### 线程池满策略

* AbortPolicy: 抛出异常，默认策略
* DiscardPolicy：丢弃新任务
* DiscardOldestPolicy：丢弃最老的任务，新任务继续提交 
* CallerRunsPolicy: 交给调用线程池所在的线程处理

#### task异常处理

如果一个线程执行的code出现了未处理异常，则该线程成为"Died Thread"（因此在使用ScheduleService时一定要注意使用try/catch，否则某次执行出现异常，则该task不可用）。线程池会创建新的线程代替"Died Thread"


#### 不同类别的ExecutorService

* Executors.newFixedThreadPool: corePoolSize和maxPoolSize相同,队列使用LinkedBlockingQueue，该线程池线程数量固定，队列长度无限	
* Executors.newWorkStealingPool: 见java中的(Fork/join框架)
* Executors.newSingleThreadExecutor: 相当于线程等于1的FixedThreadPool
* Executors.newCachedThreadPool: corePoolSize=0,maxPoolSize=Integer.MAX，每个线程存活60秒，队列使用SynchronousQueue，该线程池会根据需要创建足够多的线程，如果线程池中有可用线程则会复用线程池中的线程,适用于短时间内拥有大量段时间task的应用
* Executors.newSingleThreadScheduledExecutor: 创建只有一个能执行周期性任务的线程的线程池
* Executors.newScheduledThreadPool: 创建有多个可执行周期性任务的线程的线程池


### ThreadPoolExecutor核心源码

#### 提交task

```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}

```
可以看到提交task时，会生成一个RunnableFuture对象，用于跟踪或控制task的执行状况，submit的核心是execute()方法，可以看一下execute的实现

#### execute方法

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
}
```
如果当前线程数小于corePoolSize，则使用addWorker新建线程执行该task；否则，如果ThreadPool还在运行，则将新的command加入到workQueue队列中，注意这里有个recheck的过程，如果threadpool已经shutdown，则拒绝task，如果存在thread已经died，则通过addworker新加线程；如果添加队列失败，则尝试新加线程，如果失败则拒绝task

#### addWorker

添加新线程的函数addWorker

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
}
```
addWorker首先判断了ThreadPool的整个状态，然后通过newWorker新建Worker，新建成功，使用t.start()开始当前task，那么当当前task执行完成后，如何将线程给队列中的task使用呢？这就要看ThreadPoolExecutor的主循环方法runWorker了

#### runWorker()

```java

final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
}

```

当调用addWorker添加新的Worker时，Worker会通过ThreadFactory新建一个线程，线程的task就是Worker本身，执行的函数就是runWorker(this)，在runWorker中，线程会首先拿到Worker的firstTask执行，如果firstTask为空，则调用TreadPoolExecutor::getTask获取task执行。从这段可以看出，如果外层未捕获异常，afterExecution中又没有处理异常，会导致Thread Die


### 自实现线程池
