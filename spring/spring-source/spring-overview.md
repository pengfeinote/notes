## spring架构设计

spring是java领域中广泛应用的一个轻量级框架，为开发者提供了控制反转和面向切面变成的能力，在此之上，提供了一系列的功能，诸如存储、消息、web等。

### 总体框架

spring的总体框架如图所示：

![overview](spring-overview.png)

spring框架包含大概20个子模块，20个子模块大致可以分成以下几组：测试（Test），核心容器（Core Container），数据部分（DAta Access/Integration），web，面向切面编程（AOP），消息（Messaging）等。

#### Core Container

spring核心容器包括spring-beans, spring-core, spring-context, spring-context-support, spring-expression这几个模块。

spring-core和spring-beans为整个spring框架提供基础支撑，包括IOC(控制反转)和DI(依赖注入: Dependency Injection)功能。其中BeanFactory是工厂模式的一种典型实现，BeanFactory能够消除程序中通过编程实现的单例，并能够对程序逻辑和依赖关系进行解耦。

#### AOP

spring-aop提供了面向切面编程的程序实现，使用aop，你可以对一些通用性的功能和模块本身的功能进行解耦。

spring-aspects则继承了AspectJ。

spring-instrument提供了类植入的支持和类加载器的实现，可以在特定的应用服务器中使用。


#### web

web模块包含了spring-web，spring-webmvc，spring-websocket，spring-webmvc-portlet等模块。

spring-web提供了基于web的一套基本功能，包括文件上传、使用Servlet监听器初始化IOC容器、一个基于web的应用上下文及Http Client等功能。

spring-webmvc提供了spring model-view-controller和rest服务的具体实现，并拆分了模型代码、表单和其他web功能的具体职责。

#### Data Access/Integration

spring数据部分提供了jdbc、ORM及JMS等功能的抽象。

spring-jdbc提供了jdbc的抽象层，屏蔽了不同数据库厂商的差异。

### 模块功能

ArtifactId | Description
:-: | :-: | :-: | :-: | :-:
spring-aop | 基于代理的aop支持
spring-aspects | 基于AspectJ的切面
spring-beans | 对Beans的支持
spring-context | 应用上下文的运行时支持
spring-context-support | 支持第三方库集成到spring上下文
spring-core | 核心工具类包，支撑其他spring模块
spring-expression | spring expression language
spring-jdbc | jdbc抽象包，包括对数据源和jdbc访问的支持
spring-web | 基础的web支持模块
spring-webmvc | 基于http的mvc和rest支持
spring-websocket | 基于spring的websocket框架



