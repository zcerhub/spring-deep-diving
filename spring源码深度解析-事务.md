# 事务

## spring事务使用示例

```
@Data
@AllArgsConstructor
@ToString
public class User {

    private int id;
    private String name;
    private int age;
    private String sex;


    public User() {

    }
}
```

```
public class UserRowMapper implements RowMapper {

    public Object mapRow(ResultSet rs, int rowNum) throws SQLException {
        User person=new User(rs.getInt("id"),
                rs.getString("name"),
                rs.getInt("age"),
                rs.getString("sex"));
        return person;
    }

}
```

```
@Transactional(propagation= Propagation.REQUIRED)
public interface UserService {

    void save(User user) throws MyException;

}
```

```
public class UserServiceImpl implements UserService {

    private JdbcTemplate JdbcTemplate;

    public void setDataSource(DataSource dataSource) {
        this.JdbcTemplate=new JdbcTemplate(dataSource);
    }

    public void save(User user) throws MyException {
        JdbcTemplate.update("insert into user(name,age,sex) values(?,?,?)",
                new Object[]{user.getName(), user.getAge(), user.getSex()}, new int[]{
                        Types.VARCHAR,
                        Types.INTEGER,
                        Types.VARCHAR,
                });
        throw new RuntimeException("aa");
//        throw new MyException("aa");
    }

}
```

```
public class MyException extends Exception {

    public MyException(String aa) {
        super(aa);
    }
}
```

```
	<!-- 添加事务注解支持 -->
	<tx:annotation-driven transaction-manager="txManager" />

	<!--配置事务 -->
	<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSource"/>
	</bean>

	<bean id="dataSource"
		  class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
		<property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
		<property name="url" value="jdbc:mysql://192.168.152.5:3306/biyi_test?allowPublicKeyRetrieval=true"/>
		<property name="username" value="root"/>
		<property name="password" value="v3imYJ2@yL6Aq6Tu"/>
		<property name="initialSize" value="1" />
	</bean>

	<bean id="userService" class="com.zchuber.springsourcedeepdiving.tx.UserServiceImpl" >
		<property name="dataSource" ref="dataSource"/>
	</bean>
```

测试用例：

```
    @Test
    public void testSimpleLoad() throws MyException {
        ApplicationContext ac=new ClassPathXmlApplicationContext("beanFactoryTest-tx.xml");

        UserService userService=(UserService)ac.getBean("userService");
        User user = new User();
        user.setName("张三ccc");
        user.setAge(20);
        user.setSex("man");

        userService.save(user);

    }
```

- 当注释调方法中的throw new RuntimeException时保存用户到数据库
- 当添加throw new RuntimeException时结果抛出异常，用户没有添加成功

​    运行结果：

```
java.lang.RuntimeException: aa

	at com.zchuber.springsourcedeepdiving.tx.UserServiceImpl.save(UserServiceImpl.java:27)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:344)
```

- 当将抛出RuntimeException改为MyException时数据添加成功

结论：spring事务中发生RuntimeException的异常时事务会回滚，而非RuntimeException的异常则不会，数据修改仍旧会成功

## 事务自定义标签

spring事务的xml配置是通过TxNamespaceHandler

```
public class TxNamespaceHandler extends NamespaceHandlerSupport {

   static final String TRANSACTION_MANAGER_ATTRIBUTE = "transaction-manager";

   static final String DEFAULT_TRANSACTION_MANAGER_BEAN_NAME = "transactionManager";


   static String getTransactionManagerName(Element element) {
      return (element.hasAttribute(TRANSACTION_MANAGER_ATTRIBUTE) ?
            element.getAttribute(TRANSACTION_MANAGER_ATTRIBUTE) : DEFAULT_TRANSACTION_MANAGER_BEAN_NAME);
   }


   @Override
   public void init() {
      registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
      //解析annotation-driver的解析器
      registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
      registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
   }

}
```

AnnotationDrivenBeanDefinitionParser中的重点方法是parse方法

AnnotationDrivenBeanDefinitionParser#parse：

```
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		registerTransactionalEventListenerFactory(parserContext);
		String mode = element.getAttribute("mode");
		//解析mode为aspectj
		if ("aspectj".equals(mode)) {
			// mode="aspectj"
			registerTransactionAspect(element, parserContext);
			if (ClassUtils.isPresent("javax.transaction.Transactional", getClass().getClassLoader())) {
				registerJtaTransactionAspect(element, parserContext);
			}
		}
		else {
			//解析mode为proxy
			// mode="proxy"
			AopAutoProxyConfigurer.configureAutoProxyCreator(element, parserContext);
		}
		return null;
	}
```

可以为元素添加mode的属性：

```
	<tx:annotation-driven transaction-manager="txManager" mode="aspectj"/>
```

AnnotationDrivenBeanDefinitionParser.AopAutoProxyConfigurer#configureAutoProxyCreator：

```
public static void configureAutoProxyCreator(Element element, ParserContext parserContext) {
   AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);
	
	//使用默认的advisor的beanName
   String txAdvisorBeanName = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME;
   if (!parserContext.getRegistry().containsBeanDefinition(txAdvisorBeanName)) {
   	//解析元素节点
      Object eleSource = parserContext.extractSource(element);

      //创建AnnotationTransactionAttributeSource的BeanDefinition
      RootBeanDefinition sourceDef = new RootBeanDefinition(
            "org.springframework.transaction.annotation.AnnotationTransactionAttributeSource");
      sourceDef.setSource(eleSource);
      sourceDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
      //获得beanName
      String sourceName = parserContext.getReaderContext().registerWithGeneratedName(sourceDef);

      // 创建事务拦截器
      RootBeanDefinition interceptorDef = new RootBeanDefinition(TransactionInterceptor.class);
      interceptorDef.setSource(eleSource);
      interceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
      //设置属性transactionManagerBeanName值
      registerTransactionManager(element, interceptorDef);
      interceptorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
      String interceptorName = parserContext.getReaderContext().registerWithGeneratedName(interceptorDef);

      // 创建BeanFactoryTransactionAttributeSourceAdvisor
      RootBeanDefinition advisorDef = new RootBeanDefinition(BeanFactoryTransactionAttributeSourceAdvisor.class);
      advisorDef.setSource(eleSource);
      advisorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
      //将上面创建的两个BeanDefinition：sourceName、interceptorName设置给advisorDef的属性
      advisorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
      advisorDef.getPropertyValues().add("adviceBeanName", interceptorName);
      if (element.hasAttribute("order")) {
         advisorDef.getPropertyValues().add("order", element.getAttribute("order"));
      }
      //注册advisorDef到ioc容器中
      parserContext.getRegistry().registerBeanDefinition(txAdvisorBeanName, advisorDef);

      CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), eleSource);
      compositeDef.addNestedComponent(new BeanComponentDefinition(sourceDef, sourceName));
      compositeDef.addNestedComponent(new BeanComponentDefinition(interceptorDef, interceptorName));
      compositeDef.addNestedComponent(new BeanComponentDefinition(advisorDef, txAdvisorBeanName));
      parserContext.registerComponent(compositeDef);
   }
}
```

