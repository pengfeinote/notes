## zuul

### 概述

zuul是spring-cloud中netflix包里的一个api网关，它接受外部用户app端/网页端的请求，将请求分发至对应的service服务。

zuul在架构上与nginx相似，具有路由，分发，负载均衡，熔断等功能；同时，zuul中可以提供自定义的过滤器。zuul本质上是一个web servlet应用。

#### zuul与nginx

nginx在架构上可以和zuul结合使用，nginx提供域名服务，zuul提供路由，分发服务。从功能上讲，zuul的功能更强大，实现同样的功能，nginx可能需要各种脚本支持。

### 工作模型

#### 请求流

zuul的核心是filter，工作模型按照用户请求的流向分为以下几个模块：

1. zuul server  
    
	本质上是一个netty server Handlers，接受并分析用户的请求
2. zuul inbound filters

	一系列的pre过滤器，在请求到达真实的处理服务之前执行，可能是一些权限校验，也可能是一些日志记录等等
3. zuul origin client

	本质上也是一个filter(Endpoint Filter)，在这个filter里，zuul作为一个netty client，向请求的真实处理服务发出请求
4. zuul outbound filters
	在从真实服务获得响应之后执行的filters，可以用来做性能统计，添加通用的响应头的工作
	
#### filters

* filter类型
	1. incoming
	2. endpoint
	3. outcoming	

* 同步及异步filter
	根据filter本身的特点，如果操作比较耗时，或者还需要依赖其他应用，可以使用异步filter，不影响请求本身的运转

* 执行顺序

	跟type一起使用，定义了filter的执行顺序
	
* 获取请求体

	可以通过重载函数needsBodyBuffered来获取、使用请求体	

### 支持特性

* 服务发现
	
	可以借助eureka，也可以自定义服务列表
* 负载均衡

	默认使用ribbon的负载均衡器（轮询），也可以自定义负载均衡的类
* 自定义连接池

	zuul使用自定义的连接池技术，连接池配置可自定义
* 重试机制

	zuul自身支持超时重试机制，如果请求超时，或者service报503(Service Unavailable)，会进行重试，重试机制可配置
* 请求追踪

	一个纳秒级的请求追踪，记录功能，debug神器
* 并发控制
	可以设置service的最大并发数，超过并发数将报服务不可用
	
* Gzip压缩

	支持Gzip压缩

### 使用

#### 项目构建
	
zuul位于spring-cloud的netflix包中，使用很方便：

maven配置：

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
        <!-- Spring Boot Begin -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

```java
@EnableZuulProxy
@SpringBootApplication
public class ZuulServer {

  public static void main(String[] args) {
    SpringApplication.run(ZuulServer.class, args);
  }
}
```

EnableZuulProxy与EnableZuulServer的区别：
Proxy注解会天然注册一些用于反向代理的filter进去，而Server注解不会自动引入额外注解。proxy可以认为是server的增强版，当需要zuul搭配eureka和ribbon一起使用时，应该使用Proxy注解

#### 路由

1. 路由方式

可以使用eureka或者不使用eureka
	
使用eureka时，路由规则配置：
	
```yml
	zuul:
  		routes:
    		traditional-url:                             #传统的路由配置,此名称可以自定义
		      path: /tr-url/**                           #映射的url
		      url: http://localhost:9001/ 
```
	
不使用euraka时，路由规则配置:

```yml
	zuul:
  		routes:
			orient-service-url:                          #面向服务的路由配置,此名称可以自定义
		      path: /os-url/**
		      service-id: feign-customer   
```
	
	

2. 自定义路由规则

	PatternServiceRouteMapper对象可以让开发者通过正则表达式来自定义服务与路由映射的生成关系

### 原理及源码