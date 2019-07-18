## spring-cloud简介

### 主体结构

spring-cloud是实现在spring-boot之上的分布式系统框架，提供了一系列分布式系统的常用组件，和分布式系统的一揽子解决方案。

通过一些简单的注解，我们可以通过spring-cloud构建庞大的分布式系统

spring-cloud有庞大的组件库，如下图，重点关注Netflix下的子项目：



### 版本说明

spring-cloud项目是由多个子项目组成的，每个项目管理自己的版本号。主项目以伦敦的地铁站命名，每当一个大版本积累了很多update，或者修复了重大bug时，会再发布一版，成为SR(ServiceRelease)版，每个版本会有对应的兼容的spring-boot版本。

子项目的更新由子项目单独管理，每个子项目的版本有与之对应的兼容的主项目的SR版本，如图所示：


### 核心组件

spring目前有5大核心组件和一些分布式系统的特色组件:

####核心组件

* Netflix Euraka: 注册中心，服务的注册与发现

	Euraka是一个注册中心，同zookeeper的功能相似，service将服务信息注册到Euraka，client通过Euraka获取服务信息，然后缓存到本地，直接同service交互，会和service保持心跳，以更新租约和服务信息。
* Netflix Ribbon: 负载均衡客户端，实际是一个http的客户端
	
	一个客户测的负载均衡工具，可以将Rest风格的服务调用转换成客户端负载均衡的服务调用
* Netflix Hystrix: 断路器 

	防止故障扩散，避免“雪崩”效应的发生
	
	在微服务架构中，单个服务通常会集群部署，服务与服务之间相互依赖，如果某一个服务发生故障无法响应，将会阻塞它的上层服务，导致上层服务Servlet容器的线程耗尽，而上层服务的无响应又回导致依赖它的服务阻塞，最终形成“雪崩”。
* Netflix Zuul: 服务网关

	api网关，服务路由
	
	与nginx功能类似，在路由、分发场景下，客户端首先访问服务网关，服务网关根据api路由到具体服务，除此之外，还可以起到负载均衡的作用
	
	zuul和Ribbon的负载均衡：
	zuul的负载均衡更像是nginx的负载均衡，客户端或者app端访问服务时zuul做负载均衡。而ribbon的服务均衡更多是service之间调用的负载均衡，主要是微服务之间内部使用。  
* Spring cloud Feign

	封装了RestTemplate调用的客户端，可用于服务之间的相互调用。
	
	Feign与Ribbon的区别：
	Ribbon是一个Http和TCP的负载均衡器，从结构上来讲RestTemplate+Ribbon的功能类似于Feign。Feign是一个封装http请求的客户端，因为Spring cloud的服务都是以Http接口的形式存在的，所以使用Feign访问微服务的话，形式上相当于本地调用。Feign内部也使用了Ribbon做负载均衡
* Spring cloud config: 分布式配置

	SpringCloud Config提供服务器端和客户端。服务器存储后端的默认实现使用git，因此它轻松支持标签版本的配置环境，以及可以访问用于管理内容的各种工具。
	
	这个还是静态的，得配合Spring Cloud Bus实现动态的配置更新。

* Spring cloud Turbine: 

	集群监控工具

####特色组件
* spring actuator: 服务监控组件

	实际上该组件属于spring boot，可以用于分布式系统的可靠性监控，有一下功能：
	* 查看服务的健康状况
	* 获取环境信息
	* 获取线程的活动快照
	* 关闭应用
	* 提供http请求跟踪
	* 获取所有的url路径
	* 获取所有属性
	

	