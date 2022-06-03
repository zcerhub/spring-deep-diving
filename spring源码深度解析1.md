# 容器的基本实现

## 容器的基本用法

### beans包的层级结构

### 核心类介绍

#### DefaultListableBeanFactory

DefaultListableBeanFactory是比较纯粹的ioc工厂的实现类， 其余的ioc在其上进行扩展，比如XMLBeanFactory，内部通过XmlBeanDefinitionReader从xml读取bean配置然后注册到DefaultListableBeanFactory中。

![1653567617207](D:\学习\spring源码深度解析\assets\1653567617207.png)

- AliasRegistry：定义对alias的增删查接口
- SimpleAliasRegistry：AliasRegistry的实现类。通过map维护beanName和alias的关系
- SingletonBeanRegistry：定义了对单例bean的注册
- BeanFactory：ioc工厂接口，获取bean对象
- DefaultSingletonBeanRegistry：注册单例bean
- HierarchicalBeanFactory：通过parentBeanFactory实现ioc容器的继承，即父子容器
- BeanDefinitionRegistry：对BeanDefinition增删查的接口
- FactoryBeanRegistrySupport：在DefaultSingletonBeanRegistry的基础上增加了对FactoryBean处理的功能
- ConfigurableBeanFactory：提供可配置的beanFactory
- ListableBeanFactory：获得不同的bean实例
- AutowireCapableBeanFactory：提供创建、初始化、自动注入和销毁bean的功能

#### XmlBeanDefinitionReader

通过XmlBeanDefinitionReader读取定义在xml中的bean。经历的过程由：加载、解析和注册。涉及的类有：

- ResourceLoader：根据资源路径获得resource对象，
- BeanDefinitionReader：从resource中读取并将其解析为BeanDefinition
- DocumentLoader：从resource中获取Document对象
- EnvironmentCapable：定义获取Environment方法
- BeanDefinitionDocumentReader：从Document中解析出BeanDefinition对象
- BeanDefinitionParseDelegate：定义解析Document中element的方式

由此XmlBeanDefinitionReader的工作流程如下：

- 利用父类AbstractBeanDefinitionReader中的ResourceLoader根据资源路径加载resource
- 利用DocumentLoader将Resource转换为Document对象
- 利用BeanDefinitionDocumentReader的实现类DefaultBeanDefinitionDocumentReader类对Document进行解析，具体的element交给BeanDefinitionParseDelegate完成解析

## 容器的基础XmlBeanFactory

##### demo展示

```
public class MyTestBean {

    private String testStr="testStr";

    public String getTestStr() {
        return testStr;
    }

    public void setTestStr(String testStr) {
        this.testStr = testStr;
    }

}
```

beanFactoryTest.xml定义如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
	

	<!-- Define the Bean with Dependency Injection -->
	<bean id="myTestBean"
			class="com.zchuber.springsourcedeepdiving.ioc.MyTestBean">
	</bean>

</beans>
```

测试类如下：

```
public class TestMyTestBean {

    @Test
    public void testSimpleLoad(){
        BeanFactory bf=new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));
        MyTestBean testBean = (MyTestBean) bf.getBean("myTestBean");
        assertEquals("testStr", testBean.getTestStr());
    }

}
```

整个步骤如下：

- 配置文件封装为Resource对象

  ```
  new ClassPathResource("beanFactoryTest.xml")
  ```

  Spring使用Resource接口封装不同来源的配置，Resource定义如下：

  ```
  public interface Resource extends InputStreamSource {
  
  	boolean exists();
  	
  	default boolean isReadable() {
  		return exists();
  	}
      
  	default boolean isOpen() {
  		return false;
  	}
      
  	default boolean isFile() {
  		return false;
  	}
      
  }    
  ```

  其继承体系如下：

  ![1653744545668](D:\学习\spring源码深度解析\assets\1653744545668.png)

配置来自文件系统（FileSystemResource）、类路径（ClassPathResource）、数组（ByteArrayResource）。

Resource的常见使用方式如下：

```
        Resource resource = new ClassPathResource("beanFactoryTest.xml");
        InputStream ins = resource.getInputStream();
```

从Resource中获得InputStream对象后就可以统一对InputStream进行处理，从中获得bean的定义。其getInputStream实现方式也比较简单。

ClassPathResource：

```
		if (this.clazz != null) {
			is = this.clazz.getResourceAsStream(this.path);
		}
		else if (this.classLoader != null) {
			is = this.classLoader.getResourceAsStream(this.path);
		}
		else {
			is = ClassLoader.getSystemResourceAsStream(this.path);
		}
```

主要从class或者ClassLoader值获得inputstream对象。

接下来的解析部分交给XmlBeanDefinitionReader。

```
	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}
	
	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		//调用父类的构造函数
		super(parentBeanFactory);
		//reader处理resource从而获得BeanDefinition
		this.reader.loadBeanDefinitions(resource);
	}	
```

super(parentBeanFactory)最终会调用AbstractAutowireCapableBeanFactory#AbstractAutowireCapableBeanFactory方法：

```
	public AbstractAutowireCapableBeanFactory() {
		super();
		ignoreDependencyInterface(BeanNameAware.class);
		ignoreDependencyInterface(BeanFactoryAware.class);
		ignoreDependencyInterface(BeanClassLoaderAware.class);
	}
```

ignoreDependencyInterface用于在自动装配阶段忽略指定接口。主要是属性注入会交给XXXAware接口中的set方法。比如BeanNameAware中的setBeanName、BeanFactoryAware中的setBeanFactory。因此需要忽略对这些属性的自动装配。

### 加载Bean

主要流程：

- 封装资源文件

  ```
  	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
  		return loadBeanDefinitions(new EncodedResource(resource));
  	}
  ```

  EncodedResource中包含资源文件的编码信息

  EncodedResource的定义如下：

  ```
  	private final Resource resource;
  
  	//保存编码信息
  	@Nullable
  	private final String encoding;
  
  	//保存字符集
  	@Nullable
  	private final Charset charset;	
  	
  	public EncodedResource(Resource resource) {
  	  //设置默认编码和字符集为null
  		this(resource, null, null);
  	}
  	
  	private EncodedResource(Resource resource, @Nullable String encoding, @Nullable Charset charset) {
  		super();
  		Assert.notNull(resource, "Resource must not be null");
  		this.resource = resource;
  		this.encoding = encoding;
  		this.charset = charset;
  	}	
  	
  }	
  ```

- 从resource中获取输入流

  ```
  			//从encodedResource中获取输入流对象
  			InputStream inputStream = encodedResource.getResource().getInputStream();
  			try {
  				//创建inputSource，xml进行sax解析时需要该对象
  				InputSource inputSource = new InputSource(inputStream);
  				//处理编码信息，其实是inputSource使用
  				if (encodedResource.getEncoding() != null) {
  					inputSource.setEncoding(encodedResource.getEncoding());
  				}
  				//调用实际的从resource中记载beanDefinitions
  				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
  			}
  			finally {
  				inputStream.close();
  			}
  ```

- 加载xml文件获得创建Document对象

  ```
  			Document doc = doLoadDocument(inputSource, resource);
  			//从document中解析出BeanDefinition对象
  			int count = registerBeanDefinitions(doc, resource);
  ```

  - 获取Document

    XmlBeanDefinitionReader#doLoadDocument：

    ```
    	protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
    		//调用documentLoader的loadDocument进行加载
    		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
    				getValidationModeForResource(resource), isNamespaceAware());
    	}
    ```

    XmlBeanDefinitionReader将加载的任务交给DocumentLoader接口完成，DocumentLoader接口的实现类为DefaultDocumentLoader。DefaultDocumentLoader#loadDocument：

    ```
    	public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
    			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
    		//创建DocumentBuilderFactory对象
    		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
    		//利用DocumentBuilderFactory创建DocumentBuilder对象
    		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
    		//利用DocumentBuilder对象从inputSource解析出Document
    		return builder.parse(inputSource);
    	}
    ```

- 解析及注册BeanDefinitions

  ```
  	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
  	    //创建BeanDefinitionDocumentReader对象，实际的实现类为DefaultBeanDefinitionDocumentReader
  		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
  		//获得BeanDefinitionRegistry中已经注册的BeanDefinition的个数
  		int countBefore = getRegistry().getBeanDefinitionCount();
  		//利用documentReader注册BeanDefinition
  		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
  		return getRegistry().getBeanDefinitionCount() - countBefore;
  	}
  ```

  BeanDefinitionRegistry的实现类为XMLBeanFactory，可以从XmlBeanFactory中创建reader时看出。

  ```
  public class XmlBeanFactory extends DefaultListableBeanFactory {	
  	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);	
  	...
  }	
  ```

  实际的解析过程在DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions：

  ```
  	protected void doRegisterBeanDefinitions(Element root) {
  		BeanDefinitionParserDelegate parent = this.delegate;
  		this.delegate = createDelegate(getReaderContext(), root, parent);
  
  		//处理profile属性
  		if (this.delegate.isDefaultNamespace(root)) {
  			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
  			if (StringUtils.hasText(profileSpec)) {
  				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
  						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
  				// We cannot use Profiles.of(...) since profile expressions are not supported
  				// in XML config. See SPR-12458 for details.
  				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
  					if (logger.isDebugEnabled()) {
  						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
  								"] not matching: " + getReaderContext().getResource());
  					}
  					return;
  				}
  			}
  		}
  
  		//扩展方法，解析xml前处理
  		preProcessXml(root);
  		//解析root元素
  		parseBeanDefinitions(root, this.delegate);
  		//扩展方法，解析xml后处理		
  		postProcessXml(root);
  
  		this.delegate = parent;
  	}
  ```

  DefaultBeanDefinitionDocumentReader#parseBeanDefinitions：

  ```
  	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {		
  		if (delegate.isDefaultNamespace(root)) {
  			NodeList nl = root.getChildNodes();
  			for (int i = 0; i < nl.getLength(); i++) {
  				Node node = nl.item(i);
  				if (node instanceof Element) {
  					Element ele = (Element) node;
  					//处理默认命名空间中的元素
  					if (delegate.isDefaultNamespace(ele)) {
  						parseDefaultElement(ele, delegate);
  					}
  					else {
  						//利用BeanDefinitionParseDelegate解析自定义元素，BeanDefinitionParseDelegate的实现类为DefaultBeanDefinitionParseDelegate
  						delegate.parseCustomElement(ele);
  					}
  				}
  			}
  		}
  		else {
  			delegate.parseCustomElement(root);
  		}
  	}
  ```

  默认命名空间的解析DefaultBeanDefinitionDocumentReader#parseDefaultElement：

  ```
  	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
  		//解析import元素
  		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
  			importBeanDefinitionResource(ele);
  		}
  		//解析alias元素
  		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
  			processAliasRegistration(ele);
  		}
  		//解析bean元素
  		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
  			processBeanDefinition(ele, delegate);
  		}
  		//解析beans元素
  		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
  			// recurse
  			doRegisterBeanDefinitions(ele);
  		}
  	}
  ```

  spring中的可以自定义元素，比如

  ```
  <tx:annotation-driven/>
  ```

  该类元素使用delegate.parseCustomeElement方法解析。

# bean的加载

使用方式：

```
MyTestBean testBean = (MyTestBean) bf.getBean("myTestBean");
```

bf.getBean主要通过AbstractBeanFactory中的doGetBean获得bean对象。

AbstractBeanFactory#doGetBean：

```
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		//1.从name中获的beanName
		final String beanName = transformedBeanName(name);
		Object bean;

		//2.根据beanName从缓存中获得bean实例
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			//3.根据beanName，name和sharedInstance获得bean实例
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			//4.如果是prototype出现循环引用抛出异常
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			//5.支持父子容器，如果从父容器中获取到则返回
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				//6.如果beanName为ChildBeanDefinition将其和父BeanDefinition的定义合并为RootBeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				//7.处理BeanDefinition中的dependsOn属性
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 如果mbd为单例对象
				if (mbd.isSingleton()) {
					//8.获得单例bean，ObjectFactory中通过createBean创建单例对象
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				//如果mbd为prototype类型
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						//创建beanName对应的实例
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
				//获得其它scope类型的bean
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
					   //从Scope中获得bean
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								//创建beanName对应的bean实例
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// 如果bean实例不满足需要的类型，进行类型转换
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}