AopNamespaceUtils#registerAutoProxyCreatorIfNecessary：

```
	public static void registerAutoProxyCreatorIfNecessary(
			ParserContext parserContext, Element sourceElement) {
		//注册autoProxyCreator的BeanDefinition
		BeanDefinition beanDefinition = AopConfigUtils.registerAutoProxyCreatorIfNecessary(
				parserContext.getRegistry(), parserContext.extractSource(sourceElement));
		useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
		registerComponentIfNecessary(beanDefinition, parserContext);
	}
```

AopConfigUtils#registerAutoProxyCreatorIfNecessary：

```
	public static BeanDefinition registerAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {
		//注册InfrastructureAdvisorAutoProxyCreator的BeanDefinition
		return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
	}
```

InfrastructureAdvisorAutoProxyCreator的继承体系：

![1654387906428](D:\学习\spring源码深度解析\assets\1654387906428.png)

实现了BeanPostProcessor接口。

AbstractAutoProxyCreator#postProcessAfterInitialization：

```
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			//根据bean对应的class和beanName获得缓存key
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				//创建代理的方法
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

AbstractAutoProxyCreator#wrapIfNecessary：

主要逻辑是获得bean对应的advisors，利用advisors创建代理对象

```
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		//如果已经为该bean创建了代理对象，则返回
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		//该bean没有对应的advisor，则返回
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		//如果为基础设施的类和需要跳过的类，添加缓存key到advisedBean则返回
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		//获得bean对应的advisor
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			//根据advisor创建代理对象
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean：

```
	protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
		//获取符合条件的advisor
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}
```

AbstractAdvisorAutoProxyCreator#findEligibleAdvisors：

我们之前提到的BeanFactoryTransactionAttributeSourceAdvisor在此发生作用。

```
	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
	    //发现所有的advisor
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		//挑选出符合条件的advisor
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
```

##### BeanFactoryTransactionAttributeSourceAdvisor

BeanFactoryTransactionAttributeSourceAdvisor类实现了PointcutAdvisor接口。可以结合前面创建BeanFactoryTransactionAttributeSourceAdvisor的BeanDefinition进行理解。

![1654393121218](D:\学习\spring源码深度解析\assets\1654393121218.png)

```
public class BeanFactoryTransactionAttributeSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

	@Nullable
	private TransactionAttributeSource transactionAttributeSource;
	
	//创建Pointcut
	private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
		@Override
		@Nullable
		protected TransactionAttributeSource getTransactionAttributeSource() {
		    //返回transactionAttributeSource
			return transactionAttributeSource;
		}
	};


	//设置transactionAttributeSource，从前面设置BeanFactoryTransactionAttributeSourceAdvisor的BeanDefinition时可以看到设置的其实是AnnotationTransactionAttributeSource
	public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
		this.transactionAttributeSource = transactionAttributeSource;
	}

	//为Pointcut设置classFilter
	public void setClassFilter(ClassFilter classFilter) {
		this.pointcut.setClassFilter(classFilter);
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

}
```

AbstractAdvisorAutoProxyCreator#findAdvisorsThatCanApply：

```
	protected List<Advisor> findAdvisorsThatCanApply(
			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
		try {
		    //交给工具类处理
			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
		}
		finally {
			ProxyCreationContext.setCurrentProxiedBeanName(null);
		}
	}
```

AopUtils#findAdvisorsThatCanApply：

```
	public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
	    //candidateAdvisors为空直接返回
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new ArrayList<>();
		//处理IntroductionAdvisor类型的Advisor
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
        	//跳过已经处理过的IntroductionAdvisor
			if (candidate instanceof IntroductionAdvisor) {
				continue;
			}
			//接着选择
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}
```

AopUtils#canApply：

```
	public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
		if (advisor instanceof IntroductionAdvisor) {
			return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
		}
		//如果是PointcutAdvisor，而BeanFactoryTransactionAttributeSourceAdvisor是PointcutAdvisor的实现类，条件成立
		else if (advisor instanceof PointcutAdvisor) {
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
			//从advisor中取出Pointcut，BeanFactoryTransactionAttributeSourceAdvisor中对应的Pointcut为TransactionAttributeSourcePointcut
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			// It doesn't have a pointcut so we assume it applies.
			return true;
		}
	}
```

AopUtils#canApply：

```
	public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		//通过classFilter判断targetClass是否满足条件
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}
		//如果是TransactionAttributeSourcePointcut则返回this
		MethodMatcher methodMatcher = pc.getMethodMatcher();
		if (methodMatcher == MethodMatcher.TRUE) {
			return true;
		}

		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		//对于TransactionAttributeSourcePointcut来说返回false
		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
		}

		Set<Class<?>> classes = new LinkedHashSet<>();
		if (!Proxy.isProxyClass(targetClass)) {
			classes.add(ClassUtils.getUserClass(targetClass));
		}
		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

		for (Class<?> clazz : classes) {
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
			for (Method method : methods) {
				if (introductionAwareMethodMatcher != null ?
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :		
						//进入TransactionAttributeSourcePointcut的matches中进行方法匹配
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}
```

TransactionAttributeSourcePointcut#matches：

```
	public boolean matches(Method method, Class<?> targetClass) {
		//获得TransactionAttributeSource，其实就是AnnotationTransactionAttributeSource
		TransactionAttributeSource tas = getTransactionAttributeSource();
		return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
	}
```

AnnotationTransactionAttributeSource的继承体系：

![1654394244890](D:\学习\spring源码深度解析\assets\1654394244890.png)

AbstractFallbackTransactionAttributeSource#getTransactionAttribute：

```
	public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}

		//先从缓存中取
		Object cacheKey = getCacheKey(method, targetClass);
		TransactionAttribute cached = this.attributeCache.get(cacheKey);
		if (cached != null) {
			// Value will either be canonical value indicating there is no transaction attribute,
			// or an actual transaction attribute.
			if (cached == NULL_TRANSACTION_ATTRIBUTE) {
				return null;
			}
			else {
				return cached;
			}
		}
		else {
			//缓存中没有则进行提前处TransactionAttribute
			TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
			// Put it in the cache.
			if (txAttr == null) {
				this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
			}
			else {
				String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
				if (txAttr instanceof DefaultTransactionAttribute) {
					DefaultTransactionAttribute dta = (DefaultTransactionAttribute) txAttr;
					dta.setDescriptor(methodIdentification);
					dta.resolveAttributeStrings(this.embeddedValueResolver);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
				}
				this.attributeCache.put(cacheKey, txAttr);
			}
			return txAttr;
		}
	}
```

