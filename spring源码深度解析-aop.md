# AOP

代码中的公共代码比如日志、监控通过继承这种纵向的方式实现会产生大量的容易代码，需要通过横切的方式到对象中，这种方式称为AOP

## 动态AOP使用示例

添加依赖：

```
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjrt</artifactId>
            <version>1.8.9</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.aspectj/aspectjtools -->
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjtools</artifactId>
            <version>1.8.9</version>
        </dependency>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.7.4</version>
        </dependency>
```

创建类对象：

```
public class TestBean {

    private String testStr = "testStr";

    public String getTestStr() {
        return testStr;
    }

    public void setTestStr(String testStr) {
        this.testStr = testStr;
    }

    public void test() {
        System.out.println("test");
    }

}
```

```
@Aspect
public class AspectJTest {

    @Pointcut("execution(* com..*.test(..))")
    public void test() {
    }

    @Before("test()")
    public void beforeTest() {
        System.out.println("beforeTest");
    }

    @After("test()")
    public void afterTest() {
        System.out.println("afterTest");
    }

    @Around("test()")
    public Object aroundTest(ProceedingJoinPoint p) {
        System.out.println("before1");

        Object o=null;
        try{
            o = p.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        System.out.println("after1");
        return o;
    }

}
```

```
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
        					 http://www.springframework.org/schema/beans/spring-beans.xsd
         					  http://www.springframework.org/schema/aop
          					 http://www.springframework.org/schema/aop/spring-aop.xsd">


	<aop:aspectj-autoproxy />

	<bean id="test"
			class="com.zchuber.springsourcedeepdiving.aop.TestBean">
	</bean>

	<bean class="com.zchuber.springsourcedeepdiving.aop.AspectJTest" />


</beans>
```

测试用例：

```

    @Test
    public void testSimpleLoad(){
        ApplicationContext ac=new ClassPathXmlApplicationContext("beanFactoryTest-aop.xml");
        TestBean testBean = (TestBean) ac.getBean("test");
        testBean.test();
    }
```

运行结果：

```
before1
beforeTest
test
after1
afterTest
```

## 源码解析

### AopNamespaceHandler

```
public class AopNamespaceHandler extends NamespaceHandlerSupport {

	/**
	 * Register the {@link BeanDefinitionParser BeanDefinitionParsers} for the
	 * '{@code config}', '{@code spring-configured}', '{@code aspectj-autoproxy}'
	 * and '{@code scoped-proxy}' tags.
	 */
	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		//注册aspectj-autoproxy对应的BeanDefinitionParse：AspectJAutoProxyBeanDefinitionParser
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}

}
```

### 注册AnnotationAwareAspectJAutoProxyCreator

AspectJAutoProxyBeanDefinitionParser中主要的方法parse：

```
    @Nullable
    public BeanDefinition parse(Element element, ParserContext parserContext) {
    	//注册AspectJAnnotationAutoProxyCreator
        AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
        this.extendBeanDefinition(element, parserContext);
        return null;
    }
```

AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary：

```
	public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(
			ParserContext parserContext, Element sourceElement) {
		//注册AspectJAnnotationAutoProxyCreator
		BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(
				parserContext.getRegistry(), parserContext.extractSource(sourceElement));
		//处理元素中proxy-target-class、expose-proxy属性
		useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
		registerComponentIfNecessary(beanDefinition, parserContext);
	}
```

##### 注册或者升级AnnotationAwareAspectJAutoProxyCreator

AnnotationAwareAspectJAutoProxyCreator根据@Pointcut中的配置为符合条件的bean创建代理对象

AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary：

```
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
	}
```

AopConfigUtils.registerOrEscalateApcAsRequired：

```
private static BeanDefinition registerOrEscalateApcAsRequired(
      Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

	//如果registry已经注册internalAutoProxyCreator，对其进行设置BeanClassName
   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
      BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
      if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
         int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
         int requiredPriority = findPriorityForClass(cls);
         if (currentPriority < requiredPriority) {
            apcDefinition.setBeanClassName(cls.getName());
         }
      }
      return null;
   }
  
   //创建BeanDefinition，class为AnnotationAwareAspectJAutoProxy，注册到registry
   RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
   beanDefinition.setSource(source);
   beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
   beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
   return beanDefinition;
}
```

#### 处理proxy-target-class和expose-proxy属性

AopConfigUtils：

```
	private static void useClassProxyingIfNecessary(BeanDefinitionRegistry registry, @Nullable Element sourceElement) {
		if (sourceElement != null) {
			boolean proxyTargetClass = Boolean.parseBoolean(sourceElement.getAttribute(PROXY_TARGET_CLASS_ATTRIBUTE));
			//如果proxyTargetClass为true，设置internalAutoProxyCreator的BeanDefinition中proxyTargetClass为true
			if (proxyTargetClass) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
            //如果exposeProxy为true，设置internalAutoProxyCreator的BeanDefinition中exposeProxy为true
			boolean exposeProxy = Boolean.parseBoolean(sourceElement.getAttribute(EXPOSE_PROXY_ATTRIBUTE));
			if (exposeProxy) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}
	
	public static void forceAutoProxyCreatorToUseClassProxying(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);
		}
	}
    
	public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
		}
	}    
```

proxy-target-class：spring默认使用jdk的动态代理，如果想强制使用cglib的动态代理，令proxy-target-class为true

```
	<aop:aspectj-autoproxy proxy-target-class="true"/>
```