```

- ​	从name中获得BeanName

  ```
  		//根据name获得转换后的beanName
  		final String beanName = transformedBeanName(name);
  ```

  AbstractBeanFactory#transformedBeanName：

  ```
  	protected String transformedBeanName(String name) {
  		return canonicalName(BeanFactoryUtils.transformedBeanName(name));
  	}
  ```

  去掉name的$前缀。BeanFactoryUtils#transformedBeanName：

  ```
  	public static String transformedBeanName(String name) {
  		Assert.notNull(name, "'name' must not be null");
  		if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
  			return name;
  		}
  		return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
  			do {
  				beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
  			}
  			while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
  			return beanName;
  		});
  	}
  ```

  如果name为别名，获得对应的beanName。SimpleAliasRegistry#canonicalName：

  ```
  	public String canonicalName(String name) {
  		String canonicalName = name;
  		// Handle aliasing...
  		String resolvedName;
  		do {
  			resolvedName = this.aliasMap.get(canonicalName);
  			if (resolvedName != null) {
  				canonicalName = resolvedName;
  			}
  		}
  		while (resolvedName != null);
  		return canonicalName;
  	}
  ```

  由于name中可能带有$，也可能为别名，需要在transformedBeanName中获得其对应的beanName

- 先从缓存中获取bean实例

  ```
  Object sharedInstance = getSingleton(beanName);
  ```

  DefaultSingletonBeanRegistry#getSingleton(java.lang.String)：

  ```
  	public Object getSingleton(String beanName) {
  		//true表示允许循环引用
  		return getSingleton(beanName, true);
  	}
  	
  	
  	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  		//先从singletonObjects中获取bean实例
  		Object singletonObject = this.singletonObjects.get(beanName);
  		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
  			synchronized (this.singletonObjects) {
  				//没有从singletonObjects获取到，尝试从earlySingletonObjects获取
  				singletonObject = this.earlySingletonObjects.get(beanName);
  				//从earlySingletonObjects获取失败，并且允许循环引用的话接着获取
  				if (singletonObject == null && allowEarlyReference) {
  					//从singletonFactories中获取objectFactory
  					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
  					if (singletonFactory != null) {
  						//利用objectFactory创建singletonObject，并且将其放入到earlySingletonObjects，并从singletonFactories中移除对应的ObjectFactory
  						singletonObject = singletonFactory.getObject();
  						this.earlySingletonObjects.put(beanName, singletonObject);
  						this.singletonFactories.remove(beanName);
  					}
  				}
  			}
  		}
  		return singletonObject;
  	}	
  ```

  单例bean首先尝试从缓存中获取，缓存其实是一个map。总共有三个缓存对象：singletonObjects、earlySingletonObjects和singletonFactories。按照该次序依次获取。对于单例bean来说，第一次由于singletonFactories中没有其对应的ObjectFactory而返回null。

- 如果从缓存中获得了对象，调用getObjectForBeanInstance获得需要返回的bean实例

  ```
  bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  ```

  AbstractBeanFactory#getObjectForBeanInstance：

  ```
  	protected Object getObjectForBeanInstance(
  			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
  
  		// 如果name有$前缀直接返回beanInstance
  		if (BeanFactoryUtils.isFactoryDereference(name)) {
  			if (beanInstance instanceof NullBean) {
  				return beanInstance;
  			}
  			if (!(beanInstance instanceof FactoryBean)) {
  				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
  			}
  			if (mbd != null) {
  				mbd.isFactoryBean = true;
  			}
  			return beanInstance;
  		}
  
  		//如果beanInstance不是FactoryBean直接返回
  		if (!(beanInstance instanceof FactoryBean)) {
  			return beanInstance;
  		}
  
  		Object object = null;
  		if (mbd != null) {
  			mbd.isFactoryBean = true;
  		}
  		else {
  			object = getCachedObjectForFactoryBean(beanName);
  		}
  		if (object == null) {
  			// Return bean instance from factory.
  			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
  			// Caches object obtained from FactoryBean if it is a singleton.
  			if (mbd == null && containsBeanDefinition(beanName)) {
  				mbd = getMergedLocalBeanDefinition(beanName);
  			}
  			boolean synthetic = (mbd != null && mbd.isSynthetic());
  			//从factoryBean中获得bean实例
  			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
  		}
  		return object;
  	}
  ```

  如果sharedInstance不是FactoryBean，或者name有$前缀直接返回sharedInstance，否则获得该FactoryBean创建的bean实例。

  FactoryBeanRegistrySupport#getObjectFromFactoryBean：

  ```
  	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
  		//如果factory为单例
  		if (factory.isSingleton() && containsSingleton(beanName)) {
  			synchronized (getSingletonMutex()) {
  				//尝试从缓存中获取
  				Object object = this.factoryBeanObjectCache.get(beanName);
  				if (object == null) {
  					//调用factoryBean的getObject创建bean
  					object = doGetObjectFromFactoryBean(factory, beanName);
  					//将创建的bean实例放入缓存
  					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
  					if (alreadyThere != null) {
  						object = alreadyThere;
  					}
  					else {
  						//需要postProcess
  						if (shouldPostProcess) {
  							if (isSingletonCurrentlyInCreation(beanName)) {
  								// 如果该bean正常创建，返回object
  								return object;
  							}
  							//将beanName添加至singletonsCurrentlyInCreation
  							beforeSingletonCreation(beanName);
  							try {
  								//BeanPostProcess的postProcessAfterInitialization对beanInstance处理
  								object = postProcessObjectFromFactoryBean(object, beanName);
  							}
  							catch (Throwable ex) {
  								throw new BeanCreationException(beanName,
  										"Post-processing of FactoryBean's singleton object failed", ex);
  							}
  							finally {
  								//从singletonsCurrentlyInCreation中移除beanName。表示该bean实例创建完成
  								afterSingletonCreation(beanName);
  							}
  						}
  						if (containsSingleton(beanName)) {
  							this.factoryBeanObjectCache.put(beanName, object);
  						}
  					}
  				}
  				return object;
  			}
  		}
  		//处理非单例bean
  		else {
  			//从factoryBean中获得bean对象
  			Object object = doGetObjectFromFactoryBean(factory, beanName);
  			if (shouldPostProcess) {
  				try {
  					//利用beanPostProcess处理该bean
  					object = postProcessObjectFromFactoryBean(object, beanName);
  				}
  				catch (Throwable ex) {
  					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
  				}
  			}
  			return object;
  		}
  	}
  	
  	
  	private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
  			throws BeanCreationException {
  
  		Object object;
  		try {
  			if (System.getSecurityManager() != null) {
  				AccessControlContext acc = getAccessControlContext();
  				try {
  					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
  				}
  				catch (PrivilegedActionException pae) {
  					throw pae.getException();
  				}
  			}
  			else {
  				//调用factoryBean的getObject
  				object = factory.getObject();
  			}
  		}
  		catch (FactoryBeanNotInitializedException ex) {
  			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
  		}
  		catch (Throwable ex) {
  			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
  		}
  
  		// Do not accept a null value for a FactoryBean that's not fully
  		// initialized yet: Many FactoryBeans just return null then.
  		if (object == null) {
  			if (isSingletonCurrentlyInCreation(beanName)) {
  				throw new BeanCurrentlyInCreationException(
  						beanName, "FactoryBean which is currently in creation returned null from getObject");
  			}
  			object = new NullBean();
  		}
  		return object;
  	}	
  ```

  DefaultSingletonBeanRegistry#beforeSingletonCreation：

  ```
  	//将该单例bean添加至singletonsCurrentlyInCreation
  	protected void beforeSingletonCreation(String beanName) {
  		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
  			throw new BeanCurrentlyInCreationException(beanName);
  		}
  	}
  ```

  AbstractAutowireCapableBeanFactory#postProcessObjectFromFactoryBean：

  ```
  	protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
  		return applyBeanPostProcessorsAfterInitialization(object, beanName);
  	}
  	
      
  	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
  			throws BeansException {
  
  		Object result = existingBean;
  		for (BeanPostProcessor processor : getBeanPostProcessors()) {
  			//调用processor的postProcessAfterInitialization
  			Object current = processor.postProcessAfterInitialization(result, beanName);
  			if (current == null) {
  				return result;
  			}
  			result = current;
  		}
  		return result;
  	}	
  ```

  DefaultSingletonBeanRegistry#afterSingletonCreation：

  ```
  	protected void afterSingletonCreation(String beanName) {
  		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
  			throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
  		}
  	}
  ```

- Propotype类型的bean出现循环引用抛出异常

  ```
  			if (isPrototypeCurrentlyInCreation(beanName)) {
  				throw new BeanCurrentlyInCreationException(beanName);
  			}
  ```

  ```
  	private final ThreadLocal<Object> prototypesCurrentlyInCreation =
  			new NamedThreadLocal<>("Prototype beans currently in creation");
  			
      protected boolean isPrototypeCurrentlyInCreation(String beanName) {
         //该threadLocal保存正在创建的prototype类型的bean实例
         Object curVal = this.prototypesCurrentlyInCreation.get();
         //该prototype的bean实例正在创建
         return (curVal != null &&
               (curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
      }
  ```

  spring不支持类型为prototype的bean进行循环引用

- 从parent工厂中获取bean

  ```
  BeanFactory parentBeanFactory = getParentBeanFactory();
  ```

  spring支持容器的嵌套，

- 合并BeanDefinition获得完整的BeanDefinition

  ```
  final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
  ```

  GenericBeanDefinition、ChildBeanDefinition可以定义父BeanDefinition，从而实现了复用BeanDefinition的目的

  ```
  public class ChildBeanDefinition extends AbstractBeanDefinition {
  
  	@Nullable
  	private String parentName;
  	
  }	
  ```

  ```
  public class GenericBeanDefinition extends AbstractBeanDefinition {
  
  	@Nullable
  	private String parentName;
  	
  }	
  ```

  RootBeanDefinition没有父BeanDefinition，其中包含完整的BeanDefinition。所以getMergedLocalBeanDefinition返回的对象为RootBeanDefinition

- 获得单例bean实例

  ```
  					sharedInstance = getSingleton(beanName, () -> {
  						try {
  							return createBean(beanName, mbd, args);
  						}
  						catch (BeansException ex) {
  							// Explicitly remove instance from singleton cache: It might have been put there
  							// eagerly by the creation process, to allow for circular reference resolution.
  							// Also remove any beans that received a temporary reference to the bean.
  							destroySingleton(beanName);
  							throw ex;
  						}
  					});
  ```

  DefaultSingletonBeanRegistry#getSingleton(java.lang.String,org.springframework.beans.factory.ObjectFactory<?>)：

  ```
  	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  		Assert.notNull(beanName, "Bean name must not be null");
  		synchronized (this.singletonObjects) {
  			//从缓存singletonObjects中获取单例bean
  			Object singletonObject = this.singletonObjects.get(beanName);
  			//没有获取到
  			if (singletonObject == null) {
  				if (this.singletonsCurrentlyInDestruction) {
  					throw new BeanCreationNotAllowedException(beanName,
  							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
  							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
  				}
  				if (logger.isDebugEnabled()) {
  					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
  				}
  				//创建单例bean的准备工作
  				beforeSingletonCreation(beanName);
  				boolean newSingleton = false;
  				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
  				if (recordSuppressedExceptions) {
  					this.suppressedExceptions = new LinkedHashSet<>();
  				}
  				try {
  					//利用ObjectFactory创建单例bean，ObjectFactory的实现类利用lambda实现
  					singletonObject = singletonFactory.getObject();
  					newSingleton = true;
  				}
  				catch (IllegalStateException ex) {
  					// Has the singleton object implicitly appeared in the meantime ->
  					// if yes, proceed with it since the exception indicates that state.
  					singletonObject = this.singletonObjects.get(beanName);
  					if (singletonObject == null) {
  						throw ex;
  					}
  				}
  				catch (BeanCreationException ex) {
  					if (recordSuppressedExceptions) {
  						for (Exception suppressedException : this.suppressedExceptions) {
  							ex.addRelatedCause(suppressedException);
  						}
  					}
  					throw ex;
  				}
  				finally {
  					if (recordSuppressedExceptions) {
  						this.suppressedExceptions = null;
  					}
  					//创建单例bean的善后工作
  					afterSingletonCreation(beanName);
  				}
  				//将单例bean添加至缓存
  				if (newSingleton) {
  					addSingleton(beanName, singletonObject);
  				}
  			}
  			return singletonObject;
  		}
  	}
  ```

  创建单例bean时ObjectFactory的实现类的定义如下：

  ```
  () -> {
  						try {
  							//创建并且完成实例化的bean
  							return createBean(beanName, mbd, args);
  						}
  						catch (BeansException ex) {
  							// Explicitly remove instance from singleton cache: It might have been put there
  							// eagerly by the creation process, to allow for circular reference resolution.
  							// Also remove any beans that received a temporary reference to the bean.
  							destroySingleton(beanName);
  							throw ex;
  						}
  					}
  ```

  - 将单例bean添加至缓存

    DefaultSingletonBeanRegistry#addSingleton：

    ```
    	protected void addSingleton(String beanName, Object singletonObject) {
    		synchronized (this.singletonObjects) {
    			//将单例bean添加至singletonObjects和registeredSingletons，并从singletonFactories和earlySingletonObjects中移除
    			this.singletonObjects.put(beanName, singletonObject);
    			this.singletonFactories.remove(beanName);
    			this.earlySingletonObjects.remove(beanName);
    			this.registeredSingletons.add(beanName);
    		}
    	}
    ```

  - 创建并且初始化bean实例： 

    ```
    	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    			throws BeanCreationException {
    
    		if (logger.isTraceEnabled()) {
    			logger.trace("Creating instance of bean '" + beanName + "'");
    		} 
    		RootBeanDefinition mbdToUse = mbd;
    
    		//从BeanDefinition中获得class对象
    		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    			mbdToUse = new RootBeanDefinition(mbd);
    			//设置BeanDefinition中的BeanClass
    			mbdToUse.setBeanClass(resolvedClass);
    		}
    
    		// Prepare method overrides.
    		try {
    			mbdToUse.prepareMethodOverrides();
    		}
    		catch (BeanDefinitionValidationException ex) {
    			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
    					beanName, "Validation of method overrides failed", ex);
    		}
    
    		try {
    			//利用BeanPostProcess创建代理对象
    			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    			if (bean != null) {
    				return bean;
    			}
    		}
    		catch (Throwable ex) {
    			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
    					"BeanPostProcessor before instantiation of bean failed", ex);
    		}
    
    		try {
    			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    			if (logger.isTraceEnabled()) {
    				logger.trace("Finished creating instance of bean '" + beanName + "'");
    			}
    			return beanInstance;
    		}
    		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
    			// A previously detected exception with proper bean creation context already,
    			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
    			throw ex;
    		}
    		catch (Throwable ex) {
    			throw new BeanCreationException(
    					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    		}
    	}
    
    ```

    - 利用BeanPostProcess创建代理对象

      ```
      	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
      		Object bean = null;
      		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
      			// 如果BeanDefinition中的合成属性为false，并且存在InstantiationAwareBeanPostProcess
      			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      				Class<?> targetType = determineTargetType(beanName, mbd);
      				if (targetType != null) {
      					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
      					//如果创建了代理对象在此处调用BeanPostProcess的postProcessAfterInitilization
      					if (bean != null) {
      						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);  
      					}
      				}
      			}
      			mbd.beforeInstantiationResolved = (bean != null);
      		}
      		return bean;
      	}
      ```

      在InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation方法中可以创建代理对象

      - 创建bean实例

        ```
        	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        			throws BeanCreationException {
        
        		// 保存bean实例
        		BeanWrapper instanceWrapper = null;
        		//单例从缓存中获取
        		if (mbd.isSingleton()) {
        			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
        		}
        		//缓存中没有获取到创建bean实例，在createBeanInstance方法创建bean实例，该方法比较复杂，细节比较多不再进入，不影响整个流程的理解
        		if (instanceWrapper == null) {
        			instanceWrapper = createBeanInstance(beanName, mbd, args);
        		}
        		final Object bean = instanceWrapper.getWrappedInstance();
        		Class<?> beanType = instanceWrapper.getWrappedClass();
        		if (beanType != NullBean.class) {
        			mbd.resolvedTargetType = beanType;
        		}
        
        		// 使用MergedBeanDefinitionPostProcessor处理
        		synchronized (mbd.postProcessingLock) {
        			if (!mbd.postProcessed) {
        				try {
        					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
        				}
        				catch (Throwable ex) {
        					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
        							"Post-processing of merged bean definition failed", ex);
        				}
        				mbd.postProcessed = true;
        			}
        		}
        
        		//允许暴露刚创建的bean实例
        		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        				isSingletonCurrentlyInCreation(beanName));
        		if (earlySingletonExposure) {
        			if (logger.isTraceEnabled()) {
        				logger.trace("Eagerly caching bean '" + beanName +
        						"' to allow for resolving potential circular references");
        			}
        			//向singletonFactories添加ObjectFactory的实现
        			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
        		}
        
        		Object exposedObject = bean;
        		try {
        			//通过set注入依赖
        			populateBean(beanName, mbd, instanceWrapper);
        			//初始化bean实例
        			exposedObject = initializeBean(beanName, exposedObject, mbd);
        		}
        		catch (Throwable ex) {
        			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
        				throw (BeanCreationException) ex;
        			}
        			else {
        				throw new BeanCreationException(
        						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        			}
        		}
        
        		if (earlySingletonExposure) {
        			Object earlySingletonReference = getSingleton(beanName, false);
        			if (earlySingletonReference != null) {
        				if (exposedObject == bean) {
        					exposedObject = earlySingletonReference;
        				}
        				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        					String[] dependentBeans = getDependentBeans(beanName);
        					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        					for (String dependentBean : dependentBeans) {
        						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
        							actualDependentBeans.add(dependentBean);
        						}
        					}
        					if (!actualDependentBeans.isEmpty()) {
        						throw new BeanCurrentlyInCreationException(beanName,
        								"Bean with name '" + beanName + "' has been injected into other beans [" +
        								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
        								"] in its raw version as part of a circular reference, but has eventually been " +
        								"wrapped. This means that said other beans do not use the final version of the " +
        								"bean. This is often the result of over-eager type matching - consider using " +
        								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
        					}
        				}
        			}
        		}
        
        		// Register bean as disposable.
        		try {
        			registerDisposableBeanIfNecessary(beanName, bean, mbd);
        		}
        		catch (BeanDefinitionValidationException ex) {
        			throw new BeanCreationException(
        					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
        		}
        
        		return exposedObject;
        	}
        ```

        - 通过set进行属性注入

          ```
          	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
          		if (bw == null) {
          			if (mbd.hasPropertyValues()) {
          				throw new BeanCreationException(
          						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
          			}
          			else {
          				// Skip property population phase for null instance.
          				return;
          			}
          		}
          
          		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
          		// state of the bean before properties are set. This can be used, for example,
          		// to support styles of field injection.
          		boolean continueWithPropertyPopulation = true;
          		//使用InstantiationAwareBeanPostProcess.postProcessAfterInstantiation处理未初始化的bean实例
          		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
          			for (BeanPostProcessor bp : getBeanPostProcessors()) {
          				if (bp instanceof InstantiationAwareBeanPostProcessor) {
          					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
          						continueWithPropertyPopulation = false;
          						break;
          					}
          				}
          			}
          		}
          
          		//在postProcessAfterInstantiation中可以终止bean的初始化
          		if (!continueWithPropertyPopulation) {
          			return;
          		}
          
          		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
          
          		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
          		//按name注入或者按类型注入，此处只是将属性和注入的值放入到pvs中
          		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
          			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
          			// Add property values based on autowire by name if applicable.
          			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
          				autowireByName(beanName, mbd, bw, newPvs);
          			}
          			// Add property values based on autowire by type if applicable.
          			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
          				autowireByType(beanName, mbd, bw, newPvs);
          			}
          			pvs = newPvs;
          		}
          
          		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
          		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
          
          		PropertyDescriptor[] filteredPds = null;
          		if (hasInstAwareBpps) {
          			if (pvs == null) {
          				pvs = mbd.getPropertyValues();
          			}
          			//使用InstantiationAwareBeanPostProcess.postProcessProperties、InstantiationAwareBeanPostProcess.postProcessPropertyValues处理未初始化的bean
          			for (BeanPostProcessor bp : getBeanPostProcessors()) {
          				if (bp instanceof InstantiationAwareBeanPostProcessor) {
          					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
          					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
          					if (pvsToUse == null) {
          						if (filteredPds == null) {
          							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
          						}
          						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
          						if (pvsToUse == null) {
          							return;
          						}
          					}
          					pvs = pvsToUse;
          				}
          			}
          		}
          		if (needsDepCheck) {
          			if (filteredPds == null) {
          				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
          			}
          			checkDependencies(beanName, mbd, filteredPds, pvs);
          		}
          
          		if (pvs != null) {
          			//利用反射调用属性的set方法进行属性注入
          			applyPropertyValues(beanName, mbd, bw, pvs);
          		}
          	}
          ```

          在进行属性注入之前先利用InstantiationAwareBeanPostProcess的postProcessAfterInstantiation、postProcessProperties和postProcessPropertyValues对bean实例处理

        - bean实例的初始化

          ```
          	protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
          		if (System.getSecurityManager() != null) {
          			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
          				invokeAwareMethods(beanName, bean);
          				return null;
          			}, getAccessControlContext());
          		}
          		else {
          			//BeanNameAware、BeanClassLoaderAware和BeanFactoryAware对bean初始化
          			invokeAwareMethods(beanName, bean);
          		}
          
          		Object wrappedBean = bean;
          		if (mbd == null || !mbd.isSynthetic()) {
          			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
          		}
          
          		try {
          			//调用bean的初始化方法
          			invokeInitMethods(beanName, wrappedBean, mbd);
          		}
          		catch (Throwable ex) {
          			throw new BeanCreationException(
          					(mbd != null ? mbd.getResourceDescription() : null),
          					beanName, "Invocation of init method failed", ex);
          		}
          		if (mbd == null || !mbd.isSynthetic()) {
          			//调用BeanPostProcess的postProcessAfterInitialization
          			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
          		}
          
          		return wrappedBean;
          	}
          ```

          - XXXAware接口对bean初始化

            ```
            	private void invokeAwareMethods(final String beanName, final Object bean) {
            		if (bean instanceof Aware) {
            			if (bean instanceof BeanNameAware) {
            				//BeanNameAware的setBeanName方法调用
            				((BeanNameAware) bean).setBeanName(beanName);
            			}
            			if (bean instanceof BeanClassLoaderAware) {
            				ClassLoader bcl = getBeanClassLoader();
            				if (bcl != null) {
            				//BeanClassLoaderAware的setBeanClassLoader方法调用				
            					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            				}
            			}
            			if (bean instanceof BeanFactoryAware) {
                             //BeanFactoryAware的setBeanFactory方法调用	
            				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
            			}
            		}
            	}
            ```

          - BeanPostProcess的postProcessBeforeInitialization方法进行初始化

            ```
            	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
            			throws BeansException {
            
            		Object result = existingBean;
            		for (BeanPostProcessor processor : getBeanPostProcessors()) {
            			Object current = processor.postProcessBeforeInitialization(result, beanName);
            			if (current == null) {
            				return result;
            			}
            			result = current;
            		}
            		return result;
            	}
            ```

          - bean的初始化方法调用

            ```
            	protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
            			throws Throwable {
            
            		boolean isInitializingBean = (bean instanceof InitializingBean);
            		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            			if (logger.isTraceEnabled()) {
            				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
            			}
            			if (System.getSecurityManager() != null) {
            				try {
            					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
            						((InitializingBean) bean).afterPropertiesSet();
            						return null;
            					}, getAccessControlContext());
            				}
            				catch (PrivilegedActionException pae) {
            					throw pae.getException();
            				}
            			}
            			else {
            				//调用initializingBean的afterPropertiesSet方法
            				((InitializingBean) bean).afterPropertiesSet();
            			}
            		}
            
            		if (mbd != null && bean.getClass() != NullBean.class) {
            			String initMethodName = mbd.getInitMethodName();
            			if (StringUtils.hasLength(initMethodName) &&
            					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
            					!mbd.isExternallyManagedInitMethod(initMethodName)) {
            				//反射调用bean的初始化方法
            				invokeCustomInitMethod(beanName, bean, mbd);
            			}
            		}
            	}
            ```

            - 反射调用BeanDefinition中的初始化方法

              ```
              	protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd)
              			throws Throwable {
              		//获得BeanDefinition中的初始化方法
              		String initMethodName = mbd.getInitMethodName();
              		Assert.state(initMethodName != null, "No init method set");
              		Method initMethod = (mbd.isNonPublicAccessAllowed() ?
              				BeanUtils.findMethod(bean.getClass(), initMethodName) :
              				ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));
              
              		if (initMethod == null) {
              			if (mbd.isEnforceInitMethod()) {
              				throw new BeanDefinitionValidationException("Could not find an init method named '" +
              						initMethodName + "' on bean with name '" + beanName + "'");
              			}
              			else {
              				if (logger.isTraceEnabled()) {
              					logger.trace("No default init method named '" + initMethodName +
              							"' found on bean with name '" + beanName + "'");
              				}
              				// Ignore non-existent default lifecycle methods.
              				return;
              			}
              		}
              
              		if (logger.isTraceEnabled()) {
              			logger.trace("Invoking init method  '" + initMethodName + "' on bean with name '" + beanName + "'");
              		}
              		Method methodToInvoke = ClassUtils.getInterfaceMethodIfPossible(initMethod);
              
              		if (System.getSecurityManager() != null) {
              			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
              				ReflectionUtils.makeAccessible(methodToInvoke);
              				return null;
              			});
              			try {
              				AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
              						methodToInvoke.invoke(bean), getAccessControlContext());
              			}
              			catch (PrivilegedActionException pae) {
              				InvocationTargetException ex = (InvocationTargetException) pae.getException();
              				throw ex.getTargetException();
              			}
              		}
              		else {
              			try {
              				ReflectionUtils.makeAccessible(methodToInvoke);
              				//反射调用初始化方法
              				methodToInvoke.invoke(bean);
              			}
              			catch (InvocationTargetException ex) {
              				throw ex.getTargetException();
              			}
              		}
              	}
              ```

            spring中bean的初始化指的是bean中实现InitailizingBean的afterPropertiesSet和初始化方法的调用

### 什么是循环依赖

ioc的依赖注入分为构造器属性注入和setter属性注入，所以循环依赖包括构造函数循环依赖和setter循环依赖注入。

demo展示

```
public class TestA {