AbstractFallbackTransactionAttributeSource#computeTransactionAttribute：

```
	protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		// 跳过非public方法
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}

		// 获得目标类的方法
		Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

		// 从目标类的方法中获得txAttr
		TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
		if (txAttr != null) {
			return txAttr;
		}

		// Second try is the transaction attribute on the target class.
		txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
		if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
			return txAttr;
		}

		if (specificMethod != method) {
			// Fallback is to look at the original method.
			txAttr = findTransactionAttribute(method);
			if (txAttr != null) {
				return txAttr;
			}
			// Last fallback is the class of the original method.
			txAttr = findTransactionAttribute(method.getDeclaringClass());
			if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
				return txAttr;
			}
		}

		return null;
	}
```

AnnotationTransactionAttributeSource#findTransactionAttribute：

```
	protected TransactionAttribute findTransactionAttribute(Method method) {
		return determineTransactionAttribute(method);
	}
```

AnnotationTransactionAttributeSource#determineTransactionAttribute：

```
	protected TransactionAttribute determineTransactionAttribute(AnnotatedElement element) {
		for (TransactionAnnotationParser parser : this.annotationParsers) {
			TransactionAttribute attr = parser.parseTransactionAnnotation(element);
			if (attr != null) {
				return attr;
			}
		}
		return null;
	}
```

##### 创建AnnotationTransactionAttributeSource时设置annotationParsers

```
	public AnnotationTransactionAttributeSource(boolean publicMethodsOnly) {
		this.publicMethodsOnly = publicMethodsOnly;
		if (jta12Present || ejb3Present) {
			this.annotationParsers = new LinkedHashSet<>(4);
			this.annotationParsers.add(new SpringTransactionAnnotationParser());
			if (jta12Present) {
				this.annotationParsers.add(new JtaTransactionAnnotationParser());
			}
			if (ejb3Present) {
				this.annotationParsers.add(new Ejb3TransactionAnnotationParser());
			}
		}
		else {
		    //默认使用的是SpringTransactionAnnotationParser
			this.annotationParsers = Collections.singleton(new SpringTransactionAnnotationParser());
		}
	}
```

SpringTransactionAnnotationParser#parseTransactionAnnotation：

在该方法中解析Transactional注解的属性。

```
	public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
		//获得Transactional注解信息
		AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(
				element, Transactional.class, false, false);
		if (attributes != null) {
			return parseTransactionAnnotation(attributes);
		}
		else {
			return null;
		}
	}
```

SpringTransactionAnnotationParser#parseTransactionAnnotation：

```
	protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
		RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();
		//从注解中解析出transactionAttribute的属性封装到RuleBasedTransactionAttribute对象中
		Propagation propagation = attributes.getEnum("propagation");
		rbta.setPropagationBehavior(propagation.value());
		Isolation isolation = attributes.getEnum("isolation");
		rbta.setIsolationLevel(isolation.value());

		rbta.setTimeout(attributes.getNumber("timeout").intValue());
		String timeoutString = attributes.getString("timeoutString");
		Assert.isTrue(!StringUtils.hasText(timeoutString) || rbta.getTimeout() < 0,
				"Specify 'timeout' or 'timeoutString', not both");
		rbta.setTimeoutString(timeoutString);

		rbta.setReadOnly(attributes.getBoolean("readOnly"));
		rbta.setQualifier(attributes.getString("value"));
		rbta.setLabels(Arrays.asList(attributes.getStringArray("label")));

		List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
		for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		rbta.setRollbackRules(rollbackRules);

		return rbta;
	}
```

## 事务拦截器

TransactionInterceptor实现了MethodInterceptor接口，的继承体系如下：

![1654411176426](D:\学习\spring源码深度解析\assets\1654411176426.png)

会被spring封装为拦截链在方法调用时触发执行。

#### TransactionInterceptor#invoke

```
public Object invoke(MethodInvocation invocation) throws Throwable {
	//获得目标对象
   Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

   // 方法在事务中执行
   return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
      @Override
      @Nullable
      public Object proceedWithInvocation() throws Throwable {
         return invocation.proceed();
      }
      @Override
      public Object getTarget() {
         return invocation.getThis();
      }
      @Override
      public Object[] getArguments() {
         return invocation.getArguments();
      }
   });
}
```

#### TransactionAspectSupport#invokeWithinTransaction

事务处理的主要方法

```
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		//获得TransactionAttributeSource对象，也就是AnnotationTransactionAttributeSource
		TransactionAttributeSource tas = getTransactionAttributeSource();
		//从tas中获得txAttri对象
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		//从txAttrix中获取配置的TransactionManager
		final TransactionManager tm = determineTransactionManager(txAttr);
		
		//将tm强制转换为PlatformTransactionManager
		PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
		//如果txAttr为空，或者ptm不是CallbackPreferringPlatformTransactionManager的实现类
		if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
			// 获得txInfo对象，在txInfo中封装ptm，txAttr等事务相关信息
			TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

			Object retVal;
			try {
				// 执行aop的拦截链的下一个拦截器
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// 处理异常
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}

			if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
				// Set rollback-only in case of Vavr failure matching our rollback rules...
				TransactionStatus status = txInfo.getTransactionStatus();
				if (status != null && txAttr != null) {
					retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
				}
			}

			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			Object result;
			final ThrowableHolder throwableHolder = new ThrowableHolder();

			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				result = ((CallbackPreferringPlatformTransactionManager) ptm).execute(txAttr, status -> {
					TransactionInfo txInfo = prepareTransactionInfo(ptm, txAttr, joinpointIdentification, status);
					try {
						Object retVal = invocation.proceedWithInvocation();
						if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
							// Set rollback-only in case of Vavr failure matching our rollback rules...
							retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
						}
						return retVal;
					}
					catch (Throwable ex) {
						if (txAttr.rollbackOn(ex)) {
							// A RuntimeException: will lead to a rollback.
							if (ex instanceof RuntimeException) {
								throw (RuntimeException) ex;
							}
							else {
								throw new ThrowableHolderException(ex);
							}
						}
						else {
							// A normal return value: will lead to a commit.
							throwableHolder.throwable = ex;
							return null;
						}
					}
					finally {
						cleanupTransactionInfo(txInfo);
					}
				});
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
			catch (TransactionSystemException ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
					ex2.initApplicationException(throwableHolder.throwable);
				}
				throw ex2;
			}
			catch (Throwable ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				}
				throw ex2;
			}

			// Check result state: It might indicate a Throwable to rethrow.
			if (throwableHolder.throwable != null) {
				throw throwableHolder.throwable;
			}
			return result;
		}
	}
```