- JDK动态代理：代理对象必须是某个接口的实现类，在运行时动态的创建该接口的实现类作为代理对象
- cglib动态代理：动态创建代理类的子类作为代理对象

expose-proxy：在被代理对象的方法中内部调用不会进入通知方法。比如下面的this.b()就不会创建事务

```

public class AServiceImpl implements AService {

    @Transactional(propagation= Propagation.REQUIRED)
    public void a() {
        this.b();
    }

    @Transactional(propagation=Propagation.REQUIRES_NEW)
    public void b() {

    }
}
```

如果需要b()的事务生效，需要添加如下的配置：

```
	<aop:aspectj-autoproxy expose-proxy="true"/>
```

```
    @Transactional(propagation= Propagation.REQUIRED)
    public void a() {
        ((AService)AopContext.currentProxy()).b();
    }
```

## 创建AOP代理

AnnotationAwareAspectJAutoProxyCreator的继承体系如下：

![1654236938037](D:\学习\spring源码深度解析\assets\1654236938037.png)

父接口为BeanPostProcessor。重点关注postProcessAfterInitialization方法

AbstractAutoProxyCreator#postProcessAfterInitialization：

```
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				//创建代理对象
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

AbstractAutoProxyCreator#wrapIfNecessary：

```
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		//是基础设施的类：Pointcut、Advisor、Advice和AopInfrastructureBean，跳过aspect注解的bean
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		//获取该bean对象的advisor，如果advisor不为空则为该bean创建代理对象
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			//创建代理对象
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			//添加至缓存
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
	//获取该类所有的advisor
   List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
   //如果没空返回null
   if (advisors.isEmpty()) {
      return DO_NOT_PROXY;
   }
   return advisors.toArray();
}
```

AbstractAdvisorAutoProxyCreator#findEligibleAdvisors：

```
	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		//获得beanFactory中所有的advisor
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		//过滤处符合该bean的advisor
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		//供子类扩展
		extendAdvisors(eligibleAdvisors);
		//排序符合条件的advisor
		if (!eligibleAdvisors.isEmpty()) {
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
```

AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors：

```
protected List<Advisor> findCandidateAdvisors() {
   // 从beanFactory中获取所有的advisor的bean
   List<Advisor> advisors = super.findCandidateAdvisors();
   // 获得aspectJAdvisorsBuilder创建的advisor
   if (this.aspectJAdvisorsBuilder != null) {
      advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
   }
   //返回advisor
   return advisors;
}
```

BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors：

```
	public List<Advisor> buildAspectJAdvisors() {
		List<String> aspectNames = this.aspectBeanNames;

		if (aspectNames == null) {
			synchronized (this) {
				aspectNames = this.aspectBeanNames;
				if (aspectNames == null) {
					List<Advisor> advisors = new ArrayList<>();
					aspectNames = new ArrayList<>();
					//获取beanFactory中所有的bean
					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
					for (String beanName : beanNames) {
						//默认返回true
						if (!isEligibleBean(beanName)) {
							continue;
						}
						//获取bean对应的类型
						Class<?> beanType = this.beanFactory.getType(beanName);
						if (beanType == null) {
							continue;
						}
						//如果bean为aspect
						if (this.advisorFactory.isAspect(beanType)) {
							aspectNames.add(beanName);
							//创建AspectMetadata实例
							AspectMetadata amd = new AspectMetadata(beanType, beanName);
							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
								MetadataAwareAspectInstanceFactory factory =
										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
								//利用advisorFactory获得aspect类中的advisor
								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
								//添加advisor至缓存
								if (this.beanFactory.isSingleton(beanName)) {
									this.advisorsCache.put(beanName, classAdvisors);
								}
								else {
									this.aspectFactoryCache.put(beanName, factory);
								}
								advisors.addAll(classAdvisors);
							}
							else {
								// Per target or per this.
								if (this.beanFactory.isSingleton(beanName)) {
									throw new IllegalArgumentException("Bean with name '" + beanName +
											"' is a singleton, but aspect instantiation model is not singleton");
								}
								MetadataAwareAspectInstanceFactory factory =
										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
								this.aspectFactoryCache.put(beanName, factory);
								advisors.addAll(this.advisorFactory.getAdvisors(factory));
							}
						}
					}
					this.aspectBeanNames = aspectNames;
					return advisors;
				}
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
		List<Advisor> advisors = new ArrayList<>();
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}
```

ReflectiveAspectJAdvisorFactory#getAdvisors：

```
	public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
		//获取切面类
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		//获得切面名称
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
		//进行校验
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new ArrayList<>();
		//获得所有
		for (Method method : getAdvisorMethods(aspectClass)) {
			//获得切面类中的advisor
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}
```

ReflectiveAspectJAdvisorFactory#getAdvisorMethods：

```
	private List<Method> getAdvisorMethods(Class<?> aspectClass) {
		final List<Method> methods = new ArrayList<>();
		ReflectionUtils.doWithMethods(aspectClass, method -> {
			// 排查被Pointcut修饰的方法
			if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
				methods.add(method);
			}
		}, ReflectionUtils.USER_DECLARED_METHODS);
		methods.sort(METHOD_COMPARATOR);
		return methods;
	}
```

### 普通增强器的获取

```
	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) {

		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
		//获取切点
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}
		//根据切点信息生成advisor
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}
```

ReflectiveAspectJAdvisorFactory#getPointcut：

```
	private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}
		//获取advisor对应的pointcut
		AspectJExpressionPointcut ajexp =
				new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);	
		//如果是@Around("test()")，则pointcutExpression为test()
		ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
		if (this.beanFactory != null) {
			ajexp.setBeanFactory(this.beanFactory);
		}
		return ajexp;
	}