    private TestB testB;

    public TestA(TestB testB) {
        this.testB = testB;
    }

    public void a() {
        testB.b();
    }

    public void setTestB(TestB testB) {
        this.testB = testB;
    }

    public TestB getTestB() {
        return testB;
    }

}

public class TestB {

    private TestC testc;

    public TestB(TestC testC) {
        this.testc = testc;
    }

    public void b() {
        testc.c();
    }

    public void setTestc(TestC testc) {
        this.testc = testc;
    }

    public TestC getTestc() {
        return testc;
    }
}

public class TestC {

    private TestA testA;


    public TestC(TestA testA) {
        this.testA = testA;
    }

    public void c() {
        testA.a();
    }

    public void setTestA(TestA testA) {
        this.testA = testA;
    }

    public TestA getTestA() {
        return testA;
    }

}
```

beanFactoryTest.xml中增加bean的配置：

```
	<bean id="testA"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestA">
		<constructor-arg index="0" ref="testB" />
	</bean>

	<bean id="testB"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestB">
		<constructor-arg index="0" ref="testC" />
	</bean>

	<bean id="testC"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestC">
		<constructor-arg index="0" ref="testA" />
	</bean>
```

测试类：

```
    @Test(expected = BeanCurrentlyInCreationException.class)
    public void testCircleByConstructor(){
        BeanFactory bf=new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));
        TestA testBean = (TestA) bf.getBean("testA");
    }
