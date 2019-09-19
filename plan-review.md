## 学习计划与复盘

* flink
* dubbo(多种rpc框架的使用、实现总结spring-cloud, grpc, thrift, finagle)
* es
* 一种MQ
* spring(boot)技术栈 jdk经典结构(源码) guava经典结构(源码) 

### 流水账

#### 2019-09-16 ~ 2019-09-22

1. 计划
	* bloom过滤器
	* 手写单链表有环求节点；手写堆排序；实现正则匹配
	* dubbo继续
2. 复盘
	* 2019-09-16: Bloom过滤器实现
	* 2019-09-17: 正则匹配(星号,问号替换)
	* 2019-09-18: hashmap扩容,concurrentHashMap扩容
3. 总结
	* 布隆过滤器：概率版、廉价版的hashset
	* redis持久化：RDB(定时flush全库数据), AOF(追加写命令)两种,AOF三种模式：always(每个写flush一次), persecond（每秒flush一次）, no（根据操作系统）
	* dp，最优子结构：非星号：dp[i][j] = dp[i - 1][j - 1] && (s[i] == p[j]);星号：dp[i][j] = dp[i][j - 1](星号匹配0字符) | dp[i-1][j](星号匹配n个字符)
	* hashmap的treeify/untreeify/扩容过程
	* concurrenthashmap的扩容与hashmap比较像，不过是并行完成的，先申请新的table,然后遍历旧table，通过为节点添加fwd标志，完成多线程扩容的操作

#### 2019-09-09 ~ 2019-09-15

1. 计划
	* 一致性hash与跳表实现,链表从尾到头每隔k翻转一下
	* dubbo解析
	* 刷code 3道m，一道hard(逆序数个数，链表归并排序, 隔k反转)
	* flink
2. 复盘
	* 2019-09-09: dubbo，重点关注dubbo分组，缓存，异步，多注册中心多版本，回调等的配置 
	* 2019-09-15: 手写单链表归并排序
3. 总结
	* 每隔k翻转：先求长度，然后计算出前几个不反转，随后开始反转，注意记录各种前置节点
	* dubbo服务降级，延迟暴露，优雅停机 


#### 2019-09-02 ~ 2019-09-08

1. 计划
	* java线程池
	* 刷code
	* dubbo
2. 复盘
	* 2019-09-04: ThreadPool状态列表，核心代码，执行流程
	* 2019-09-05: permutation kmp lcs
3. 总结
	* Worker作为ThreadPoolExecutor中的核心元素，通过getTask从Queue中获取task并执行
	* dubbo 线程模型配置，错误处理配置

#### 2019-08-26 ~ 2019-09-01

1. 计划
	* java线程池
	* 3个mid，一个hard，一个CV
	* Flink
2. 复盘
	* 2019-08-26: java线程池
3. 总结
	* ThreadPoolExecutor

#### 2019-08-19 ~ 2019-08-25

1. 计划
	* 实现两个有序数组求中位数
	* flink文档阅读
	* jdk同步相关(锁系列, Executor系列)
2. 复盘
	* 2019-08-19 有序数组求中位数
	* 2019-08-20 AQS中Condition队列
	* 2019-08-21 java内存模型(https://mp.weixin.qq.com/s/C7GoTSr0axr1skPn44vtqg)
3. 总结
	* 两个有序数组求中位数，重点在中位数左右的元素数相等，先将两个数组分成两个元素数相等的两部分，然后调整指针，如果一部分的最大值小于另一部分的最小值，则求中位数成功
	* ReentrantLock与Synchronized的实现区别：前者是通过AQS（volatile+CAS）实现，后者是通过java 对象头的monitor锁实现的
	* AQS中有一个同步队列和多个(>=0)Condition队列，Condition队列用于获取锁之后等待资源
	* java内存模型：线程模型，指令重排序，顺序一致性，volatile, 锁，final
	* volatile实现，内存屏障(LoadLoad, LoadStore, StoreLoad, StoreStore),happens-before,synchronized使用monitor实现（对象头加threadId）
	* synchronized最初偏向锁，有线程竞争时自旋变轻量级，自旋过长时升级为重量级

#### 2019-08-12 ~ 2019-08-18

1. 计划
	* 阅读flink文档
	* 回顾redis相关内容
2. 复盘
	* 2019-08-12: redis(pipeline scan) flink DataStream Api
	* 2019-08-13: flink time attr
	* 2019-08-14: git rebase 和 cherry-pick
	* 2019-08-17: AQS实现 
3. 总结
	* redis pipeline 减少RTT，减少内核态调用
	* scan使用游标的方式，减少命令的阻塞时间，效果优于keys, getall, range等
	* flink DataStream api: Iterations，通过env设置job的参数，例如水印的生产方式，控制latency(通过setBufferTimeout)
	* rebase(合并commit记录，合并分支)，cherry-pick(挑选单独commit进行合并)
	* flink可以处理late event，并行task的水印处理机制
	* AQS通过(双向队列+Node状态+acquire/release/acquireShared/releaseShared四个函数)实现了java中各种同步类的顶层抽象，同步类只需要实现try类接口即可

#### 2019-08-05 ~ 2019-08-11

1. 计划
	* spring-cloud总结
	* flink
2. 复盘
	* 2019-08-05: ribbon的几种负载均衡方式及使用方法
	* 2019-08-06: ribbon+eureka+feign的使用
	* 2019-08-07: 空
	* 2019-08-08: flink api的基本概念
	* 2019-08-09: flink java8 lambda
3. 总结
	* ribbon自定义的几种负载均衡策略
	* 使用feign定义负载均衡策略
	* flink支持的数据类型（重点是POJO与泛型），field expression获取key，获取执行环境的方法
	* 在涉及泛型时，通过returns(Types type)来告诉flink泛型的类型


#### 2019-07-29 ~ 2019-08-04

1. 计划
	* spring-cloud部署完成
	* spring-cloud组件总结
	* flink api部分文档阅读完毕
	* maven package总结
	* 刷题15道以上
	* 2019-08-03&2018-08-04: 刷mid题5-8道，构建完整的spring-cloud微服务(<font color='red'>未实现</font>)

2. 复盘
	* 2019-07-29: spring-cloud service provider部署,hystrix使用 刷一题
	* 2019-07-30: 总结hystrix的设计特点和用法 刷一题
	* 2019-07-31: 整理maven打包相关插件
	* 2019-08-01&2019-08-02：整理feign和hystrix结合的使用方式

3. 总结
	* spring-aop坑 同一类内的函数调用不适用spring aop(代理bean的问题)
	* 断路器在设计上主要考虑“快速失败/及时降级回退”的效果,以避免微服务架构下的雪崩效应,在此之上，hystrix还实现了请求隔离，请求合并的功能
	* 请求隔离可以通过不同线程/线程池和信号量实现，对于不同的服务使用不同的线程池，同一个服务的不同请求，使用不同的线程，线程用满时拒绝服务，是Hystrix的重要设计
	* Hystrix使用Command模式封装请求，可以同步或者异步的执行请求，可以添加观察者
	* maven-assembly-plugin maven-shade-plugin spring-boot-maven-plugin