```

![1654240459107](D:\学习\spring源码深度解析\assets\1654240459107.png)

#### 创建Advisor对象

```
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
```

InstantiationModelAwarePointcutAdvisorImpl：

```
	public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
			Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
		//设置属性
		this.declaredPointcut = declaredPointcut;
		this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
		this.methodName = aspectJAdviceMethod.getName();
		this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
		this.aspectJAdviceMethod = aspectJAdviceMethod;
		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
		this.aspectInstanceFactory = aspectInstanceFactory;
		this.declarationOrder = declarationOrder;
		this.aspectName = aspectName;

		if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			// Static part of the pointcut is a lazy type.
			Pointcut preInstantiationPointcut = Pointcuts.union(
					aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

			// Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
			// If it's not a dynamic pointcut, it may be optimized out
			// by the Spring AOP infrastructure after the first evaluation.
			this.pointcut = new PerTargetInstantiationModelPointcut(
					this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
			this.lazy = true;
		}
		else {
			// A singleton aspect.
			this.pointcut = this.declaredPointcut;
			this.lazy = false;
			//创建advisor实例
			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
		}
	}
```

InstantiationModelAwarePointcutAdvisorImpl#instantiateAdvice：

```
	private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
		//从aspectJFactory中获得advisor
		Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
				this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
		return (advice != null ? advice : EMPTY_ADVICE);
	}
```

ReflectiveAspectJAdvisorFactory#getAdvice：

```
	public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		validate(candidateAspectClass);
		
		//获取方法上的aspectJ注解：包括@Pointcut、@Before、@After、@AfterReturning、@AfterThrowing、@Around
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}

		// If we get here, we know we have an AspectJ method.
		// Check that it's an AspectJ-annotated class
		if (!isAspect(candidateAspectClass)) {
			throw new AopConfigException("Advice must be declared inside an aspect type: " +
					"Offending method '" + candidateAdviceMethod + "' in class [" +
					candidateAspectClass.getName() + "]");
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Found AspectJ method: " + candidateAdviceMethod);
		}

		AbstractAspectJAdvice springAdvice;
		//根据不同的注解创建不同的advice
		switch (aspectJAnnotation.getAnnotationType()) {
			//注解为Pointcut创建null
			case AtPointcut:
				if (logger.isDebugEnabled()) {
					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
				}
				return null;
			//注解为Around，创建AspectJAroundAdvice
			case AtAround:
				springAdvice = new AspectJAroundAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtBefore:
			//注解为Before，创建AspectJMethodBeforeAdvice
				springAdvice = new AspectJMethodBeforeAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
              //注解为After，创建AspectJMethodAfterAdvice
			case AtAfter:
				springAdvice = new AspectJAfterAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			//注解为AfterReturning，创建AspectJMethodAfterReturningAdvice				
			case AtAfterReturning:
				springAdvice = new AspectJAfterReturningAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
					springAdvice.setReturningName(afterReturningAnnotation.returning());
				}
				break;
			//注解为AfterThrowing，创建AspectJMethodAfterThrowingAdvice						
			case AtAfterThrowing:
				springAdvice = new AspectJAfterThrowingAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
				}
				break;
			default:
				throw new UnsupportedOperationException(
						"Unsupported advice type on method: " + candidateAdviceMethod);
		}

		// Now to configure the advice...
		springAdvice.setAspectName(aspectName);
		springAdvice.setDeclarationOrder(declarationOrder);
		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
		if (argNames != null) {
			springAdvice.setArgumentNamesFromStringArray(argNames);
		}
		springAdvice.calculateArgumentBindings();

		return springAdvice;
	}
```

##### MethodBeforeAdviceInterceptor

MethodBeforeAdvice会被保证成MethodBeforeAdviceInterceptor使用

```
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {

	private final MethodBeforeAdvice advice;


	/**
	 * Create a new MethodBeforeAdviceInterceptor for the given advice.
	 * @param advice the MethodBeforeAdvice to wrap
	 */
	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}


	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		//先调用MethodBeforeAdvice的before方法
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		//调用业务方法
		return mi.proceed();
	}

}

```

AspectJMethodBeforeAdvice#before：

```
	public void before(Method method, Object[] args, @Nullable Object target) throws Throwable {
		invokeAdviceMethod(getJoinPointMatch(), null, null);
	}
```

AbstractAspectJAdvice#invokeAdviceMethod：

```
	protected Object invokeAdviceMethod(
			@Nullable JoinPointMatch jpMatch, @Nullable Object returnValue, @Nullable Throwable ex)
			throws Throwable {
			//调用通知对应的方法
		return invokeAdviceMethodWithGivenArgs(argBinding(getJoinPoint(), jpMatch, returnValue, ex));
	}