```

结果：

程序抛出BeanCurrentlyInCreationException异常

#### 构造器循环依赖

Spring中出现构造器循环依赖会抛出BeanCurrentlyInCreationException异常。Spring利用singletonsCurrentlyInCreatation检测构造器循环依赖，singletonsCurrentlyInCreatation中保存正在创建的单例bean。

抛出异常的方法为DefaultSingletonBeanRegistry#beforeSingletonCreation：

```
	protected void beforeSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}
```

程序的流程为：

- 调用getBean获得testA的bean实例时，此时容器中没有testA。创建testA实例，使用构造器依赖注入时需要testB

  ![1653995228707](D:\学习\spring源码深度解析\assets\1653995228707.png)

  通过getBean获取testA的调用栈。构造器依赖注入时需要testB实例

- 由于容器中没有testB，创建testB时使用构造依赖注入需要testC

  ![1653995358167](D:\学习\spring源码深度解析\assets\1653995358167.png)

  通过getBean获取testB的调用栈。构造器依赖注入时需要testC实例

- 由于容器中没有testC，创建testC时使用构造依赖注入需要testA

  ![1653995465679](D:\学习\spring源码深度解析\assets\1653995465679.png)

  通过getBean获取testC的调用栈。构造器依赖注入时需要testA实例

- 再次通过getBean获取testA实例

  ![1653995529571](D:\学习\spring源码深度解析\assets\1653995529571.png)

  由于singletonsCurrentlyInCreatation已经有testA，则添加testA返回false。方法beforeSingletonCreation中抛出BeanCurrentlyInCreatationException

#### setter循环依赖

##### demo展示：

beanFactoryTest.xml配置如下：

```
	<bean id="testA"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestA" >
		<property name="testB" ref="testB"/>
	</bean>

	<bean id="testB"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestB">
		<property name="testC" ref="testC"/>
	</bean>

	<bean id="testC"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestC">
		<property name="testA" ref="testA"/>
	</bean>