#### TransactionAspectSupport#determineTransactionManager

```
	protected TransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
		// txAttrix中没有设置获得默认的TransactionManager
		if (txAttr == null || this.beanFactory == null) {
			return getTransactionManager();
		}

		String qualifier = txAttr.getQualifier();
		if (StringUtils.hasText(qualifier)) {
			return determineQualifiedTransactionManager(this.beanFactory, qualifier);
		}
		//拦截器中配置了transactionManagerBeanName
		else if (StringUtils.hasText(this.transactionManagerBeanName)) {
			return determineQualifiedTransactionManager(this.beanFactory, this.transactionManagerBeanName);
		}
		//使用默认txManager
		else {
			TransactionManager defaultTransactionManager = getTransactionManager();
			if (defaultTransactionManager == null) {
				defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
				if (defaultTransactionManager == null) {
					defaultTransactionManager = this.beanFactory.getBean(TransactionManager.class);
					this.transactionManagerCache.putIfAbsent(
							DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
				}
			}
			return defaultTransactionManager;
		}
	}

```

TransactionAspectSupport#determineQualifiedTransactionManager：

```
	private TransactionManager determineQualifiedTransactionManager(BeanFactory beanFactory, String qualifier) {
	   //尝试从缓存中获取
		TransactionManager txManager = this.transactionManagerCache.get(qualifier);
		if (txManager == null) {
			//从ioc容器中利用beanName获取TransactionManager类型的bean
			txManager = BeanFactoryAnnotationUtils.qualifiedBeanOfType(
					beanFactory, TransactionManager.class, qualifier);
			//将txManager放入到缓存
			this.transactionManagerCache.putIfAbsent(qualifier, txManager);
		}
		return txManager;
	}
```

#### TransactionAspectSupport#createTransactionIfNecessary