```

AbstractAspectJAdvice#invokeAdviceMethodWithGivenArgs：

```
	protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
		Object[] actualArgs = args;
		if (this.aspectJAdviceMethod.getParameterCount() == 0) {
			actualArgs = null;
		}
		try {
			ReflectionUtils.makeAccessible(this.aspectJAdviceMethod);
			// 利用反射调用切面类中advice对应的方法
			return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);
		}
		catch (IllegalArgumentException ex) {
			throw new AopInvocationException("Mismatch on arguments to advice method [" +
					this.aspectJAdviceMethod + "]; pointcut expression [" +
					this.pointcut.getPointcutExpression() + "]", ex);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
```

##### AspectJAfterAdvice

AspectJAfterAdvice不需要向AspectJMethodBeforeAdvice那样包装在MethodBeforeAdviceInterceptor中。

```
public class AspectJAfterAdvice extends AbstractAspectJAdvice
		implements MethodInterceptor, AfterAdvice, Serializable {

	public AspectJAfterAdvice(
			Method aspectJBeforeAdviceMethod, AspectJExpressionPointcut pointcut, AspectInstanceFactory aif) {
		//advice对应的方法，以及pointcut
		super(aspectJBeforeAdviceMethod, pointcut, aif);
	}


	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			//调用业务逻辑方法
			return mi.proceed();
		}
		finally {
			//调用advice对应的方法
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}

	@Override
	public boolean isBeforeAdvice() {
		return false;
	}

	@Override
	public boolean isAfterAdvice() {
		return true;
	}

}
```

### 寻找匹配的advisor

```
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
```

AbstractAdvisorAutoProxyCreator#findAdvisorsThatCanApply：

```
	protected List<Advisor> findAdvisorsThatCanApply(
			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
		try {
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
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new ArrayList<>();
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
			//如果advisor中的pointcut匹配该类中的方法，将advisor添加至eligibleAdvisors
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
		else if (advisor instanceof PointcutAdvisor) {
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			// 没有pointcut默认匹配
			return true;
		}
	}
```

AopUtils#canApply：

```
	public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		Assert.notNull(pc, "Pointcut must not be null");
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}

		MethodMatcher methodMatcher = pc.getMethodMatcher();
		if (methodMatcher == MethodMatcher.TRUE) {
			// No need to iterate the methods if we're matching any method anyway...
			return true;
		}

		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
		}

		Set<Class<?>> classes = new LinkedHashSet<>();
		if (!Proxy.isProxyClass(targetClass)) {
			classes.add(ClassUtils.getUserClass(targetClass));
		}
		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

		for (Class<?> clazz : classes) {
			//获得该类对象中所有的方法
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
			for (Method method : methods) {
				if (introductionAwareMethodMatcher != null ?
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
						//利用methodMatcher进行匹配targetClass中的method方法
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}

```

MethodMatcher的子类实现，包括处理AspectJ表达式的AspectJExpressionPointcut

![1654243440282](D:\学习\spring源码深度解析\assets\1654243440282.png)

### 创建代理

AbstractAutoProxyCreator#createProxy：

```
	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		//拷贝proxyConfig的配置到proxyFactory
		proxyFactory.copyFrom(this);
		
		//设置proxyTargetClass
		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		//将拦截器封装为advisor
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		//添加advisors到proxyFactory
		proxyFactory.addAdvisors(advisors);
		//设置被代理的bean
		proxyFactory.setTargetSource(targetSource);
		//供子类自定义
		customizeProxyFactory(proxyFactory);
		
		//设置冻住proxyFactory的配置，不再允许被修改
		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		//创建代理类
		return proxyFactory.getProxy(getProxyClassLoader());
	}
```

#### 将拦截器封装为Advisor

AbstractAutoProxyCreator#buildAdvisors：

```
	protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
		// 获得AbstractAutoProxyCreator类中advisor
		Advisor[] commonInterceptors = resolveInterceptorNames();

		List<Object> allInterceptors = new ArrayList<>();
		if (specificInterceptors != null) {
			allInterceptors.addAll(Arrays.asList(specificInterceptors));
			if (commonInterceptors.length > 0) {
				if (this.applyCommonInterceptorsFirst) {
					allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
				}
				else {
					allInterceptors.addAll(Arrays.asList(commonInterceptors));
				}
			}
		}
		if (logger.isTraceEnabled()) {
			int nrOfCommonInterceptors = commonInterceptors.length;
			int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
			logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
					" common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
		}

		Advisor[] advisors = new Advisor[allInterceptors.size()];
		for (int i = 0; i < allInterceptors.size(); i++) {
			//在wrap中根据拦截器获取advisor
			advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
		}
		return advisors;
	}
```

DefaultAdvisorAdapterRegistry#wrap：

```
	public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
		//如果为advisor的子类返回。
		if (adviceObject instanceof Advisor) {
			return (Advisor) adviceObject;
		}
		if (!(adviceObject instanceof Advice)) {
			throw new UnknownAdviceTypeException(adviceObject);
		}
		Advice advice = (Advice) adviceObject;
		//如果advice为MethodInterceptor的实现类，创建DefaultPointcutAdvisor
		if (advice instanceof MethodInterceptor) {
			// So well-known it doesn't even need an adapter.
			return new DefaultPointcutAdvisor(advice);
		}
		for (AdvisorAdapter adapter : this.adapters) {
			// 如果adapter中可以处理该advice，创建创建DefaultPointcutAdvisor
			if (adapter.supportsAdvice(advice)) {
				return new DefaultPointcutAdvisor(advice);
			}
		}
		throw new UnknownAdviceTypeException(advice);
	}
```

adapters默认为MethodBeforeAdviceAdapter、AfterReturningAdviceAdapter和ThrowsAdviceAdapter

```

	private final List<AdvisorAdapter> adapters = new ArrayList<>(3);


	/**
	 * Create a new DefaultAdvisorAdapterRegistry, registering well-known adapters.
	 */
	public DefaultAdvisorAdapterRegistry() {
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}
```

#### 创建代理

ProxyFactory#getProxy：

```
	public Object getProxy(@Nullable ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}