```

测试用例如下：

```
    @Test
    public void testCircleBySetter(){
        BeanFactory bf=new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));
        TestA testA = (TestA) bf.getBean("testA");
        System.out.println(testA);
    }
```

结果输出：

```
com.zchuber.springsourcedeepdiving.ioc.TestA@6e06451e
```

setter注入时发生循环依赖也能正常完成beand的创建和初始化。

##### 过程分析：

- 通过getBean获取testA的bean实例时，此时一、二，三级缓存中都没有testA实例，创建testA实例。通过无参构造函数创建testA实例，然后进行setter依赖注入时需要testB实例。

  ![1653997435270](D:\学习\spring源码深度解析\assets\1653997435270.png)

  在populate方法中进行setter属性注入时需要testB实例。

  注：在创建testA实例后，setter属性注入前向三级缓存singletonFactories中添加testA和ObjectFactory对象。

  AbstractAutowireCapableBeanFactory#doCreateBean:

  ```
  		instanceWrapper = createBeanInstance(beanName, mbd, args);
  		...
  		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  		...
  		populateBean(beanName, mbd, instanceWrapper);
  		exposedObject = initializeBean(beanName, exposedObject, mbd);
  ```

  DefaultSingletonBeanRegistry#addSingletonFactory:

  ```
  	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
  		synchronized (this.singletonObjects) {
  			if (!this.singletonObjects.containsKey(beanName)) {
  				this.singletonFactories.put(beanName, singletonFactory);
  				this.earlySingletonObjects.remove(beanName);
  				this.registeredSingletons.add(beanName);
  			}
  		}
  	}
  ```

  addSingletonFactory方法主要是将testA添加至三级缓存

  AbstractAutowireCapableBeanFactory#getEarlyBeanReference：

  ```
  	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
  		Object exposedObject = bean;
  		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
  			for (BeanPostProcessor bp : getBeanPostProcessors()) {
  				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
  					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
  					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
  				}
  			}
  		}
  		return exposedObject;
  	}
  ```

  getEarlyBeanReference方法在没有SmartInstantiationAwareBeanPostProcessor的情况下直接返回bean对象。

- 通过getBean获取testB实例时，此时一、二，三级缓存中都没有testB实例，创建、初始化testB实例。通过无参构造函数创建testB实例，然后进行setter依赖注入时需要testC实例

  ![1653997597073](D:\学习\spring源码深度解析\assets\1653997597073.png)

  创建testB实例后在populate方法中进行setter属性注入时需要testC实例

- 通过getBean获取testC实例时，此时一、二，三级缓存中都没有testC实例，创建、初始化testC实例。通过无参构造函数创建testC实例，然后进行setter依赖注入时需要testA实例

  ![1653997687585](D:\学习\spring源码深度解析\assets\1653997687585.png)

  创建testC实例后在populate方法中进行setter属性注入时需要testA实例

- 通过getBean获取testA实例时，由于三级缓存中存在testA和ObjectFactory对象，会从缓存中获取到未完成初始化的testA实例后返回。

  ![1653998545693](D:\学习\spring源码深度解析\assets\1653998545693.png)

  此时testA实例中的testB属性未注入。

- 获得到testA后testC中setter属性注入完成，从而testC的初始化

- 获得到testC后testB中setter属性注入完成，从而testB的初始化

- 获得testB后testA中的setter属性注入完成，从而完成testA的初始化

  何时二级缓存earlySIngletonObjects中删除testA，一级缓存中添加testA？

  DefaultSingletonBeanRegistry#getSingleton：

  ```
  	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  	  ...
        singletonObject = singletonFactory.getObject();
        ...
        addSingleton(beanName, singletonObject);
      }  
  ```

  singletonFactory.getObject会调用createBean方法创建并初始化testA实例。在addSingleton方法中将testA从二级缓存中移除，添加至一级缓存。

  DefaultSingletonBeanRegistry#addSingleton：

  ```
  	protected void addSingleton(String beanName, Object singletonObject) {
  		synchronized (this.singletonObjects) {
  			this.singletonObjects.put(beanName, singletonObject);
  			this.singletonFactories.remove(beanName);
  			this.earlySingletonObjects.remove(beanName);
  			this.registeredSingletons.add(beanName);
  		}
  	}
  ```

一级缓存singletonObjects保存完成初始化的bean实例，二级缓存earlySingletonObjects中保存未完成初始化的bean实例，三级缓存singletonFactories保存获取bean实例的ObjectFactory。

一级缓存一定会有，三级缓存在创建对象时也会有，二级缓存只有发生循环依赖的bean实例才会有。比如在该类中一级缓存一定有testA、testB和testC，二级缓存只有testA在完成初始化前存在，三级缓存在testA、testB和testC完成初始化前都存在。利用三级缓存可以避免每个未初始化的bean在添加到二级缓存前都经过SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference的处理。在该类中只有testA需要这种处理。

```
	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
				}
			}
		}
		return exposedObject;
	}
```

#### scope为prototype的bean无法循环依赖

beanFactoryTest.xml配置如下：

```
	<bean id="testA"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestA" scope="prototype" >
		<property name="testB" ref="testB"/>
	</bean>

	<bean id="testB"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestB" scope="prototype">
		<property name="testC" ref="testC"/>
	</bean>

	<bean id="testC"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestC" scope="prototype">
		<property name="testA" ref="testA"/>
	</bean>
```

运行结果：

```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'testA' defined in class path resource [beanFactoryTest.xml]: Cannot resolve reference to bean 'testB' while setting bean property 'testB'; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'testB' defined in class path resource [beanFactoryTest.xml]: Cannot resolve reference to bean 'testC' while setting bean property 'testC'; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'testC' defined in class path resource [beanFactoryTest.xml]: Cannot resolve reference to bean 'testA' while setting bean property 'testA'; nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'testA': Requested bean is currently in creation: Is there an unresolvable circular reference?

	at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveReference(BeanDefinitionValueResolver.java:342)
	at org.springframework.beans.factory.support.BeanDefinitionValueResolver.resolveValueIfNecessary(BeanDefinitionValueResolver.java:113)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyPropertyValues(AbstractAutowireCapableBeanFactory.java:1706)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1451)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:594)
```

由于一、二、三级缓存中只保存单例bean，所以prototype类型的bean无法解决setter注入的循环依赖

# 容器的功能扩展

ApplicationContext在BeanFactory的基础上做了功能扩展。建议使用ApplicationContext而不是BeanFactory。

使用方法：

```
ApplicationContext ac=new ClassPathXmlApplicationContext("beanFactoryTest.xml");
```

ClassPathXmlApplicationContext#ClassPathXmlApplicationContext：

```
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		//设置配置文件的路径
		setConfigLocations(configLocations);
		if (refresh) {
			//刷新applicationcontext
			refresh();
		}
	}
