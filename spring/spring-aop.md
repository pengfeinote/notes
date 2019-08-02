动态代理：cglib jdk 反射


关于AspectJ表达式：
http://blog.51cto.com/5914679/2092253
Joint Point:连接点 指定的方法
PointCut: 切点

pointcut表达式：

	execution:使用比较多，主要用来匹配 访问修饰符；返回类型；包名；类名； 参数列表
	例：@PointCut("execution(public String com.tigerbrokers.stock.push.*(..))")
		public: 表示只匹配public的方法
		String: 表示只匹配返回类型为String的方法
		com.tigerbrokers.stock.push.*: 表示匹配包下的所有类
		..: 表示匹配任意参数列表
		如果括号里是(Long, ..)表示匹配第一个参数是Long的方法

	within:within可以用来匹配特定类，也可以匹配包下面所有类的所有方法
	例: @PointCut("within(com.tigerbrokers.stock.push..*)")  匹配该包下的所有类的所有方法
	    @PointCut("within(com.tigerbrokers.stock.OrderDao)") 只匹配OrderDao
		

	this和target都用来匹配特定类型的示例
	this:使用cglib代理实现切面，如果对象不是接口的话，使用this
	target:使用jdk动态代理来实现切面，如果对象是个接口或实现了接口的类的话，使用target

	@args:这个指示符是用来匹配连接点的参数的，@args指出连接点在运行时传过来的参数的类必须要有指定的注解，假设我们希望切入所有在运行时接受实@Entity注解的bean对象的方法：
	@Pointcut("@args(org.baeldung.aop.annotations.Entity)")
	public void methodsAcceptingEntities() {}
	为了在切面里接收并使用这个被@Entity的对象，我们需要提供一个参数给切面通知：JointPoint:
	@Before("methodsAcceptingEntities()")
	public void logMethodAcceptionEntityAnnotatedBean(JoinPoint jp) {
    		logger.info("Accepting beans with @Entity annotation: " + jp.getArgs()[0]);
	}

	@Target 匹配有目标注解的类的方法作为连接点
		@PointCut("@target(org.springframework.stereotype.Repository)")  匹配有Repository注解的连接点

	@within 这个指示器，指定匹配必须包括某个注解的的类里的所有连接点：
		

	@annotation  这个指示器匹配那些有指定注解的连接点，比如，我们可以新建一个这样的注解@Loggable:
		我们可以使用@Loggable注解标记哪些方法执行需要输出日志：
	

切点表达式组合：
	可以使用&&、||、!、三种运算符来组合切点表达式，表示与或非的关系。

	@Pointcut("@target(org.springframework.stereotype.Repository)")
	public void repositoryMethods() {}

	@Pointcut("execution(* *..create*(Long,..))")
	public void firstLongParamMethods() {}

	@Pointcut("repositoryMethods() && firstLongParamMethods()")
	public void entityCreationMethods() {}	

切点执行顺序
	@Around
	@Before
		func()
	@Around
	@After
	@AfterReturn or @AfterThrowing

	多个切面时：实现org.springframework.core.Ordered接口 或 使用Ordered注解实现，值越小的aspect越先执行
		Aspect1(5)  Aspect2(6)

	Aspect1.Around
	Aspect1.Before
		Aspect2.Around
		Aspect2.Before
			func()
		Aspect2.Around
		Aspect2.After
		Aspect2.AfterReturn or AfterThrowing

	Aspect1.Around
	Aspect1.After
	Aspect1.AfterReturn or AfterThrowing

AspectJ Jdk动态代理 与CGlib

静态代理：针对每个被代理类生成一个新类,编译器就生成
动态代理：运行时生成新的代理类
	jdk动态代理：
		InvocationHandler::invoke(Object proxy, Method method, Object[] args)，只支持实现了interface的类
		Proxy::newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
	cglib动态代理：使用asm开源库，修改字节码，生成被代理类的子类作为代理类

aspectj包：
	https://www.cnblogs.com/davidwang456/p/5633940.html


spring aop:
	调用方与被调用方不能在一个类中
	同一个类中，方法A调用方法B（方法B上加有注解），注解无效
  	针对所有的Spring AOP注解，Spring在扫描bean的时候如果发现有此类注解，那么会动态构造一个代理对象。
  	如果你想要通过类X的对象直接调用其中带注解的A方法，此注解是有效的。因为此时，Spring会判断你将要调用的方法上存在AOP注解，那么会使用类X的代理对象调用A方法。
  但是假设类X中的A方法会调用带注解的B方法，而你依然想要通过类X对象调用A方法，
  那么B方法上的注解是无效的。因为此时Spring判断你调用的A并无注解，所以使用的还是原对象而非代理对象。接下来A再调用B时，在原对象内B方法的注解当然无效了。