```

##### 创建代理

DefaultAopProxyFactory#createAopProxy：

```
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		//是否进行优化、ProxyTargetClass是否为true，被代理类是否有接口
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			//使用cgblib创建动态代理
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			//使用jdk创建动态代理
			return new JdkDynamicAopProxy(config);
		}
	}
```

可见，spring具体使用cglib换是jdk的原则是：

- optimize是否为true：是否需要优化，只有cglib可以进行优化。该值默认为false。
- proxyTargetClass是否为true：默认为false
- 如果用户不特殊设置会根据被代理类是否有接口决定使用cglib换是jdk：如果被代理类没有接口使用jdk，否则使用使用cglib。

jdk动态代理和cglib动态代理的区别？

- 使用jdk创建代理需要被代理类由接口
- cglib使用继承的方式可以对类创建该类的子类为动态代理，需要被代理类的方法不能为final

#### 获取代理

##### 使用示例

```
public interface UserService {

    void add();

}
```

```
public class UserServiceImpl implements UserService {

    public void add() {
        System.out.println("------------------------add-------------------");
    }

}
```

```
public class MyInnovationHandler implements InvocationHandler {


    private Object target;

    public MyInnovationHandler( Object target ) {
        super();
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        System.out.println("------------------------------before-------------------");

        Object result = method.invoke(target, args);

        System.out.println("------------------------------after-------------------");
        return result;
    }

    public Object getProxy() {
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), target.getClass().getInterfaces(),
                this);
    }

}
```

测试用例：

```
    @Test
    public void testProxy(){
        UserService userService = new UserServiceImpl();

        MyInnovationHandler inh = new MyInnovationHandler(userService);

        UserService proxy= (UserService) inh.getProxy();
        proxy.add();
    }
```

运行结果：

```
------------------------------before-------------------
------------------------add-------------------
------------------------------after-------------------
```

主要有三点：

- 被代理对象通过构造函数传入
- invoke方法定义增强逻辑
- getProxy获得代理对象

#### JdkDynamicAopProxy

JdkDynamicAopProxy#getProxy：

```
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}
	
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isTraceEnabled()) {
			logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		//创建代理对象
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}	
```

![1654253274903](D:\学习\spring源码深度解析\assets\1654253274903.png)

JDKDynamicAopProxy实现了InvocationHandler，当代理方法被调用时会进入invoke方法中：

```
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Object target = null;

		try {
			//处理equals方法
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
			//处理hashcode方法
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -> dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;
			//如果exposeProxy为true，将代理对象设置到AopContext中
			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			//获取目标对象
			target = targetSource.getTarget();
			//获取目标类
			Class<?> targetClass = (target != null ? target.getClass() : null);

			// 获取目标对象该方法的advisor
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// 如果拦截链为空，直接调用目标对象的方法
			if (chain.isEmpty()) {				
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				//创建MethodInvocation对象，封装拦截链
				MethodInvocation invocation =
						new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// 拦截链开始拦截
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target &&
					returnType != Object.class && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}

```

#### 如有必要将advice封装为MethodInterceptor

AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice：

```
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
		MethodCacheKey cacheKey = new MethodCacheKey(method);
		List<Object> cached = this.methodCache.get(cacheKey);
		if (cached == null) {
			cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);
			this.methodCache.put(cacheKey, cached);
		}
		return cached;
	}

```

经过该方法会将Advice的子类中非MethodInterceptor实现类比如AspectJAfterReturningAdvice、AspectJMethodBeforeAdvice。AspectJMethodBeforeAdvice会被封装在MethodBeforeAdviceInterceptor，AspectJAfterReturningAdvice会被封装在AfterReturningAdvice Interceptor。

AspectJAfterReturningAdvice的继承体系

![1654254018767](D:\学习\spring源码深度解析\assets\1654254035264.png)

AspectJMethodBeforeAdvice的继承体系

![1654254236297](D:\学习\spring源码深度解析\assets\1654254236297.png)



而AspectAfterAdvice、AspectAroundAdvice和AspectAfterThrowingAdvice本身已经是MethodInterceptor接口，不需要额外的处理。

AspectJAfterAdvice的继承体系，实现了MethodInterceptor接口

![1654253944953](D:\学习\spring源码深度解析\assets\1654253944953.png)

AspectJAfterThrowingAdvice的继承体系，实现了MethodInterceptor接口

![1654254089069](D:\学习\spring源码深度解析\assets\1654254089069.png)

AspectJAroundAdvice的继承体系，实现了MethodInterceptor接口

![1654254159321](D:\学习\spring源码深度解析\assets\1654254159321.png)

##### 调用链debug

![1654254767424](D:\学习\spring源码深度解析\assets\1654254767424.png)

通过debug可以验证上面的分析

#### 拦截链的调用过程

拦截链被封装到ReflectiveMethodInvocation中，主要的方法为proceed。

ReflectiveMethodInvocation#proceed：

```
	public Object proceed() throws Throwable {
		//最后一个拦截器处理完毕通过反射调用目标对象的方法
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}
		
		//获得下一个拦截器
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			//调用拦截器的invoke方法，注意会将methodInvocation对象传递给拦截器
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}