```

## 设置配置文件路径

```
	public void setConfigLocations(@Nullable String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
				//解析指定路径，如果location中包含变量则进行替换
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
```

## 扩展功能

```
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备刷新上下文
			prepareRefresh();

			//创建并且刷新BeanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 注册beanFactory
			prepareBeanFactory(beanFactory);

			try {
				//postprocess BeanFactory
				postProcessBeanFactory(beanFactory);

				// 调用BeanFactoryPostProcessors的方法
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册BeanPostProcessors
				registerBeanPostProcessors(beanFactory);

				// 初始化国际化资源
				initMessageSource();

				// 初始化applicationContext中的广播器
				initApplicationEventMulticaster();

				// 供子类扩展的构子
				onRefresh();

				// 向上下文中注册ApplicationListener
				registerListeners();

				//创建并且初始化非lazy-init的单例bean
				finishBeanFactoryInitialization(beanFactory);

				//  发布refresh的相关时间
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

```

## 上下文环境准备

```
	protected void prepareRefresh() {
		// Switch to active.
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// 供子类初始化propertySource
		initPropertySources();

		// 校验环境中需要的属性是否都满足
		getEnvironment().validateRequiredProperties();

		// Store pre-refresh ApplicationListeners...
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```

initPropertySources()使用场景：容器缺少必备的环境变量时会启动失败。

示例展示：

自定义MyClassPathXmlApplicationContext类：

```
public class MyClassPathXmlApplicationContext extends ClassPathXmlApplicationContext {

    public MyClassPathXmlApplicationContext(String... configLocations) {
        super(configLocations);
    }


    @Override
    protected void initPropertySources() {
    	//添加VAR到环境的必需属性中
        getEnvironment().setRequiredProperties("VAR");
    }


}
```

这样启动后会通过getEnvironment().validateRequiredProperties()坚持环境变量中是否存在VAR变量，没有的话会异常退出

## 获取刷新后的BeanFactory

AbstractApplicationContext#obtainFreshBeanFactory：

```
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}
```

AbstractRefreshableApplicationContext#refreshBeanFactory：

```
	protected final void refreshBeanFactory() throws BeansException {
		//如果已经创建BeanFactory，将其销毁，这也是refreshBeanFactory中refresh的含义
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			//创建DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			//供子类扩展的方法
			customizeBeanFactory(beanFactory);
			//加载并注册BeanDefinition到beanFactory中，留给子类重写
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
	
```

### 定制BeanFactory

设置是否允许BeanDefinition覆盖和循环引用

```
	protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
		if (this.allowBeanDefinitionOverriding != null) {
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		if (this.allowCircularReferences != null) {
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}	
```

示例展示：

MyClassPathXmlApplicationContext中重写了customizeBeanFactory方法

```
public class MyClassPathXmlApplicationContext extends ClassPathXmlApplicationContext {

    public MyClassPathXmlApplicationContext(String... configLocations) {
        super(configLocations);
    }

    @Override
    protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory){
        super.setAllowBeanDefinitionOverriding(true);
        super.setAllowCircularReferences(true);
        super.customizeBeanFactory(beanFactory);
    }


}
```

老版本的也会在此设置BeanFactory的AutowireCandidateResolver

### 加载BeanDefinition

AbstractRefreshableApplicationContext#loadBeanDefinitions：

```
	protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory)
			throws BeansException, IOException;
```

实例的加载交给子类实现。如果是子类时AbstractXmlApplicationContext实现如下：

AbstractXmlApplicationContext#loadBeanDefinitions：

```
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		//创建XmlBeanDefinitionReader对象
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// 设置beanDefinitionReader对象
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		//留给子类初始化BeanDefinitionReader
		initBeanDefinitionReader(beanDefinitionReader);
		//利用beanDefinitionReader加载并注册BeanDefinition
		loadBeanDefinitions(beanDefinitionReader);
	}
	

	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		//从resource中加载
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		
		//从配置路径中加载
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
```

因为XmlBeanDefinitionReader初始化时获得DefaultListableBeanFactory，则会将加载的BeanDefinition注册到DefaultListableBeanFactory。

### 返回刷新后的BeanFactory

```
	public final ConfigurableListableBeanFactory getBeanFactory() {
		synchronized (this.beanFactoryMonitor) {
			if (this.beanFactory == null) {
				throw new IllegalStateException("BeanFactory not initialized or already closed - " +
						"call 'refresh' before accessing beans via the ApplicationContext");
			}
			//直接返回成员变量beanFactory，在上一步refreshBeanFactory()中为成员变量beanFactory赋值
			return this.beanFactory;
		}
	}
```

## 功能扩展

```
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 为beanFactory设置beanClassLoader
		beanFactory.setBeanClassLoader(getClassLoader());
		//支持spel语法
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		//支持属性的类型转换
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// 添加BeanPostProcessor并且设置属性中忽略的依赖接口
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		//设置自动装配的特殊规则
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		//添加ApplicationListenerDetector
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// 添加AspectJ的支持
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// 注册系统默认的bean
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

### 增加Spel语法的支持

```
	<bean id="testA"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestA"   >
		<property name="testB" value="#{testB}"/>
	</bean>-->

	<bean id="testB"
		  class="com.zchuber.springsourcedeepdiving.ioc.TestB"  >
		<property name="testC" ref="testC"/>
	</bean>
```

spel语法使用${beanName}的形式。

StandardBeanExpressResolver是为了支持Spel语法

### 增加属性注册编辑器

#### 属性注册编辑器的作用

示例展示：

```
public class UserManager {

    private Date dataValue;

    public Date getDataValue() {
        return dataValue;
    }

    public void setDataValue(Date dataValue) {
        this.dataValue = dataValue;
    }

    @Override
    public String toString() {
        return "dataValue: "+dataValue;
    }

}
```

```
	<bean id="userManager"
		  class="com.zchuber.springsourcedeepdiving.ioc.UserManager"  >
		<property name="dataValue">
			<value>2022-06-01</value>
		</property>
	</bean>
```

测试类：

```
    @Test
    public void testDate(){
        ApplicationContext ac=new ClassPathXmlApplicationContext("beanFactoryTest.xml");
        UserManager userManager = (UserManager) ac.getBean("userManager");
        System.out.println(userManager);
    }
```

运行结果：

```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'userManager' defined in class path resource [beanFactoryTest.xml]: Initialization of bean failed; nested exception is org.springframework.beans.ConversionNotSupportedException: Failed to convert property value of type 'java.lang.String' to required type 'java.util.Date' for property 'dataValue'; nested exception is java.lang.IllegalStateException: Cannot convert value of type 'java.lang.String' to required type 'java.util.Date' for property 'dataValue': no matching editors or conversion strategy found

	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:603)
```

抛出类型不匹配的异常。UserMapper中需要Date类型的dateValue，但是<bean/>标签的dateValue确实String类型。

##### 如何解决哪？

新增PropertyEditorSupport的子类DatePropertyEditor：

```
public class DatePropertyEditor extends PropertyEditorSupport {

    private String format = "yyyy-MM-dd";

    public DatePropertyEditor() {
    }

    public DatePropertyEditor(String format) {
        this.format=format;
    }

    public void setFormat(String format) {
        this.format = format;
    }

    @Override
    public void setAsText(String argo) {
        System.out.println(argo);
        SimpleDateFormat sdf = new SimpleDateFormat(format);
        try {
            Date d=sdf.parse(argo);
            this.setValue(d);
        } catch (ParseException e) {
            e.printStackTrace();
        }

    }


}
```

```
	<bean id="userManager"
		  class="com.zchuber.springsourcedeepdiving.ioc.UserManager"  >
		<property name="dataValue">
			<value>2022-06-01</value>
		</property>
	</bean>
```

- 向CustomEditorConfigure中的customEditors属性添加元素

  ```
  	<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer"  >
  		<property name="customEditors">
  			<map>
  				<entry key="java.util.Date" value="com.zchuber.springsourcedeepdiving.ioc.DatePropertyEditor">
  				</entry>
  			</map>
  		</property>
  	</bean>
  ```

  运行结果：

  ```
  2022-06-01
  dataValue: Wed Jun 01 00:00:00 CST 2022
  ```

  customEditors中的key为java.util.Date，value为DatePropertyEditor，当对bean进行属性注入的时候碰到date类型的属性会调用DatePropertyEditor处理

- 向CustomPropertyConfigurer中的PropertyEditorRegistrars数组添加元素

  新建PropertyEditorRegistrar的实现类：

  ```
  public class DatePropertyEditorRegistrar implements PropertyEditorRegistrar {
  
  
      public void registerCustomEditors(PropertyEditorRegistry registry) {
          registry.registerCustomEditor(Date.class,new DatePropertyEditor("yyyy-MM-dd"));
      }
  
  
  }
  ```

  ```
  	<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer"  >
  		<property name="propertyEditorRegistrars">
  			<list>
  				<bean class="com.zchuber.springsourcedeepdiving.ioc.DatePropertyEditorRegistrar"/>
  			</list>
  		</property>
  	</bean>
  ```

  运行结果：

  ```
  2022-06-01
  dataValue: Wed Jun 01 00:00:00 CST 2022
  ```

ResourceEditorRegistrar的定义如下：

```
public class ResourceEditorRegistrar implements PropertyEditorRegistrar {

	//定义各种类型的propertyEditor
	public void registerCustomEditors(PropertyEditorRegistry registry) {
		ResourceEditor baseEditor = new ResourceEditor(this.resourceLoader, this.propertyResolver);
		doRegisterEditor(registry, Resource.class, baseEditor);
		doRegisterEditor(registry, ContextResource.class, baseEditor);
		doRegisterEditor(registry, InputStream.class, new InputStreamEditor(baseEditor));
		doRegisterEditor(registry, InputSource.class, new InputSourceEditor(baseEditor));
		doRegisterEditor(registry, File.class, new FileEditor(baseEditor));
		doRegisterEditor(registry, Path.class, new PathEditor(baseEditor));
		doRegisterEditor(registry, Reader.class, new ReaderEditor(baseEditor));
		doRegisterEditor(registry, URL.class, new URLEditor(baseEditor));

		ClassLoader classLoader = this.resourceLoader.getClassLoader();
		doRegisterEditor(registry, URI.class, new URIEditor(classLoader));
		doRegisterEditor(registry, Class.class, new ClassEditor(classLoader));
		doRegisterEditor(registry, Class[].class, new ClassArrayEditor(classLoader));

		if (this.resourceLoader instanceof ResourcePatternResolver) {
			doRegisterEditor(registry, Resource[].class,
					new ResourceArrayPropertyEditor((ResourcePatternResolver) this.resourceLoader, this.propertyResolver));
		}
	}
	
	private void doRegisterEditor(PropertyEditorRegistry registry, Class<?> requiredType, PropertyEditor editor) {
		if (registry instanceof PropertyEditorRegistrySupport) {
			((PropertyEditorRegistrySupport) registry).overrideDefaultEditor(requiredType, editor);
		}
		else {
			registry.registerCustomEditor(requiredType, editor);
		}
	}	
	
}	
```

ResourceEditorRegistrar的中最重要的方法是registerCustomEditors，该方法何时被调用？

AbstractBeanFactory#registerCustomEditors：

```
	protected void registerCustomEditors(PropertyEditorRegistry registry) {
		...
        for (PropertyEditorRegistrar registrar : this.propertyEditorRegistrars) {
        	registrar.registerCustomEditors(registry);
        ...
    }    
```

![1654128773632](D:\学习\spring源码深度解析\assets\1654128773632.png)

重点看initBeanWrapper方法。

AbstractBeanFactory#initBeanWrapper：

```
	protected void initBeanWrapper(BeanWrapper bw) {
		bw.setConversionService(getConversionService());
		registerCustomEditors(bw);
	}
```

BeanWrapper实现PropertyEditorRegistry接口，其主要的实现类为BeanWrapperImpl：

![1654128893231](D:\学习\spring源码深度解析\assets\1654128893231.png)

Spring在populate和initializeBean方法中使用的是BeanWrapperImpl。

BeanWrapperImpl也继承了PropertyEditorRegistrySupport类。

PropertyEditorRegistrySupport#createDefaultEditors：

```
	private void createDefaultEditors() {
		this.defaultEditors = new HashMap<>(64);

		// Simple editors, without parameterization capabilities.
		// The JDK does not contain a default editor for any of these target types.
		this.defaultEditors.put(Charset.class, new CharsetEditor());
		this.defaultEditors.put(Class.class, new ClassEditor());
		this.defaultEditors.put(Class[].class, new ClassArrayEditor());
		this.defaultEditors.put(Currency.class, new CurrencyEditor());
		this.defaultEditors.put(File.class, new FileEditor());
		this.defaultEditors.put(InputStream.class, new InputStreamEditor());
		this.defaultEditors.put(InputSource.class, new InputSourceEditor());
		this.defaultEditors.put(Locale.class, new LocaleEditor());
		this.defaultEditors.put(Path.class, new PathEditor());
		this.defaultEditors.put(Pattern.class, new PatternEditor());
		this.defaultEditors.put(Properties.class, new PropertiesEditor());
		this.defaultEditors.put(Reader.class, new ReaderEditor());
		this.defaultEditors.put(Resource[].class, new ResourceArrayPropertyEditor());
		this.defaultEditors.put(TimeZone.class, new TimeZoneEditor());
		this.defaultEditors.put(URI.class, new URIEditor());
		this.defaultEditors.put(URL.class, new URLEditor());
		this.defaultEditors.put(UUID.class, new UUIDEditor());
		this.defaultEditors.put(ZoneId.class, new ZoneIdEditor());

		// Default instances of collection editors.
		// Can be overridden by registering custom instances of those as custom editors.
		this.defaultEditors.put(Collection.class, new CustomCollectionEditor(Collection.class));
		this.defaultEditors.put(Set.class, new CustomCollectionEditor(Set.class));
		this.defaultEditors.put(SortedSet.class, new CustomCollectionEditor(SortedSet.class));
		this.defaultEditors.put(List.class, new CustomCollectionEditor(List.class));
		this.defaultEditors.put(SortedMap.class, new CustomMapEditor(SortedMap.class));

		// Default editors for primitive arrays.
		this.defaultEditors.put(byte[].class, new ByteArrayPropertyEditor());
		this.defaultEditors.put(char[].class, new CharArrayPropertyEditor());

		// The JDK does not contain a default editor for char!
		this.defaultEditors.put(char.class, new CharacterEditor(false));
		this.defaultEditors.put(Character.class, new CharacterEditor(true));

		// Spring's CustomBooleanEditor accepts more flag values than the JDK's default editor.
		this.defaultEditors.put(boolean.class, new CustomBooleanEditor(false));
		this.defaultEditors.put(Boolean.class, new CustomBooleanEditor(true));

		// The JDK does not contain default editors for number wrapper types!
		// Override JDK primitive number editors with our own CustomNumberEditor.
		this.defaultEditors.put(byte.class, new CustomNumberEditor(Byte.class, false));
		this.defaultEditors.put(Byte.class, new CustomNumberEditor(Byte.class, true));
		this.defaultEditors.put(short.class, new CustomNumberEditor(Short.class, false));
		this.defaultEditors.put(Short.class, new CustomNumberEditor(Short.class, true));
		this.defaultEditors.put(int.class, new CustomNumberEditor(Integer.class, false));
		this.defaultEditors.put(Integer.class, new CustomNumberEditor(Integer.class, true));
		this.defaultEditors.put(long.class, new CustomNumberEditor(Long.class, false));
		this.defaultEditors.put(Long.class, new CustomNumberEditor(Long.class, true));
		this.defaultEditors.put(float.class, new CustomNumberEditor(Float.class, false));
		this.defaultEditors.put(Float.class, new CustomNumberEditor(Float.class, true));
		this.defaultEditors.put(double.class, new CustomNumberEditor(Double.class, false));
		this.defaultEditors.put(Double.class, new CustomNumberEditor(Double.class, true));
		this.defaultEditors.put(BigDecimal.class, new CustomNumberEditor(BigDecimal.class, true));
		this.defaultEditors.put(BigInteger.class, new CustomNumberEditor(BigInteger.class, true));

		// Only register config value editors if explicitly requested.
		if (this.configValueEditorsActive) {
			StringArrayPropertyEditor sae = new StringArrayPropertyEditor();
			this.defaultEditors.put(String[].class, sae);
			this.defaultEditors.put(short[].class, sae);
			this.defaultEditors.put(int[].class, sae);
			this.defaultEditors.put(long[].class, sae);
		}
	}

```

createDefaultEditors中注册了许多默认的PropertyEditor接口的实现类。只有该方法中没有注册PropertyEditor才需要额外定义和注册

该过程涉及的主要类为：

- PropertyEditorSupport及其子类：定义属性转换逻辑
- PropertyEditorRegistrar及其子类：向PropertyEditorRegistry中注册PropertyEditor
- CustomEditorConfigurer：通过属性PropertyEditorRegistrars数组和customEditors保存PropertyEditorRegistrar和自定义的转换关系

### 添加ApplicationContextAwareProcessor

ApplicationContextAwareProcessor作为BeanPostProcessor，主要看postProcessBeforeInialization中：

```
class ApplicationContextAwareProcessor implements BeanPostProcessor {

	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
				bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
				bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
			return bean;
		}

		AccessControlContext acc = null;

		if (System.getSecurityManager() != null) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			//调用Aware接口的方法
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
		//为bean设置Environment对象
		if (bean instanceof EnvironmentAware) {
			((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
		}
        //为bean设置EmbeddedValueResolverAware对象
		if (bean instanceof EmbeddedValueResolverAware) {
			((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
		}
        //为bean设置ResourceLoader对象		
		if (bean instanceof ResourceLoaderAware) {
			((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
		}
        //为bean设置ApplicationEventPublisher对象				
		if (bean instanceof ApplicationEventPublisherAware) {
			((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
		}
        //为bean设置MessageSource对象		
		if (bean instanceof MessageSourceAware) {
			((MessageSourceAware) bean).setMessageSource(this.applicationContext);
		}
        //为bean设置ApplicationContext对象		
		if (bean instanceof ApplicationContextAware) {
			((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
		}
	}
}	
```

### 设置依赖中忽略的接口

由于在ApplicationContextaAwareProcessor中已经设置了相应类型的属性，所以需要添加忽略该类型的依赖

```
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

## BeanFactory的后处理

### 激活注册的BeanFactoryPostProcessor

BeanFactoryPostProcessor作用的时间点在所有的BeanDefinition被加载成功，没有bean实例被创建之间的时间点。因而可以利用BeanFactoryPostProcessor对BeanDefinition进行修改或者添加新的BeanDefinition。如果需要控制多个BeanFactoryPostProcessor实现类的顺序，可以让其继承Order。

BeanFactoryPostProcessor作用范围是容器，只会对该容器中的BeanDefinition进行处理。

### BeanFactoryPostProcessor的典型应用：PropertyPlaceholderConfigurer

使用示例：

```
public class HelloMessage {


    private String mes;

    public void setMes(String mes) {
        this.mes=mes;
    }

    public String getMes() {
        return mes;
    }

}
```

```
	<bean id="helloMsg" class="com.zchuber.springsourcedeepdiving.ioc.HelloMessage" >
		<property name="mes">
			<value>${bean.message}</value>
		</property>
	</bean>
	
	<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer" >
		<property name="location">
			<value>propertyPlaceholderConfigurer.properties</value>
		</property>
	</bean>	
```

propertyPlaceholderConfigurer.properties：

```
bean.message=helllo message
```

测试用例：

```
    @Test
    public void testHelloMsg(){
        ApplicationContext ac=new ClassPathXmlApplicationContext("beanFactoryTest.xml");
        HelloMessage msg = (HelloMessage) ac.getBean("helloMsg");
        System.out.println(msg.getMes());
    }
```

运行结果：

```
helllo message
```



PropertyPlaceholderConfigurer的继承体系如下：![1654167725527](D:\学习\spring源码深度解析\assets\1654167725527.png)

PropertyPlaceholderConfigure继承PlaceholderConfigurerSupport，而PlaceholderConfigurerSupport继承BeanFactoryPostProcessor接口。对于BeanFactoryPostProcessor接口最重要的方法是postProcessBeanFactory。

PropertyResourceConfigurer#postProcessBeanFactory：

```
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		try {
			Properties mergedProps = mergeProperties();

			// Convert the merged properties, if necessary.
			convertProperties(mergedProps);

			// Let the subclass process the properties.
			processProperties(beanFactory, mergedProps);
		}
		catch (IOException ex) {
			throw new BeanInitializationException("Could not load properties", ex);
		}
	}
