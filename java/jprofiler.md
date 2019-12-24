1: 查看cpu热点问题

CPU views: CPU消耗的分布及时间(cpu时间或者运行时间); 方法的执行图; 方法的执行统计(最大，最小，平均运行时间等)

 call tree中可以查看消耗cpu较多的方法，以及方法的调用详细堆栈

2: 线程问题跟踪

Thread: 当前jvm所有线程的运行状态，线程持有锁的状态，可dump线程。

可以实时查看线程状态，支持搜索，如果有大量红色状态的线程就证明线程阻塞情况

3: 内存泄漏问题

在 Live memory->Recorded Objects 中点击**record allocation data**按钮，开始统计一段时间内创建的对象信息。执行一次**Run GC**后看看当前对象信息的大小，并点击工具栏中**Mark Current**按钮(其实就是给当前对象数量打个标记。执行一次Run GC，然后再继续观察;执行一次Run GC，然后再继续观察...。最后看看哪些对象在不断GC后，数量还一直上涨的。最后你看到的信息可能和下图类似

在Heap walker中分析刚才记录的对象信息

Heap walker: 对一定时间内收集的内存对像信息进行静态分析，功能强大且使用。包含对象的outgoing reference, incoming reference, biggest object等