```



#### 拦截器中的MethodInterceptor的顺序

拦截链的顺序和debug中的保持一致，先通过advice的类型排序，类型相同再通过advice对应的方法名排序。

![1654258180301](D:\学习\spring源码深度解析\assets\1654258180301.png)

不同advice的优先级如下：

1. AfterThrowing
2. AfterReturning
3. After
4. Around
5. Before



ReflectiveAspectJAdvisorFactory#getAdvisorMethods：

```
	private List<Method> getAdvisorMethods(Class<?> aspectClass) {
		final List<Method> methods = new ArrayList<>();
		ReflectionUtils.doWithMethods(aspectClass, method -> {
			// Exclude pointcuts
			if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
				methods.add(method);
			}
		}, ReflectionUtils.USER_DECLARED_METHODS);
		methods.sort(METHOD_COMPARATOR);
		return methods;
	}
```

切面类中的方法安装METHOD_COMPARATOR进行排序。

METHOD_COMPARATOR的定义如下：

```
public class ReflectiveAspectJAdvisorFactory extends AbstractAspectJAdvisorFactory implements Serializable {

	private static final Comparator<Method> METHOD_COMPARATOR;

	static {
		//不同advice的优先级从小到大的顺序：
		Comparator<Method> adviceKindComparator = new ConvertingComparator<>(
				new InstanceComparator<>(
						Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class),
				(Converter<Method, Annotation>) method -> {
					AspectJAnnotation<?> annotation =
						AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(method);
					return (annotation != null ? annotation.getAnnotation() : null);
				});
		Comparator<Method> methodNameComparator = new ConvertingComparator<>(Method::getName);
		//先按照类型然后按照通知对应的方法名
		METHOD_COMPARATOR = adviceKindComparator.thenComparing(methodNameComparator);
	}
}	
```



#### 不同MethodInterceptor中invoke方法的定义

##### advisor第一次排序：Advice的优先级从小到大

ReflectiveAspectJAdvisorFactory#getAdvisorMethods：

```
	private List<Method> getAdvisorMethods(Class<?> aspectClass) {
		final List<Method> methods = new ArrayList<>();
		ReflectionUtils.doWithMethods(aspectClass, method -> {
			// 排除被Pointcut修饰的方法
			if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
				methods.add(method);
			}
		}, ReflectionUtils.USER_DECLARED_METHODS);
		//方法按照比较其进行排序
		methods.sort(METHOD_COMPARATOR);
		return methods;
	}
```

比较器的定义如下，ReflectiveAspectJAdvisorFactory：

```
public class ReflectiveAspectJAdvisorFactory extends AbstractAspectJAdvisorFactory implements Serializable {

	private static final Comparator<Method> METHOD_COMPARATOR;

