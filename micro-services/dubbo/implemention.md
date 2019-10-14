## dubbo关键实现

### 负载均衡的实现

dubbo的负载均衡全部基于AbstractLoadBalance的子类来实现，AbstractLoadBalance提供了选择invoker的虚方法，实现类可以通过重写该方法实现具体的负载均衡策略，在dubbo中，所有invoker都有权重的概念，负载均衡方法都会基于一定的权重实现.

dubbo提供了权重的计算函数，计算中会有一个系统预热机制(与jvm预热类似默认为10分钟),如果系统预热时间不够，则会进行降权处理

```java
    protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);
```

#### 负载均衡策略

* 随机选择invoker

	如果invokers权重相同，则随机选择index，否则根据权重选择invoker。如何根据权重随机选择，使权重大的被选中的概率更高呢？

* 轮询选取invoker

	轮询时，如果某个invoker响应较慢，会导致请求堆积。如果权重都相同，则简单的index+1，否则根据一定算法进行轮询，权重更大的轮询到的几率更高。

* 最小活跃invoker
	
	活跃数指的是一个provider中活跃的请求数，每当收到一个新请求，active加一，请求结束后active减一，一个provider拥有最少的active，说明它的请求处理效率最高(?)，也表明负载最低

	dubbo中的最小活跃指的是带权重的最小活跃，如果几个provider的最小活跃相同，则会根据权重随机选取一个provider

* 一致性hash invoker
	
	带虚拟节点的一致性hash实现，每个invoker默认有160个虚拟节点，采用TreeMap的方式存储一致性hash环，使用TreeMap.tailMap(fromKey)的方式寻找调用的invoker，dubbo使用md5的方式辅助计算hashCode，key为调用签名（即serviceKey+methodName），如果invokers改变，则会重新构造一致性hash环
	dubbo对一致性hash的源码实现比较经典，建议阅读

#### 权重规则

可以手动设置权重，dubbo会根据预热时间来重新计算权重


### dubbo路由

dubbo通过Router和RouterFactory两个接口来实现路由，dubbo本身有条件路由/标签路由/脚本路由三种方式，条件路由是最常用的路由方式，用户也可以自己实现两个接口来自定义路由规则，

```java

@SPI
public interface RouterFactory {

    /**
     * Create router.
     *
     * @param url
     * @return router
     */
    @Adaptive("protocol")
    Router getRouter(URL url);

}

public interface Router extends Comparable<Router> {

    /**
     * get the router url.
     *
     * @return url
     */
    URL getUrl();

    /**
     * route.
     *
     * @param invokers
     * @param url        refer url
     * @param invocation
     * @return routed invokers
     * @throws RpcException
     */
    <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;

}
```