```
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
      @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

   // 如果txAttr的名称为空，将其封装在DelegatingTransactionAttribute，并将joinpointIdentification作为name
   if (txAttr != null && txAttr.getName() == null) {
      txAttr = new DelegatingTransactionAttribute(txAttr) {
         @Override
         public String getName() {
            return joinpointIdentification;
         }
      };
   }

   TransactionStatus status = null;
   if (txAttr != null) {
      if (tm != null) {
         //tm从txAttr中获得事务的状态
         status = tm.getTransaction(txAttr);
      }
      else {
         if (logger.isDebugEnabled()) {
            logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                  "] because no transaction manager has been configured");
         }
      }
   }
   return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

### 实际的事务处理流程

#### AbstractPlatformTransactionManager#getTransaction

```
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
      throws TransactionException {

   // 获得TransactionDefinition，没有使用默认的
   TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());
	//创建DataSourceTransanctionObject对象
   Object transaction = doGetTransaction();
   boolean debugEnabled = logger.isDebugEnabled();
	
	//如果存在事务
   if (isExistingTransaction(transaction)) {
      // 根据传播特性的设置进行处理
      return handleExistingTransaction(def, transaction, debugEnabled);
   }

   // Check definition settings for new transaction.
   if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
      throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
   }

   // 如果没有存在的事务，传播属性为PROPAGATION_MANDATORY则抛出异常
   if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
      throw new IllegalTransactionStateException(
            "No existing transaction found for transaction marked with propagation 'mandatory'");
   }
   //如果没有存在的事务，处理传播属性为PROPAGATION_REQUIRED、PROPAGATION_REQUIRES_NEW、PROPAGATION_NESTED
   else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
         def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
         def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        //创建suspendedResources，不需要挂起任何资源
      SuspendedResourcesHolder suspendedResources = suspend(null);
      if (debugEnabled) {
         logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
      }
      try {
         //开启新的事务，设置transaction中connectionHolder的属性
         return startTransaction(def, transaction, debugEnabled, suspendedResources);
      }
      catch (RuntimeException | Error ex) {
         resume(null, suspendedResources);
         throw ex;
      }
   }
   else {
      // Create "empty" transaction: no actual transaction, but potentially synchronization.
      if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
         logger.warn("Custom isolation level specified but no actual transaction initiated; " +
               "isolation level will effectively be ignored: " + def);
      }
      //新建transactionStatus，
      boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
      return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
   }
}
```

DataSourceTransactionManager#doGetTransaction：

```
protected Object doGetTransaction() {
   //新建DataSourceTransactionObject对象
   DataSourceTransactionObject txObject = new DataSourceTransactionObject();
   //设置txObject中是否运行savepointAllowed
   txObject.setSavepointAllowed(isNestedTransactionAllowed());
   //获得conHolder，该对象中包含connection
   ConnectionHolder conHolder =
         (ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
   //设置txObject的connectionHolder属性
   txObject.setConnectionHolder(conHolder, false);
   return txObject;
}
```

#### DataSourceTransactionManager#isExistingTransaction

```
protected boolean isExistingTransaction(Object transaction) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
   //根据txObject中是否存在connectionHolder，或者即使存在其transactionActive属性是否为true
   return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
}
```

#### AbstractPlatformTransactionManager#handleExistingTransaction

```
private TransactionStatus handleExistingTransaction(
      TransactionDefinition definition, Object transaction, boolean debugEnabled)
      throws TransactionException {
	//传播特性为PROPAGATION_NEVER抛出异常
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
      throw new IllegalTransactionStateException(
            "Existing transaction found for transaction marked with propagation 'never'");
   }
	//传播特性为PROPAGATION_NOT_SUPPORTED
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
      if (debugEnabled) {
         logger.debug("Suspending current transaction");
      }
      //挂起事务
      Object suspendedResources = suspend(transaction);
      boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
      //更新事务状态
      return prepareTransactionStatus(
            definition, null, false, newSynchronization, debugEnabled, suspendedResources);
   }
	//传播特性为PROPAGATION_REQUIRES_NEW
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
      if (debugEnabled) {
         logger.debug("Suspending current transaction, creating new transaction with name [" +
               definition.getName() + "]");
      }
      //挂起当前transaction
      SuspendedResourcesHolder suspendedResources = suspend(transaction);
      try {
      	//开启新的事务，新事务中包含被挂起的事务
         return startTransaction(definition, transaction, debugEnabled, suspendedResources);
      }
      catch (RuntimeException | Error beginEx) {
         resumeAfterBeginException(transaction, suspendedResources, beginEx);
         throw beginEx;
      }
   }
	
	//传播特性为PROPAGATION_NESTED
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
      //对于不允许嵌套事务会抛出异常
      if (!isNestedTransactionAllowed()) {
         throw new NestedTransactionNotSupportedException(
               "Transaction manager does not allow nested transactions by default - " +
               "specify 'nestedTransactionAllowed' property with value 'true'");
      }
      if (debugEnabled) {
         logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
      }
      //通过保存点的方式实现嵌套事务（默认方式）
      if (useSavepointForNestedTransaction()) {
         // 创建新的TransactionStatus
         DefaultTransactionStatus status =
               prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
         //创建保存点
         status.createAndHoldSavepoint();
         return status;
      }
      else {
         //如果嵌入事务不是通过保存点的方式实现，创建新的事务，此时相当于REQUIRED_NEW
         return startTransaction(definition, transaction, debugEnabled, null);
      }
   }

   if (isValidateExistingTransaction()) {
      if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
         Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
         if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
            Constants isoConstants = DefaultTransactionDefinition.constants;
            throw new IllegalTransactionStateException("Participating transaction with definition [" +
                  definition + "] specifies isolation level which is incompatible with existing transaction: " +
                  (currentIsolationLevel != null ?
                        isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                        "(unknown)"));
         }
      }
      if (!definition.isReadOnly()) {
         if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            throw new IllegalTransactionStateException("Participating transaction with definition [" +
                  definition + "] is not marked as read-only but existing transaction is");
         }
      }
   }
   //默认事务开启同步
   boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
   //创建transactionStatus后准备同步
   return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```

AbstractPlatformTransactionManager#suspend：

```
	protected final SuspendedResourcesHolder suspend(@Nullable Object transaction) throws TransactionException {
	    //如果Synchronization被激活
		if (TransactionSynchronizationManager.isSynchronizationActive()) {
			//获得所有的suspendedSynchronizations
			List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
			try {
				Object suspendedResources = null;
				if (transaction != null) {
				   //移除该transaction对象和线程绑定相关的资源
					suspendedResources = doSuspend(transaction);
				}
				String name = TransactionSynchronizationManager.getCurrentTransactionName();
				TransactionSynchronizationManager.setCurrentTransactionName(null);
				boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
				TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
				Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
				TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
				boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
				TransactionSynchronizationManager.setActualTransactionActive(false);
				//创建SuspendResourceHolder封装transaction的信息
				return new SuspendedResourcesHolder(
						suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
			}
			catch (RuntimeException | Error ex) {
				// doSuspend failed - original transaction is still active...
				doResumeSynchronization(suspendedSynchronizations);
				throw ex;
			}
		}
		else if (transaction != null) {
			//没有同步需求，暂停当前transaction，将其封装到SuspendResourceHolder中
			Object suspendedResources = doSuspend(transaction);
			return new SuspendedResourcesHolder(suspendedResources);
		}
		else {
			// Neither transaction nor synchronization active.
			return null;
		}
	}
```

AbstractPlatformTransactionManager#doSuspendSynchronization：

```
	private List<TransactionSynchronization> doSuspendSynchronization() {
		//获得所有的TransactionSynchronization
		List<TransactionSynchronization> suspendedSynchronizations =
				TransactionSynchronizationManager.getSynchronizations();
	    //调用synchronization的suspend钩子
		for (TransactionSynchronization synchronization : suspendedSynchronizations) {
			synchronization.suspend();
		}
		//移除所有的TransactionSynchronization
		TransactionSynchronizationManager.clearSynchronization();
		return suspendedSynchronizations;
	}
```

DataSourceTransactionManager#doSuspend：

```
protected Object doSuspend(Object transaction) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
   //设置txObject中的connectionHolder为null
   txObject.setConnectionHolder(null);
   //移除DataSource关联的connect
   return TransactionSynchronizationManager.unbindResource(obtainDataSource());
}
```

#### AbstractPlatformTransactionManager#prepareTransactionStatus

```
protected final DefaultTransactionStatus prepareTransactionStatus(
      TransactionDefinition definition, @Nullable Object transaction, boolean newTransaction,
      boolean newSynchronization, boolean debug, @Nullable Object suspendedResources) {
	//创建DefaultTransactionStatus
   DefaultTransactionStatus status = newTransactionStatus(
         definition, transaction, newTransaction, newSynchronization, debug, suspendedResources);
   //设置事务同步的配置
   prepareSynchronization(status, definition);
   return status;
}
```

AbstractPlatformTransactionManager#newTransactionStatus：

```
	protected DefaultTransactionStatus newTransactionStatus(
			TransactionDefinition definition, @Nullable Object transaction, boolean newTransaction,
			boolean newSynchronization, boolean debug, @Nullable Object suspendedResources) {

		boolean actualNewSynchronization = newSynchronization &&
				!TransactionSynchronizationManager.isSynchronizationActive();
		return new DefaultTransactionStatus(
				transaction, newTransaction, actualNewSynchronization,
				definition.isReadOnly(), debug, suspendedResources);
	}
```

AbstractPlatformTransactionManager#prepareSynchronization：

```
protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
	//是否开启同步状态
   if (status.isNewSynchronization()) {
     //设置TransactionSynchronizationManager的属性
      TransactionSynchronizationManager.setActualTransactionActive(status.hasTransaction());
      TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(
            definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT ?
                  definition.getIsolationLevel() : null);
      TransactionSynchronizationManager.setCurrentTransactionReadOnly(definition.isReadOnly());
      TransactionSynchronizationManager.setCurrentTransactionName(definition.getName());
      TransactionSynchronizationManager.initSynchronization();
   }
}
```

TransactionSynchronizationManager#initSynchronization：

初始化synchronizations

```
	public static void initSynchronization() throws IllegalStateException {
		if (isSynchronizationActive()) {
			throw new IllegalStateException("Cannot activate transaction synchronization - already active");
		}
		synchronizations.set(new LinkedHashSet<>());
	}