	static {
		//优先级从小到大：Around、Before、After、AfterReturning、AfterThrowing
		Comparator<Method> adviceKindComparator = new ConvertingComparator<>(
				new InstanceComparator<>(
						Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class),
				(Converter<Method, Annotation>) method -> {
					AspectJAnnotation<?> annotation =
						AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(method);
					return (annotation != null ? annotation.getAnnotation() : null);
				});
	    //按照方法名称比较
		Comparator<Method> methodNameComparator = new ConvertingComparator<>(Method::getName);
		//先是按照advice进行排序，advice相等的情况按照方法名称排序
		METHOD_COMPARATOR = adviceKindComparator.thenComparing(methodNameComparator);
	}
}	
```

通过方法创建Advisor的过程是有序的：顺序是先按照advice的优先级，然后按照方法名称排序。

advice的优先级从小到大：

1. Around
2. Before
3. After
4. AfterReturning
5. AfterThrowing

##### debug验证

![1654303311152](D:\学习\spring源码深度解析\assets\1654303311152.png)

此时advisor的顺序是按照advice的顺序排列的。

##### 第二次排序：

AbstractAdvisorAutoProxyCreator#findEligibleAdvisors：

```
	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		if (!eligibleAdvisors.isEmpty()) {
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
```

AspectJAwareAdvisorAutoProxyCreator#sortAdvisors：

```
protected List<Advisor> sortAdvisors(List<Advisor> advisors) {
	//对advisor进行排序
   List<PartiallyComparableAdvisorHolder> sorted = PartialOrder.sort(partiallyComparableAdvisors);
}
```

##### debug展示

![1654307287251](D:\学习\spring源码深度解析\assets\1654307287251.png)

排序后的advice为：

- AfterThrowing
- AfterReturning
- After
- Around
- Before

#### 拦截链的调用

拦截链的调用顺序：

1. AspectJAfterThrowingAdvice
2. AfterReturningAdviceInterceptor
3. AspectJAfterAdvice
4. AspectJAroundAdvice
5. MethodBeforeAdviceInterceptor

##### AspectJAfterThrowingAdvice#invoke：

	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			//进入methodInvocation的proceed方法。由于拦截链最后一个已经执行，此时调用目标对象的方法
			return mi.proceed();
		}
		catch (Throwable ex) {
			//只有抛出异常时才会执行AfterThrowing通知对应的方法。没有异常则不会执行
			if (shouldInvokeOnThrowing(ex)) {
				invokeAdviceMethod(getJoinPointMatch(), null, ex);
			}
			throw ex;
		}
	}
##### AfterReturningAdviceInterceptor#invoke：

```
	public Object invoke(MethodInvocation mi) throws Throwable {
		//调用下一个拦截器的invoke方法
		Object retVal = mi.proceed();
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}
```

##### AspectJAfterAdvice#invoke：

```
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			//调用下一个拦截器的invoke方法
			return mi.proceed();
		}
		finally {
			//不管是否抛出异常都会执行
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
```

##### AspectJAroundAdvice#invoke：

```
	public Object invoke(MethodInvocation mi) throws Throwable {
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
		ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
		JoinPointMatch jpm = getJoinPointMatch(pmi);
		//调用around通知的方法，实例代码中aroundTest方法会被调用。pjp中封装了methodInvocation
		return invokeAdviceMethod(pjp, jpm, null, null);
	}

```

```
    @Around("test()")
    public Object aroundTest(ProceedingJoinPoint p) {
        System.out.println("before1");

        Object o=null;
        try{
            o = p.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        System.out.println("after1");
        return o;
    }
```

结果输出：

```
before1
```

##### MethodBeforeAdviceInterceptor#invoke

```
	public Object invoke(MethodInvocation mi) throws Throwable {
		//调用before通知的方法
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		//调用methodInvocation方法中，此时拦截链已经执行完毕，执行切入点方法
		return mi.proceed();
	}
```

MethodBeforeAdviceInterceptor为最后一个拦截器，进入methodInvocation的proceed方法中会调用invokeJoinpoint方法。

```
	public Object proceed() throws Throwable {
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}
 }		
```

切入点的方法如下：

        public void test() {
            System.out.println("------------------------add-------------------");
        }
结果输出：

```
before1
beforeTest
------------------------add-------------------
```

切入点方法执行完毕返回，此时methodInvocation.proceed方法结束。

##### MethodBeforeAdviceInterceptor#invoke

	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		//methodInvocation.proceed方法返回，invoke方法结束，返回到上一个拦截链
		return mi.proceed();
	}

##### AspectJAroundAdvice#invoke：

```
	public Object invoke(MethodInvocation mi) throws Throwable {
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
		ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
		JoinPointMatch jpm = getJoinPointMatch(pmi);
		//调用around通知的方法结束则invoke方法结束
		return invokeAdviceMethod(pjp, jpm, null, null);
	}

```

```
    @Around("test()")
    public Object aroundTest(ProceedingJoinPoint p) {
        System.out.println("before1");

        Object o=null;
        try{
        	//MethodBeforeAdviceInterceptor的invoke运行结束后，p.proceed方法返回
            o = p.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        //输出after1
        System.out.println("after1");
        //方法结束后返回到上一个拦截链AspectJAfterAdvice
        return o;
    }
```

结果输出：

```
before1
beforeTest
------------------------add-------------------
after1
```

##### AspectJAfterAdvice#invoke：

```
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			//methodInvocation.procee方法结束后调用finally中的方法
			return mi.proceed();
		}
		finally {
			//调用after方法修饰的方法，调用结束后返回到上一个拦截链AfterReturningInterceptor
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
```
```
    @After("test()")
    public void afterTest() {
    	//输出afterTest1
        System.out.println("afterTest");
    }
```

结果输出：

```
before1
beforeTest
------------------------add-------------------
after1
afterTest
```

##### AfterReturningAdviceInterceptor#invoke：

```
	public Object invoke(MethodInvocation mi) throws Throwable {
		//methodInvocation.proceed方法调用结束后获得返回值
		Object retVal = mi.proceed();
		//调用AfterReturning修饰的方法
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		//返回到上一个拦截链AspectJAfterThrowingAdvice
		return retVal;
	}
```
```
@AfterReturning("test()")
public void afteReturningTest() {
	//输出afteReturningTest
    System.out.println("afteReturningTest");
}
```

结果输出：

```
before1
beforeTest
------------------------add-------------------
after1
afterTest
afteReturningTest

```

##### AspectJAfterThrowingAdvice#invoke：

	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			//methodInvocation.proceed方法调用结束返回，如果调用的过程抛出异常进入catch中处理
			return mi.proceed();
		}
		catch (Throwable ex) {
			//调用的过程出现异常会执行AfterThrowing修饰的方法，由于该demo中没有抛出异常所以不会进入catch
			if (shouldInvokeOnThrowing(ex)) {
				invokeAdviceMethod(getJoinPointMatch(), null, ex);
			}
			throw ex;
		}
	}
```
    @AfterThrowing("test()")
    public void afterThrowingTest() {
    	//抛出异常会输出afterThrowingTest
        System.out.println("afterThrowingTest");
    }
```

结果输出：

```
before1
beforeTest
------------------------add-------------------
after1
afterTest
afteReturningTest
```

### CGLIB创建AOP原理

#### cglib使用示例

```
public class EnhancerDemo {

    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(EnhancerDemo.class);
        enhancer.setCallback(new MethodInterceportImpl());

        EnhancerDemo demo=(EnhancerDemo)enhancer.create();
        demo.test();
        System.out.println(demo);
    }


    public void  test() {
        System.out.println("test");
    }


    private static class MethodInterceportImpl implements MethodInterceptor {

        public Object intercept(Object obj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
            System.err.println("Before invoke "+method);
            Object result = methodProxy.invokeSuper(obj, args);
            System.out.println("After Advice "+method);
            return result;
        }
    }
}
```

结果输出：

```
Before invoke public void com.zchuber.springsourcedeepdiving.aop.EnhancerDemo.test()
test
After Advice public void com.zchuber.springsourcedeepdiving.aop.EnhancerDemo.test()
After Advice public native int java.lang.Object.hashCode()
After Advice public java.lang.String java.lang.Object.toString()
com.zchuber.springsourcedeepdiving.aop.EnhancerDemo$$EnhancerByCGLIB$$b46dfae1@73c6c3b2
Before invoke public java.lang.String java.lang.Object.toString()
Before invoke public native int java.lang.Object.hashCode()
```

主要的类：

- Enhancer
- MethodInterceptor：使用cglib创建的代理的方法调用会进入intercept方法，在该方法中定义增强逻辑

#### CglibAopProxy

CglibAopProxy#getProxy：

```
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isTraceEnabled()) {
			logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
		}

		try {
			Class<?> rootClass = this.advised.getTargetClass();
			Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

			Class<?> proxySuperClass = rootClass;
			if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
				proxySuperClass = rootClass.getSuperclass();
				Class<?>[] additionalInterfaces = rootClass.getInterfaces();
				for (Class<?> additionalInterface : additionalInterfaces) {
					this.advised.addInterface(additionalInterface);
				}
			}

			// Validate the class, writing log messages as necessary.
			validateClassIfNecessary(proxySuperClass, classLoader);

			//创建enhancer对象
			Enhancer enhancer = createEnhancer();
			if (classLoader != null) {
				enhancer.setClassLoader(classLoader);
				if (classLoader instanceof SmartClassLoader &&
						((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
					enhancer.setUseCache(false);
				}
			}
			//设置enhancer
			enhancer.setSuperclass(proxySuperClass);
			enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));
			//获得各种Callback，在Callback中定义拦截逻辑
			Callback[] callbacks = getCallbacks(rootClass);
			Class<?>[] types = new Class<?>[callbacks.length];
			for (int x = 0; x < types.length; x++) {
				types[x] = callbacks[x].getClass();
			}
			// fixedInterceptorMap only populated at this point, after getCallbacks call above
			enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
			enhancer.setCallbackTypes(types);

			//利用enhancer.create()创建代理对象
			return createProxyClassAndInstance(enhancer, callbacks);
		}
		catch (CodeGenerationException | IllegalArgumentException ex) {
			throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
					": Common causes of this problem include using a final class or a non-visible class",
					ex);
		}
		catch (Throwable ex) {
			// TargetSource.getTarget() failed
			throw new AopConfigException("Unexpected AOP exception", ex);
		}
	}
