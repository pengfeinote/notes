
上一章我们分析了如何通过netty构建一个简单的网络应用，包括服务器端和客户端的实现，其中涉及到了netty框架中多个组件的使用，包括Channel, EventLoop，pipeline等。那么这些组件是如何分工协作，完成一个网络请求的完整响应呢。本章我们将通过源码+断点的方式，来分析netty框架处理网络请求的完整流程。

在开始之前，我们先提出几个问题，将解决这几个问题做为分析netty处理流程的目标：

1. netty使用操作系统接口来监听网络请求，监听请求的代码在哪个类中？netty如何选择监听策略(select, poll, epoll)

2. netty应用中一般有两个线程池，bossGroup和workerGroup，bossGroup线程负责接收请求，workerGroup负责处理具体的IO，两个线程组分别做哪些具体工作，在哪里进行切换

3. netty从哪里开始调用ChannelPipeLine，在哪里终止调用，异常如何处理？

4. Channel与网络连接的关系，网络请求的数据如何读入到channel中，channel中写入的数据如何传到网络连接中，Channel和网络连接是一对一的关系吗？网络连接和channel是同时关闭吗？