```

#### AbstractPlatformTransactionManager#startTransaction

```
	private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
			boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {
		//默认newSynchronization为true
		boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
		//创建DefaultTransactionStatus
		DefaultTransactionStatus status = newTransactionStatus(
				definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
	    //开启事务
		doBegin(transaction, definition);
		prepareSynchronization(status, definition);
		return status;
	}
```

DataSourceTransactionManager#doBegin：

```
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;

		try {
			//txObject中没有connectionHolder，获得事务需要同步
			if (!txObject.hasConnectionHolder() ||
					txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
				//从DataSource中获得Connection
				Connection newCon = obtainDataSource().getConnection();
				if (logger.isDebugEnabled()) {
					logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
				}
				//设置txObject的connectHolder属性
				txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
			}
			//默认事务需要开启同步
			txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
			con = txObject.getConnectionHolder().getConnection();
			
			//设置definition中的隔离级别到connect中
			Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
			txObject.setPreviousIsolationLevel(previousIsolationLevel);
			txObject.setReadOnly(definition.isReadOnly());

			//设置connection的自动提交为false
			if (con.getAutoCommit()) {
				txObject.setMustRestoreAutoCommit(true);
				if (logger.isDebugEnabled()) {
					logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
				}
				con.setAutoCommit(false);
			}

			prepareTransactionalConnection(con, definition);
			//设置ConnectionHolder中的transactionActive为true。该标志用来确认事务是否激活
			txObject.getConnectionHolder().setTransactionActive(true);
			
			//获得definition中的超时时间，并设置给connectionHolder
			int timeout = determineTimeout(definition);
			if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
				txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
			}

			//将con和DataSource绑定到线程中
			if (txObject.isNewConnectionHolder()) {
				TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
			}
		}

		catch (Throwable ex) {
			if (txObject.isNewConnectionHolder()) {
				DataSourceUtils.releaseConnection(con, obtainDataSource());
				txObject.setConnectionHolder(null, false);
			}
			throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
		}
	}

```

DataSourceTransactionManager#prepareTransactionalConnection：

```
protected void prepareTransactionalConnection(Connection con, TransactionDefinition definition)
      throws SQLException {

	//是否设置read only
   if (isEnforceReadOnly() && definition.isReadOnly()) {
      try (Statement stmt = con.createStatement()) {    
         stmt.executeUpdate("SET TRANSACTION READ ONLY");
      }
   }
}
```

#### AbstractTransactionStatus#createAndHoldSavepoint

![1654418450523](D:\学习\spring源码深度解析\assets\1654418450523.png)

DefaultTransactionStatus实现SavepointManger接口。

org.springframework.transaction.support.AbstractTransactionStatus#createAndHoldSavepoint：

```
	public void createAndHoldSavepoint() throws TransactionException {
		setSavepoint(getSavepointManager().createSavepoint());
	}
```

DefaultTransactionStatus#getSavepointManager：

```
protected SavepointManager getSavepointManager() {
   Object transaction = this.transaction;
   //对于DefaultTransactionStatus返回true
   if (!(transaction instanceof SavepointManager)) {
      throw new NestedTransactionNotSupportedException(
            "Transaction object [" + this.transaction + "] does not support savepoints");
   }
   return (SavepointManager) transaction;
}
```

此处的transaction为DataSourceTransactionObject，DataSourceTransactionObject继承了JdbcTransactionObjectSupport。

![1654418803192](D:\学习\spring源码深度解析\assets\1654418803192.png)

JdbcTransactionObjectSupport#createSavepoint：

```
public Object createSavepoint() throws TransactionException {
   ConnectionHolder conHolder = getConnectionHolderForSavepoint();
   try {
      if (!conHolder.supportsSavepoints()) {
         throw new NestedTransactionNotSupportedException(
               "Cannot create a nested transaction because savepoints are not supported by your JDBC driver");
      }
      if (conHolder.isRollbackOnly()) {
         throw new CannotCreateTransactionException(
               "Cannot create savepoint for transaction which is already marked as rollback-only");
      }
      return conHolder.createSavepoint();
   }
   catch (SQLException ex) {
      throw new CannotCreateTransactionException("Could not create JDBC savepoint", ex);
   }
}
```

JdbcTransactionObjectSupport#getConnectionHolderForSavepoint：

```
protected ConnectionHolder getConnectionHolderForSavepoint() throws TransactionException {
   if (!isSavepointAllowed()) {
      throw new NestedTransactionNotSupportedException(
            "Transaction manager does not allow nested transactions");
   }
   if (!hasConnectionHolder()) {
      throw new TransactionUsageException(
            "Cannot create nested transaction when not exposing a JDBC transaction");
   }
   //获得transaction关联的connectionHolder
   return getConnectionHolder();
}
```

### 事务异常处理

spring中使用的TransactionAttribute为RuleBasedTransactionAttribute，RuleBasedTransactionAttribute的继承体系如下：

![1654419838291](D:\学习\spring源码深度解析\assets\1654419838291.png)

TransactionAspectSupport#completeTransactionAfterThrowing

```
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
   if (txInfo != null && txInfo.getTransactionStatus() != null) {
      if (logger.isTraceEnabled()) {
         logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
               "] after exception: " + ex);
      }
      //如果存在transactionAttribute，并且满足回滚条件
      if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
         try {
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            throw ex2;
         }
      }
      else {
         // 如果不满足回滚规则，即使发生异常照样提交
         try {
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            throw ex2;
         }
      }
   }
}
```

RuleBasedTransactionAttribute#rollbackOn：

```
	public boolean rollbackOn(Throwable ex) {
		RollbackRuleAttribute winner = null;
		int deepest = Integer.MAX_VALUE;
		//根据回滚规则进行回滚
		if (this.rollbackRules != null) {
			for (RollbackRuleAttribute rule : this.rollbackRules) {
				int depth = rule.getDepth(ex);
				if (depth >= 0 && depth < deepest) {
					deepest = depth;
					winner = rule;
				}
			}
		}

		//没有设置回滚规则，调用父类的
		if (winner == null) {
			return super.rollbackOn(ex);
		}

		return !(winner instanceof NoRollbackRuleAttribute);
	}
```

DefaultTransactionAttribute#rollbackOn：

```
public boolean rollbackOn(Throwable ex) {
   return (ex instanceof RuntimeException || ex instanceof Error);
}
```

spring默认回滚的异常是RuntimeException和Error的子类。

#### AbstractPlatformTransactionManager#rollback

```
	public final void rollback(TransactionStatus status) throws TransactionException {
		//如果transactionStatus已经完成，抛出异常，不允许重复回滚
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		//执行回滚
		processRollback(defStatus, false);
	}