```

在该方法完成BeanDefinition中的占位符的替换

#### 使用自定义BeanFactoryPostProcessor

利用BeanFactoryPostProcessor的子类隐藏BeanDefinition中的敏感词

示例展示：

```
public class SensitiveWordRemovingBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    private Set<String> sensitiveWord = new HashSet<String>();



    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        String[] beanNames = beanFactory.getBeanDefinitionNames();
        for(String beanName:beanNames) {
            BeanDefinition bd = beanFactory.getBeanDefinition(beanName);
            StringValueResolver valueResolver = new StringValueResolver(){

                public String resolveStringValue(String word) {
                    if(isSensitiveWord(word)){
                        return "******";
                    }
                    return word;
                }

                private boolean isSensitiveWord(String word) {
                    return sensitiveWord.contains(word);
                }
            };
            BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);
            visitor.visitBeanDefinition(bd);
        }
    }


    public void setSensitiveWord(Set sensitiveWord) {
        this.sensitiveWord = sensitiveWord;
    }
}

```

```
public class SimplePostProcessor {

    private String connectionString;
    private String password;
    private String uername;

    public void setConnectionString(String connectionString) {
        this.connectionString = connectionString;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public void setUername(String uername) {
        this.uername = uername;
    }

    @Override
    public String toString() {
        return "SimplePostProcessor{" +
                "connectionString='" + connectionString + '\'' +
                ", password='" + password + '\'' +
                ", uername='" + uername + '\'' +
                '}';
    }
}
```

```
	<bean id="swr" class="com.zchuber.springsourcedeepdiving.ioc.SensitiveWordRemovingBeanFactoryPostProcessor" >
		<property name="sensitiveWord">
			<set>
				<value>hello</value>
				<value>world</value>
			</set>
		</property>
	</bean>

	<bean id="spp" class="com.zchuber.springsourcedeepdiving.ioc.SimplePostProcessor" >
		<property name="connectionString" value="hello" />
		<property name="password" value="world" />
		<property name="uername" value="nihoa" />
	</bean>
```

测试方法：

```
    @Test
    public void testSensitiveWord(){
        ConfigurableListableBeanFactory bf = new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));
        SensitiveWordRemovingBeanFactoryPostProcessor swr = (SensitiveWordRemovingBeanFactoryPostProcessor) bf.getBean("swr");
        swr.postProcessBeanFactory(bf);
        SimplePostProcessor spp = (SimplePostProcessor) bf.getBean("spp");
        System.out.println(spp);
    }