```

CglibAopProxy#getCallbacks：

```
	private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
		// Parameters used for optimization choices...
		boolean exposeProxy = this.advised.isExposeProxy();
		boolean isFrozen = this.advised.isFrozen();
		boolean isStatic = this.advised.getTargetSource().isStatic();

		//创建aop的拦截器，最重要的callback。
		Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

		// Choose a "straight to target" interceptor. (used for calls that are
		// unadvised but can return this). May be required to expose the proxy.
		Callback targetInterceptor;
		if (exposeProxy) {
			targetInterceptor = (isStatic ?
					new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
		}
		else {
			targetInterceptor = (isStatic ?
					new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
		}

		// Choose a "direct to target" dispatcher (used for
		// unadvised calls to static targets that cannot return this).
		Callback targetDispatcher = (isStatic ?
				new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());

		Callback[] mainCallbacks = new Callback[] {
				aopInterceptor,  // for normal advice
				targetInterceptor,  // invoke target without considering advice, if optimized
				new SerializableNoOp(),  // no override for methods mapped to this
				targetDispatcher, this.advisedDispatcher,
				new EqualsInterceptor(this.advised),
				new HashCodeInterceptor(this.advised)
		};

		Callback[] callbacks;

		// If the target is a static one and the advice chain is frozen,
		// then we can make some optimizations by sending the AOP calls
		// direct to the target using the fixed chain for that method.
		if (isStatic && isFrozen) {
			Method[] methods = rootClass.getMethods();
			Callback[] fixedCallbacks = new Callback[methods.length];
			this.fixedInterceptorMap = new HashMap<>(methods.length);

			// TODO: small memory optimization here (can skip creation for methods with no advice)
			for (int x = 0; x < methods.length; x++) {
				Method method = methods[x];
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, rootClass);
				fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
						chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
				this.fixedInterceptorMap.put(method, x);
			}

			// Now copy both the callbacks from mainCallbacks
			// and fixedCallbacks into the callbacks array.
			callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
			System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
			System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
			this.fixedInterceptorOffset = mainCallbacks.length;
		}
		else {
			callbacks = mainCallbacks;
		}
		return callbacks;
	}
```

DynamicAdvisedInterceptor实现MethodInterceptor接口，重点方法intercept()：

```
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				// Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
				target = targetSource.getTarget();
				Class<?> targetClass = (target != null ? target.getClass() : null);
				//获得拦截链，实现和jdk的相同
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				//如果拦截链为空，调用目标方法
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// 否则，创建CglibMethodInvocation，调用proceed方法。
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null && !targetSource.isStatic()) {
					targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}

```

CglibMethodInvocation#proceed：

```
		public Object proceed() throws Throwable {
			try {
				//调用父类ReflectiveMethodInvocation的proceed方法，
				return super.proceed();
			}
			catch (RuntimeException ex) {
				throw ex;
			}
			catch (Exception ex) {
				if (ReflectionUtils.declaresException(getMethod(), ex.getClass())) {
					throw ex;
				}
				else {
					throw new UndeclaredThrowableException(ex);
				}
			}
		}
```

cglib除了创建代理的方式不同，对通知的处理基本和jdk的方式保持一致。cglib通过DynamicAdvisedInterceptor封装了处理通知的方式，调用代理类的方法会进入该类的intercept方法中进行增强