```

AbstractPlatformTransactionManager#processRollback：

```
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
   try {
      boolean unexpectedRollback = unexpected;

      try {
      	//触发beforeCompletion同步
         triggerBeforeCompletion(status);
		//如果存在保存点
         if (status.hasSavepoint()) {
            if (status.isDebug()) {
               logger.debug("Rolling back transaction to savepoint");
            }
            //回滚至上一个保存点
            status.rollbackToHeldSavepoint();
         }
         //如果是新事务
         else if (status.isNewTransaction()) {
            if (status.isDebug()) {
               logger.debug("Initiating transaction rollback");
            }
            //回滚整个事务
            doRollback(status);
         }
         else {
            // Participating in larger transaction
            if (status.hasTransaction()) {
               if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                  if (status.isDebug()) {
                     logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                  }
                  doSetRollbackOnly(status);
               }
               else {
                  if (status.isDebug()) {
                     logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                  }
               }
            }
            else {
               logger.debug("Should roll back transaction but cannot - no transaction available");
            }
            // Unexpected rollback only matters here if we're asked to fail early
            if (!isFailEarlyOnGlobalRollbackOnly()) {
               unexpectedRollback = false;
            }
         }
      }
      catch (RuntimeException | Error ex) {
      	//回滚是发送异常，触发afterCompletion同步
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
         throw ex;
      }
      	//触发afterCompletion同步
      triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

      // Raise UnexpectedRollbackException if we had a global rollback-only marker
      if (unexpectedRollback) {
         throw new UnexpectedRollbackException(
               "Transaction rolled back because it has been marked as rollback-only");
      }
   }
   finally {
   	//回滚结束后清理工作
      cleanupAfterCompletion(status);
   }
}
```

AbstractTransactionStatus#rollbackToHeldSavepoint：

```
public void rollbackToHeldSavepoint() throws TransactionException {
	//获得保存点
   Object savepoint = getSavepoint();
   if (savepoint == null) {
      throw new TransactionUsageException(
            "Cannot roll back to savepoint - no savepoint associated with current transaction");
   }
   
   //利用connection对保存点进行回滚、释放
   getSavepointManager().rollbackToSavepoint(savepoint);
   getSavepointManager().releaseSavepoint(savepoint);
   //清空保存点
   setSavepoint(null);
}
```

JdbcTransactionObjectSupport#rollbackToSavepoint：

```
public void rollbackToSavepoint(Object savepoint) throws TransactionException {
   ConnectionHolder conHolder = getConnectionHolderForSavepoint();
   try {
   	//利用connection的rollback进行回滚
      conHolder.getConnection().rollback((Savepoint) savepoint);
      conHolder.resetRollbackOnly();
   }
   catch (Throwable ex) {
      throw new TransactionSystemException("Could not roll back to JDBC savepoint", ex);
   }
}
```

JdbcTransactionObjectSupport#releaseSavepoint：

```
public void releaseSavepoint(Object savepoint) throws TransactionException {
   ConnectionHolder conHolder = getConnectionHolderForSavepoint();
   try {
   	//利用connection释放保存点
      conHolder.getConnection().releaseSavepoint((Savepoint) savepoint);
   }
   catch (Throwable ex) {
      logger.debug("Could not explicitly release JDBC savepoint", ex);
   }
}
```

##### DataSourceTransactionManager#doRollback

```
protected void doRollback(DefaultTransactionStatus status) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
   Connection con = txObject.getConnectionHolder().getConnection();
   if (status.isDebug()) {
      logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
   }
   try {
   	//利用connection进行回滚
      con.rollback();
   }
   catch (SQLException ex) {
      throw translateException("JDBC rollback", ex);
   }
}
```

##### AbstractPlatformTransactionManager#cleanupAfterCompletion

```
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
	//status状态设置完成
   status.setCompleted();
   //清理事务中的同步
   if (status.isNewSynchronization()) {
      TransactionSynchronizationManager.clear();
   }
   //清空事务
   if (status.isNewTransaction()) {
      doCleanupAfterCompletion(status.getTransaction());
   }
   if (status.getSuspendedResources() != null) {
      if (status.isDebug()) {
         logger.debug("Resuming suspended transaction after completion of inner transaction");
      }
      Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
      //恢复挂起的事务
      resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
   }
}
```

DataSourceTransactionManager#doCleanupAfterCompletion：

```
protected void doCleanupAfterCompletion(Object transaction) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

   // 从线程中移除connectionHolder
   if (txObject.isNewConnectionHolder()) {
      TransactionSynchronizationManager.unbindResource(obtainDataSource());
   }

      //重置连接为事务开启前连接的设置
   Connection con = txObject.getConnectionHolder().getConnection();
   try {
   		//恢复连接的自动提交
      if (txObject.isMustRestoreAutoCommit()) {
         con.setAutoCommit(true);
      }
      DataSourceUtils.resetConnectionAfterTransaction(
            con, txObject.getPreviousIsolationLevel(), txObject.isReadOnly());
   }
   catch (Throwable ex) {
      logger.debug("Could not reset JDBC Connection after transaction", ex);
   }
	//如果是新的connectionHolder，释放connection
   if (txObject.isNewConnectionHolder()) {
      if (logger.isDebugEnabled()) {
         logger.debug("Releasing JDBC Connection [" + con + "] after transaction");
      }
      DataSourceUtils.releaseConnection(con, this.dataSource);
   }

   txObject.getConnectionHolder().clear();
}
```

#### 释放连接

如果是连接是datasource管理，释放连接只是将连接的引用数减一。

DataSourceUtils#releaseConnection

```
	public static void releaseConnection(@Nullable Connection con, @Nullable DataSource dataSource) {
		try {
			doReleaseConnection(con, dataSource);
		}
		catch (SQLException ex) {
			logger.debug("Could not close JDBC Connection", ex);
		}
		catch (Throwable ex) {
			logger.debug("Unexpected exception on closing JDBC Connection", ex);
		}
	}
