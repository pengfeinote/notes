## spring容器初始化流程

https://blog.csdn.net/liuyi6/article/details/86549309


### 创建BeanFactory

### 加载Bean定义

### 创建Bean


### 可插拔组件


#### BeanDefinitionRegistryPostProcessor

是一种特殊的BeanFactoryPostProcessor，方法postProcessBeanDefinitionRegistry可以自定义注册Bean定义的逻辑

#### BeanFactoryPostProcessor

BeanFactoryPostProcessor是在spring容器加载了bean的定义文件之后，在bean实例化之前执行的。接口方法的入参是ConfigurrableListableBeanFactory，使用该参
数，可以获取到相关bean的定义信息
实现该接口，可以在spring的bean创建之前，修改bean的定义属性。也就是说，Spring允许BeanFactoryPostProcessor在容器实例化任何其它bean之前读取配置元数据
，并可以根据需要进行修改，例如可以把bean的scope从singleton改为prototype，也可以把property的值给修改掉。可以同时配置多个BeanFactoryPostProcessor，>并通过设置'order'属性来控制各个BeanFactoryPostProcessor的执行次序。
A BeanFactoryPostProcessor may interact with and modify bean
 * definitions, but never bean instances.

 /**
         * Modify the application context's internal bean factory after its standard
         * initialization. All bean definitions will have been loaded, but no beans
         * will have been instantiated yet. This allows for overriding or adding
         * properties even to eager-initializing beans.
         * @param beanFactory the bean factory used by the application context
         * @throws org.springframework.beans.BeansException in case of errors
         */
        void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;


#### BeanPostProcessor

如果我们需要在Spring容器完成Bean的实例化、配置和其他的初始化前后添加一些自己的逻辑处理，我
们就可以定义一个或者多个BeanPostProcessor接口的实现，然后注册到容器中。
        Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
        实例化、依赖注入完毕，在调用显示的初始化之前完成一些定制的初始化任务

        Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
        实例化、依赖注入、初始化完毕时执行

 
