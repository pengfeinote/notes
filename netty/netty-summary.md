## netty概述

### 简介

netty是一个广泛使用的java网络编程框架，它简化了和流线化了网络应用的开发，使开发者可以快速的构建自己的网络程序。许多知名的网络程序都是基于netty开发的，如dubbo、grpc等

netty在原有的java网络程序上提供一下功能：

* netty nio ： netty的nio对jdk的nio做了进一步的封装(接口、类的封装)和优化(epoll的空轮询bug)
* channel pipelin: Channel, ChannelPipeline, ChannelHandlerContext, ChannelHandler类所组成的处理链
* reactor: 网络io的线程模型，分离连接建立，消息处理及网络通信
* ByteBuf: 同NIO类似，对jdk ByteBuffer的重新实现和优化



//todo

### netty功能设计

![overview](netty.png)

如上图所示，netty框架主要由Transport Service(传输模块)、Protocol Support(协议模块)和netty核心库三部分组成。


**transport-service**

传输模块对应channels包下面的channel相关类，提供了组合channel和读写channel数据、注册IO线程，引用pipeline等功能。channel本质上是一个逻辑连接，将连接、线程与pipeline组合起来。

**netty-core**

netty核心库中提供了一套支持零拷贝的高性能Byte Buffer，提供数据的缓存和传输机制；除此之外，针对不同的传输类型，netty提供了统一的API接口；事件机制是handler之间，handler与channel之间的交互机制

**protocol-support**

协议本质上是一堆handler的组合，通过handler的有序组合处理请求来实现具体的协议。netty中内置了多种常见的网络协议，如Http、WebSocket和Protobuf等

### 源码结构(以4.1为例)

**common**

common包主要包含一些常用的工具类，还有netty自己的一套并发包，里面有netty自定义的future、promise等。common包是netty的基础包，其他包都依赖common包

**buffer**

buffer包里包含了netty自实现的ByteBuffer，对JDK的ByteBuffer进行了重新实现和优化。netty中的ByteBuffer不仅可以申请对内存，还可以申请堆外内存，另外，netty实现了内存池对ByteBuf进行管理

**codec**

codec系列包中实现了各种经典的网络协议，包括协议数据结构的定义以及相关的Handler等

**handler**

handler包中包含了一些网络编程中通用的Handler，例如ip、log及timeout等

**transport**

transport提供了包括Bootstrap, Channel, Eventloop一系列核心类的实现

* Bootstrap：应用启动的辅助类
* Channel：一个逻辑连接，通过ChannelPipeine使用ChannelHandler对请求进行流水线形式的io处理
* Eventloop：一个eventloop代表一个线程，每个请求会被分配到一个eventloop中进行io操作

**resolver**

reolver包主要对address、hostname等进行解析处理