```

DataSourceUtils#doReleaseConnection：

```
public static void doReleaseConnection(@Nullable Connection con, @Nullable DataSource dataSource) throws SQLException {
   if (con == null) {
      return;
   }
   //如果DataSource不为null
   if (dataSource != null) {
      //获得该线程中DataSource对应的connectionHolder
      ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
      if (conHolder != null && connectionEquals(conHolder, con)) {
         // 释放connection
         conHolder.released();
         return;
      }
   }
   //如果连接不是有DataSource管理的，关闭连接
   doCloseConnection(con, dataSource);
}
```

ConnectionHolder#released：

```
public void released() {
   //调用父类的方法
   super.released();
   if (!isOpen() && this.currentConnection != null) {
      if (this.connectionHandle != null) {
         this.connectionHandle.releaseConnection(this.currentConnection);
      }
      this.currentConnection = null;
   }
}
```

ResourceHolderSupport#released：

其实就是referenceCount--，并没有关闭connection

```
public void released() {
   this.referenceCount--;
}
```

#### 回滚后信息的清除

AbstractPlatformTransactionManager#cleanupAfterCompletion：

```
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
	//设置事务状态为完成，避免重复回滚或提交
   status.setCompleted();
   //如果是新的同步
   if (status.isNewSynchronization()) {
   	  //利用TransactionSynchronizationManager清理同步信息
      TransactionSynchronizationManager.clear();
   }
   //如果是新的事务
   if (status.isNewTransaction()) {
   	  //清除事务信息
      doCleanupAfterCompletion(status.getTransaction());
   }
   //如果status中包含被暂停的事务
   if (status.getSuspendedResources() != null) {
      if (status.isDebug()) {
         logger.debug("Resuming suspended transaction after completion of inner transaction");
      }
      Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
      //恢复暂停的事务
      resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
   }
}
```

DataSourceTransactionManager#doCleanupAfterCompletion：

```
protected void doCleanupAfterCompletion(Object transaction) {
	//将transaction强转为DataSourceTransactionObject
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

   // 如果是新的connectionHolder，解除线程和datasource的绑定
   if (txObject.isNewConnectionHolder()) {
      TransactionSynchronizationManager.unbindResource(obtainDataSource());
   }

   //重置连接
   Connection con = txObject.getConnectionHolder().getConnection();
   try {
      if (txObject.isMustRestoreAutoCommit()) {
         con.setAutoCommit(true);
      }
      DataSourceUtils.resetConnectionAfterTransaction(
            con, txObject.getPreviousIsolationLevel(), txObject.isReadOnly());
   }
   catch (Throwable ex) {
      logger.debug("Could not reset JDBC Connection after transaction", ex);
   }
	//释放连接
   if (txObject.isNewConnectionHolder()) {
      if (logger.isDebugEnabled()) {
         logger.debug("Releasing JDBC Connection [" + con + "] after transaction");
      }
      DataSourceUtils.releaseConnection(con, this.dataSource);
   }
	
	//清理connectionHolder
   txObject.getConnectionHolder().clear();
}
```

ConnectionHolder#clear：

```
public void clear() {
   super.clear();
   this.transactionActive = false;
   this.savepointsSupported = null;
   this.savepointCounter = 0;
}
```

##### 恢复暂停事务

AbstractPlatformTransactionManager#resume：

```
	protected final void resume(@Nullable Object transaction, @Nullable SuspendedResourcesHolder resourcesHolder)
			throws TransactionException {

		if (resourcesHolder != null) {
			Object suspendedResources = resourcesHolder.suspendedResources;
			if (suspendedResources != null) {
				//恢复事务
				doResume(transaction, suspendedResources);
			}
			List<TransactionSynchronization> suspendedSynchronizations = resourcesHolder.suspendedSynchronizations;
			//设置同步信息
			if (suspendedSynchronizations != null) {
				TransactionSynchronizationManager.setActualTransactionActive(resourcesHolder.wasActive);
				TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(resourcesHolder.isolationLevel);
				TransactionSynchronizationManager.setCurrentTransactionReadOnly(resourcesHolder.readOnly);
				TransactionSynchronizationManager.setCurrentTransactionName(resourcesHolder.name);
				doResumeSynchronization(suspendedSynchronizations);
			}
		}
	}
```

DataSourceTransactionManager#doResume：

```
protected void doResume(@Nullable Object transaction, Object suspendedResources) {
	//将datasource和暂停的事务绑定到线程上
   TransactionSynchronizationManager.bindResource(obtainDataSource(), suspendedResources);
}
```

AbstractPlatformTransactionManager#doResumeSynchronization：

```
private void doResumeSynchronization(List<TransactionSynchronization> suspendedSynchronizations) {
   TransactionSynchronizationManager.initSynchronization();
   //调用事务同步器并且注册到TransactionSynchronizationManager中
   for (TransactionSynchronization synchronization : suspendedSynchronizations) {
      synchronization.resume();
      TransactionSynchronizationManager.registerSynchronization(synchronization);
   }
}
```

### 事务提交

#### AbstractPlatformTransactionManager#commit

```
public final void commit(TransactionStatus status) throws TransactionException {
    //如果事务已经完成了，抛出异常，不允许重复提交
   if (status.isCompleted()) {
      throw new IllegalTransactionStateException(
            "Transaction is already completed - do not call commit or rollback more than once per transaction");
   }

   DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
   if (defStatus.isLocalRollbackOnly()) {
      if (defStatus.isDebug()) {
         logger.debug("Transactional code has requested rollback");
      }
      processRollback(defStatus, false);
      return;
   }

   if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
      if (defStatus.isDebug()) {
         logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
      }
      processRollback(defStatus, true);
      return;
   }
	//实际提交
   processCommit(defStatus);
}
```

AbstractPlatformTransactionManager#processCommit：

```
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
   try {
      boolean beforeCompletionInvoked = false;

      try {
         boolean unexpectedRollback = false;
         //供子类扩展
         prepareForCommit(status);
         //触发同步调用
         triggerBeforeCommit(status);
         triggerBeforeCompletion(status);
         beforeCompletionInvoked = true;
		//如果有保存点
         if (status.hasSavepoint()) {
            if (status.isDebug()) {
               logger.debug("Releasing transaction savepoint");
            }
            unexpectedRollback = status.isGlobalRollbackOnly();
            //释放保存点
            status.releaseHeldSavepoint();
         }
         //如果是新事务
         else if (status.isNewTransaction()) {
            if (status.isDebug()) {
               logger.debug("Initiating transaction commit");
            }
            unexpectedRollback = status.isGlobalRollbackOnly();
            //提交事务
            doCommit(status);
         }
         //如果是其它的不做其它动作
         else if (isFailEarlyOnGlobalRollbackOnly()) {
            unexpectedRollback = status.isGlobalRollbackOnly();
         }

         // Throw UnexpectedRollbackException if we have a global rollback-only
         // marker but still didn't get a corresponding exception from commit.
         if (unexpectedRollback) {
            throw new UnexpectedRollbackException(
                  "Transaction silently rolled back because it has been marked as rollback-only");
         }
      }
      catch (UnexpectedRollbackException ex) {
         // can only be caused by doCommit
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
         throw ex;
      }
      catch (TransactionException ex) {
         // can only be caused by doCommit
         if (isRollbackOnCommitFailure()) {
            doRollbackOnCommitException(status, ex);
         }
         else {
           //触发事务完成的同步
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
         }
         throw ex;
      }
      catch (RuntimeException | Error ex) {
         if (!beforeCompletionInvoked) {
            triggerBeforeCompletion(status);
         }
         doRollbackOnCommitException(status, ex);
         throw ex;
      }

      //触发事务的同步
      try {
         triggerAfterCommit(status);
      }
      finally {
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
      }

   }
   finally {
      //清理事务的状态
      cleanupAfterCompletion(status);
   }
}
```
只有事务有保存点或者是新事务才释放保存点、或者进行提交。

##### 事务提交

DataSourceTransactionManager#doCommit

```
protected void doCommit(DefaultTransactionStatus status) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
   Connection con = txObject.getConnectionHolder().getConnection();
   if (status.isDebug()) {
      logger.debug("Committing JDBC transaction on Connection [" + con + "]");
   }
   try {
      //调用连接的commit方法进行提交
      con.commit();
   }
   catch (SQLException ex) {
      throw translateException("JDBC commit", ex);
   }
}
```

