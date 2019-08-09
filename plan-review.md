## 学习计划与复盘

* flink
* dubbo(多种rpc框架的使用、实现总结spring-cloud, grpc, thrift, finagle)
* es
* 一种MQ
* spring(boot)技术栈 jdk经典结构(源码) guava经典结构(源码) 

### 流水账

#### 2019-08-05 ~ 2019-08-11

1. 计划
	* spring-cloud总结
	* flink
2. 复盘
	* 2019-08-05: ribbon的几种负载均衡方式及使用方法
	* 2019-08-06: ribbon+eureka+feign的使用
	* 2019-08-07: 空
	* 2019-08-08: flink api的基本概念
3. 总结
	* ribbon自定义的几种负载均衡策略
	* 使用feign定义负载均衡策略
	* flink支持的数据类型（重点是POJO与泛型），field expression获取key，获取执行环境的方法


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
