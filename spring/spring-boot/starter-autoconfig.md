## spring-boot-starter和spring-boot的autoconfiguration

### @EnableAutoConfiguration

该注解使用Import引入了EnableAutoConfigurationImportSelector.class，该类会读取jar包中的META-INF/spring.factories文件，文件中定义了需要引入的自动配置类

### spring-boot-starter实现

以spring-boot-starter-actuator为例，本质上是一个jar包(spring-boot-starter-actuactor.jar)，jar包中包含了一个META-INF的文件夹，里面的spring.factories文件包含了需要引入的模块，或者在maven文件夹的pom.xml中引入指定的依赖

### spring-boot-autoconfiguration实现

在autoconfiguration的jar包中，也包含spring.factories文件，文件中指定了要扫描的类，以Ribbon为例：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration
```

RibbonAutoConfiguration实际上是spring的一个configuration，里面包含了使用ribbon所需要的bean，这样就不需要用户自己定义ribbon相关的bean了，用户只要在配置文件中做相关配置即可。

如果想要加入开关，可以在Configuration中加入@ConditionalOnClass(EnableRibbonConfig.class)，这样用户只有加入了EnableRibbonConfig才会启用自动配置，

Conditional相关的注解还有@ConditionalOnProperty, @ConditionalOnBean, @ConditionalOnMissing等等

使用AutoConfigureBefore和AutoConfigureAfter可以控制autoConfig的顺序