```

运行结果：

```
SimplePostProcessor{connectionString='******', password='******', uername='nihao'}
```

SimplePostProcessor中的敏感属性被隐藏了。

#### 激活BeanFactoryPostProcessor

```
	//入参beanFactoryPostProcessors表示为AbstractApplicationContext属性中的beanFactoryPostProcessor，主要和beanFactory管理的bean为beanFactoryPostProcessors进行区分
	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// 保存处理过的beanFactoryPostProcessor
		Set<String> processedBeans = new HashSet<>();
		
		//如果beanFactory为BeanDefinitionRegistry的实现类
		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			//AbstractApplicationContext的属性BeanfactoryPostProcessor分为两类：BeanDefinitionRegistryPostProcessor实现类（registryProcessors）和非BeanDefinitionRegistryPostProcessor实现类（regularPostProcessors）
			List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();
			
			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				//处理BeanDefinitionRegistryPostProcessor
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					//调用registryProcessor的postProcessorBeanDefinitionRegistry
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					//添加至registryProcessors
					registryProcessors.add(registryProcessor);
				}
				else {
					//添加至regularPostProcessors
					regularPostProcessors.add(postProcessor);
				}
			}

			
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

			// 获得当前beanFactory中bean形式的BeanDefinitionRegistryPostProcessors
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					//currentRegistryProcessors保存实现PriorityOrdered接口的BeanDefinitionRegistryPostProcessor
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			//排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			//添加至registryProcessors
			registryProcessors.addAll(currentRegistryProcessors);
			//调用BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			//清理currentRegistryProcessors
			currentRegistryProcessors.clear();

			// 获得beanFactory中bean形式的BeanDefinitionRegistryPostProcessor
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				//跳过已经处理的BeanDefinitionRegistryPostProcessor，并且实现Order接口
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					//添加至currentRegistryProcessors
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					//标记该BeanDefinitionRegistryPostProcessor已经被处理
					processedBeans.add(ppName);
				}
			}
			//排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);			
			registryProcessors.addAll(currentRegistryProcessors);
			//调用BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			//清理currentRegistryProcessors
			currentRegistryProcessors.clear();

			// 处理其余BeanDefinitionRegistryPostProcessor直到没有新的BeanDefinitionRegistryPostProcessor出现
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			// 调用registryProcessors、regularPostProcessors中的postProcessBeanFactory
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// 调用beanFactoryPostProcessor中的postProcessBeanFactory
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// 获得beanFactory中bean形式的BeanFactoryPostProcessor
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				//保存PriorityOrder接口的BeanFactoryPostProcessor
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
            //保存Order接口的BeanFactoryPostProcessor
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
            //保存其余的BeanFactoryPostProcessor
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		//排序并且处理PriorityOrder接口的BeanFactoryPostProcessor
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		//排序并且处理Order接口的BeanFactoryPostProcessor
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		//处理Order接口的BeanFactoryPostProcessor
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
	}
```

注意两种形式的BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor：AbstractApplicationContext属性的，由beanFactory管理bean形式的。整体上先处理AbstractApplicationContext属性的然后再处理bean形式的。

- AbstractApplicationContext 属性的BeanFactoryPostProcessor通过addBeanFactoryPostProcessor添加新的BeanFactoryPostProcessor

AbstractApplicationContext：

```
	private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors = new ArrayList<>();
	
	
    public List<BeanFactoryPostProcessor> getBeanFactoryPostProcessors() {
		return this.beanFactoryPostProcessors;
	}
	
	public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor) {
		Assert.notNull(postProcessor, "BeanFactoryPostProcessor must not be null");
		this.beanFactoryPostProcessors.add(postProcessor);
	}	
```

BeanDefinitionRegistryPostProcessor为BeanFactoryPostProcessor的子类

```
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

- 记录beanFactoryPostProcessor的list
  - registryProcessors：记录AbstractApplicationContext属性中和beanFactory管理的bean的BeanDefinitionRegistryPostProcessor
  - regularPostProcessors：记录AbstractApplicationContext属性中的其余BeanFactoryPostProcessors

AbstractApplicationContext属性中BeanFactoryPostProcessor处理不保证有序，如果需要BeanFactoryPostProcessor的顺序，需要通过bean的形式并且接口实现PriorityOrder或者Order接口

### 注册BeanPostProcessor

```
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// 注册BeanPostProcessorChecker
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// 分别保存不同类型的BeanPostProcessor
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			//处理实现PriorityOrder接口的BeanPostProcessor
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			//处理实现Order接口的BeanPostProcessor			
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			//处理无序的BeanPostProcessor			
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		//对PriorityOrder接口的BeanPostProcessor进行排序并且注册
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// 排序、注册实现Order接口的BeanPostProcessor
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// 注册不需要排序的BeanPostProcessor
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// 排序、注册实现MergeBeanDefinitionPostProcessor接口
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// 注册ApplicationListenerDetector
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```

#### 如何理解注册？

先看两个方法的定义：

PostProcessorRegistrationDelegate#registerBeanPostProcessors：

```
	private static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {

		for (BeanPostProcessor postProcessor : postProcessors) {
			beanFactory.addBeanPostProcessor(postProcessor);
		}
	}
```

AbstractBeanFactory#addBeanPostProcessor：

```
	public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
		Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
		// 移除已经注册过的BeanPostProcessor
		this.beanPostProcessors.remove(beanPostProcessor);
		if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
			this.hasInstantiationAwareBeanPostProcessors = true;
		}
		if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
			this.hasDestructionAwareBeanPostProcessors = true;
		}
		this.beanPostProcessors.add(beanPostProcessor);
	}
```

从而可以得出：

- 注册BeanPostProcessor就是将BeanPostProcessor添加至beanFactory中属性beanPostProcessors列表中
- 注册的过程碰到之前已经注册的BeanPostProcessor会将其移除。所以添加MergedBeanDefinitionPostProcessor时会移除已有的BeanPostProcessor，beanFactory中的beanPostProcessor列表不存在重复

注册BeanPostProcessor的过程就是将有beanFactory管理的bean的BeanPostProcessor变为beanFactory的属性中的beanPostProcessor列表。

### 初始化消息资源

### 初始化ApplicationEventMulticaster

#### 使用示例

```
public class TestEvent extends ApplicationEvent {

    public String msg;

    public TestEvent(Object source) {
        super(source);
    }

    public TestEvent(Object source,String msg) {
        super(source);
        this.msg=msg;
    }

    public void print() {
        System.out.println(msg);
    }

}
```

```
public class TestListener implements ApplicationListener {


    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof TestEvent) {
            TestEvent testEvent = (TestEvent) event;
            testEvent.print();
        }
    }

}
```

```
	<bean id="testListener" class="com.zchuber.springsourcedeepdiving.ioc.TestListener" />
```

添加测试用例：

```
    @Test
    public void testMyTestEvent(){
        ApplicationContext ac=new ClassPathXmlApplicationContext("beanFactoryTest.xml");
        TestEvent event = new TestEvent("hello", "msg");
        ac.publishEvent(event);
    }
```

运行结果：

```
msg
```

#### 初始化ApplicationEventMulticaster

```
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		//beanFactory中包含applicationEventMulticaster
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			//获得applicationEventMulticaster
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
			//否则创建applicationEventMulticaster
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			//注册到beanFactory中
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
```

初始化的过程就是从工厂或者创建SimpleApplicationEventMulticaster然后赋值给AbstractApplicationContext的applicationEventMulticaster

SimpleApplicationEventMulticaster的主要方法为multicastEvent。

SimpleApplicationEventMulticaster#multicastEvent：

```
	//将事件发送给所有的listener
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

multicastEvent方法会将接收到的event发送给所以的监听器，在监听器中决定是否处理该event

### 注册监听器

```
	protected void registerListeners() {
		// 将AbstractApplicationContext属性中的ApplicationListener注册给ApplicationEventMulticaster
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// 将beanFactory中类型为ApplicationListener的bean注册给ApplicationEventMulticaster
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// 发布事件
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```

## 初始化非延迟加载单例

```
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		//为beanFactory设置ConversionService
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// 添加EmbeddedValueResolver
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// 实例化LoadTimeWeaverAware的bean
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		//不允许修改BeanDefinitions
		beanFactory.freezeConfiguration();

		//实例化所有的非lazy单例bean
		beanFactory.preInstantiateSingletons();
	}
```

#### ConversionService的设置

##### 使用示例1

```
public class String2DateConverter implements Converter<String, Date> {

    public Date convert(String source) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        try {
            return  sdf.parse(source);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

```
	<bean id="userManager"
		  class="com.zchuber.springsourcedeepdiving.ioc.UserManager"  >
		<property name="dataValue">
			<value>2022-06-01</value>
		</property>
	</bean>

	<bean id="conversionService" class="org.springframework.context.support.ConversionServiceFactoryBean" >
		<property name="converters">
			<list>
				<bean class="com.zchuber.springsourcedeepdiving.ioc.String2DateConverter" />
			</list>
		</property>
	</bean>
```

测试用例：

```
    @Test
    public void testStringDateConvertUserManager(){
        ApplicationContext ac=new ClassPathXmlApplicationContext("beanFactoryTest.xml");
        UserManager userManager = (UserManager) ac.getBean("userManager");
        System.out.println(userManager);
    }
```

运行结果：

```
dataValue: Wed Jun 01 00:00:00 CST 2022
```

ConverterService也可以起到和PropertyEditorSupport相同的效果

##### 使用示例2

```
    @Test
    public void testStringDateConvert(){
        DefaultConversionService conversionService = new DefaultConversionService();
        conversionService.addConverter(new String2DateConverter());
        String dateStr = "2022-06-03";
        Date date=conversionService.convert(dateStr, Date.class);
        System.out.println(date);
    }
```

运行结果：

```
Fri Jun 03 00:00:00 CST 2022
```

#### 冻结配置

```
	public void freezeConfiguration() {
		this.configurationFrozen = true;
		this.frozenBeanDefinitionNames = StringUtils.toStringArray(this.beanDefinitionNames);
	}
```

冻结BeanDefinition，其实就是设置configurationFrozen的属性为true。

#### 初始非延迟加载

spring 会将非lazy的单例bean提前初始化，这样有利于发现BeanDefinition中的错误

```
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		//实例化所有的非lazy单例bean
		for (String beanName : beanNames) {
			//合并BeanDefinition
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			//只处理非abstract、非lazy的单例bean
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		//调用smartSingleton的afterSingletonInstantiated
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

## finishRefresh

在该方法中完成初始化lifecycle的实现类，发布ContextRefreshedEvent

```
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

### initLifecycleProcessor

```
	protected void initLifecycleProcessor() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		//如果beanFactory中存在lifecycleProcessor
		if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
			//赋值给applicationContext的lifecycleProcessor属性
			this.lifecycleProcessor =
					beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
			}
		}
		else {
			//创建并赋值abstractApplicationContext的lifecycleProcessor属性，并注册到beanFactory中
			DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
			defaultProcessor.setBeanFactory(beanFactory);
			this.lifecycleProcessor = defaultProcessor;
			beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + LIFECYCLE_PROCESSOR_BEAN_NAME + "' bean, using " +
						"[" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
			}
		}
	}
```

beanFactory可以直接注册单例bean

### lifecycleProcessor中onRefresh

```
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}	
```

### publishEvent

```
publishEvent(new ContextRefreshedEvent(this));
```

发布ContextRefreshedEvent事件

