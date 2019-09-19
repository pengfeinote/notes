## dubbo特性

dubbo分为5个角色：

1. provider: 服务提供者
2. consumer: 服务消费方
3. registry: 注册中心
4. monitor：监控中心
5. container：provider容器

关注dubbo缓存，多注册中心，异步，mock机制等用法

### 功能

#### 配置方式

1. xml
2. 注解
3. 程序加载等

#### 配置项

1. check = false
2. 失败策略：retry, failfast, failsafe(忽略失败),Failback(定时重发),forking(并行调用，有成功则返回，浪费资源对实时性较高), Broadcase(广播，任一报错则错误) 
3. 负载均衡： 随机，轮训，最少访问，一致性哈希
4. 线程模型:  dispatch指定是否在IO线程中处理事件，threads执行线程池类型
5. 可以在配置中配置多个注册中心
6. 同一接口的不同实现可以使用group分组，可以使用version做版本控制
7. dubbo提供merger机制，将不同组的结果merge到一起
8. 结果缓存: 使用cache配置consumer可以缓存结果，缓存策略有lru, threadlocal或者自实现cache
9. consumer异步：consumer异步与provider异步，dubbo的异步以CompletableFuture为基础，基于 NIO 的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小。Consumer异步的实现方式：
	* 需要服务提供者事先定义CompletableFuture签名的服务
	* 配置async=true之后，使用RpcContext手动调用
	* 消费方使用default重载服务签名
10. provider异步：provider异步与consumer异步并不冲突，使用CompletableFuture.supplyAsync提供异步服务
10. mock机制: 提供服务降级或是测试方式 
11. callback机制: onreturn, oninvoke, onthrow三种方式的callback
12. 延迟暴露: 等待spring容器初始化完成再暴露服务，过早暴露服务可能低版本dubbo造成死锁
13. 延迟连接：dubbo服务本身需要一定时间初始化，可以延迟连接
14. 服务降级: 本质上还是使用mock机制服务降级
15. 优雅停机: 使用ShutdownHook优雅停机，可以设置停机超时时间。停机时，provider/consumer不接受新请求，原请求将执行完成

#### 服务化实践

服务参数及返回值建议使用 POJO 对象，即通过 setter, getter 方法表示属性的对象。服务参数及返回值不建议使用接口，因为数据模型抽象的意义不大，并且序列化需要接口实现类的元信息，并不能起到隐藏实现的意图。服务参数及返回值都必须是传值调用，而不能是传引用调用，消费方和提供方的参数或返回值引用并不是同一个，只是值相同，Dubbo 不支持引用远程对象。

provider端配置属性
1. 建议配置客户端参数
	* timeout: 方法调用超时时间
	* retries: 失败重试次数
	* loadbalance: 负载均衡算法
	* actives: Consumer端的最大并发限制
2. 服务端参数
	* threads: 服务线程池大小
	* executes: 一个服务提供者并行执行请求上限,即当 Provider 对一个服务的并发调用达到上限后，新调用会阻塞，此时 Consumer 可能会超时。在方法上配置 dubbo:method 则针对该方法进行并发限制，在接口上配置 dubbo:service，则针对该服务进行并发限制

### 架构

### 源码设计
