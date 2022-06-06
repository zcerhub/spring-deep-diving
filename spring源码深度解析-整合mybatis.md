# 整合MyBatis

## 独立使用Mybatis

使用mybatis需要经过经过如下的步骤：

#### 创建和数据库中表映射的Pojo

```
@Data
@NoArgsConstructor
@ToString
public class User {

    private Integer id;
    private String name;
    private Integer age;

    public User(String name, Integer age) {
        this.name=name;
        this.age=age;
    }


}
```

#### 创建Mapper接口

```
public interface UserMapper {

    void insertUser(User user);

    User getUser(Integer id);

}
```

#### 创建mybatis配置文件

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration> 
    <!-- 全局参数 --> 
	<settings>
		<!-- 设置但JDBC类型为空时,某些驱动程序要指定值,default:OTHER -->
		<setting name="jdbcTypeForNull" value="NULL"/> 
	</settings>

	<typeAliases>
		<typeAlias  alias="User" type="com.zchuber.springsourcedeepdiving.mybatis.User"/>
	</typeAliases>

	<environments default="development">
		<environment id="development">
			<transactionManager  type="jdbc" />
			<dataSource type="POOLED">
				<property name="driver" value="com.mysql.cj.jdbc.Driver"/>
				<property name="url" value="jdbc:mysql://192.168.152.5:3306/biyi_test?allowPublicKeyRetrieval=true"/>
				<property name="username" value="root"/>
				<property name="password" value="v3imYJ2@yL6Aq6Tu"/>
			</dataSource>
		</environment>
	</environments>

	<mappers>
		<mapper resource="mappers/userMapper.xml"/>
	</mappers>

</configuration>
```

#### 创建mapper对应的xml文件

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!-- mybatis TransactionDao mapper configuration -->
<mapper namespace="com.zchuber.springsourcedeepdiving.mybatis.UserMapper">

    <!-- flushCache to db configuration -->
    <insert id="insertUser" parameterType="User" >
        INSERT INTO user
        (name, age)
        VALUES (#{name}, #{age})
    </insert>

    <!-- in the examples with TransactionDao try useCache=false in this case cache won't be used and each time query the db-->
    <select id="getUser" resultType="User" parameterType="java.lang.Integer" >
        SELECT * FROM user where id=#{id}
    </select>



</mapper>
```

#### 测试用例

```
public class TestMyBatis {

    SqlSessionFactory ssf;

    {
        String resource = "mybatis-config.xml";
        Reader reader = new BufferedReader(
                new InputStreamReader(TestMyBatis.class.getClassLoader().getResourceAsStream(resource)));
        ssf= new SqlSessionFactoryBuilder().build(reader);
    }

	//利用mybatis插入元素
    @Test
    public void testInsertUser()  {
        SqlSession session = ssf.openSession();
        UserMapper userMapper = session.getMapper(UserMapper.class);
        User user = new User("tom", Integer.valueOf(5));
        userMapper.insertUser(user);
        session.commit();
        session.close();
    }
}    
```

利用mybatis查询元素

```
@Test
public void testQueryUser()  {
    SqlSession session = ssf.openSession();
    UserMapper userMapper = session.getMapper(UserMapper.class);
    User user=userMapper.getUser(1);
    System.out.println(user);
    session.close();
}
```

结果输出：

```
User(id=1, name=张三, age=20)
```

使用mybatis中最重要的两个类SqlSessionFactory和Mapper实现类。通过sqlsessionFactory获得sqlSession，利用sqlsession获得mapper接口的实现类

## Spring整合MyBatis

#### 使用示例

- 创建Pojo对象User

- 创建Mapper

- mybatis配置文件

  只保留typeAliases和mappers配置

  ```
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
  <configuration> 
  	<typeAliases>
  		<typeAlias  alias="User" type="com.zchuber.springsourcedeepdiving.mybatis.User"/>
  	</typeAliases>
  
  	<mappers>
  		<mapper resource="mappers/userMapper.xml"/>
  	</mappers>
  
  </configuration>
  ```

- spring配置文件

  ```
  <?xml version="1.0" encoding="UTF-8"?>
  
  <beans xmlns="http://www.springframework.org/schema/beans"
  	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  	   xsi:schemaLocation="http://www.springframework.org/schema/beans
          					 http://www.springframework.org/schema/beans/spring-beans.xsd">
  
  
  
  	<bean id="dataSource"
  		  class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
  		<property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
  		<property name="url" value="jdbc:mysql://192.168.152.5:3306/biyi_test?allowPublicKeyRetrieval=true"/>
  		<property name="username" value="root"/>
  		<property name="password" value="v3imYJ2@yL6Aq6Tu"/>
  		<property name="initialSize" value="1" />
  	</bean>
  
  	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean" >
  		//配置mybatis配置文件的路径
  		<property name="configLocation" value="classpath:mybatis-config-spring.xml"/>
  		//配置datasource
  		<property name="dataSource" ref="dataSource"/>
  	</bean>
  
  	<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" >
  		//配置mapperInterface的接口为UserMapper
  		<property name="mapperInterface" value="com.zchuber.springsourcedeepdiving.mybatis.UserMapper"/>
  		//配置sqlSessionFactory
  		<property name="sqlSessionFactory" ref="sqlSessionFactory"/>
  	</bean>
  
  </beans>
  
  ```

  配置SqlSessionFactoryBean、MapperFactoryBean

#### 测试用例

```
    @Test
    public void testSpringMybatis()  {
        ApplicationContext context = new ClassPathXmlApplicationContext("beanFactoryTest-mybatis.xml");
        UserMapper userDao = (UserMapper) context.getBean("userMapper");
        System.out.println(userDao.getUser(1));
    }
```

结果输出：

```
User(id=1, name=张三, age=20)
```

### Spring整合Mybatis原理分析

核心类为SqlSessionFactoryBean和MapperFactoryBean

#### SqlSessionFactoryBean

SqlSessionFactoryBean的继承体系如下：

![1654329879460](D:\学习\spring源码深度解析\assets\1654329879460.png)

SqlSessionFactoryBean实现了InitializingBean和FactoryBean接口。先查看InitializingBean的afterPropertiesSet方法

##### SqlSessionFactoryBean#afterPropertiesSet：

```
  @Override
  public void afterPropertiesSet() throws Exception {
  	//非空校验
    notNull(dataSource, "Property 'dataSource' is required");
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
              "Property 'configuration' and 'configLocation' can not specified with together");
	//创建sqlSessionFactory
    this.sqlSessionFactory = buildSqlSessionFactory();
  }
```

SqlSessionFactoryBean#buildSqlSessionFactory：

```
  protected SqlSessionFactory buildSqlSessionFactory() throws IOException {

    Configuration configuration;

    XMLConfigBuilder xmlConfigBuilder = null;
    //获得configuration对象
    if (this.configuration != null) {
     //已经存在configuration为其设置属性
      configuration = this.configuration;
      if (configuration.getVariables() == null) {
        configuration.setVariables(this.configurationProperties);
      } else if (this.configurationProperties != null) {
        configuration.getVariables().putAll(this.configurationProperties);
      }
      //利用XMLConfigBuilder解析配置文件获得configuration对象
    } else if (this.configLocation != null) {
      xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
      configuration = xmlConfigBuilder.getConfiguration();
    } else {
      //否则新建configuration对象
      configuration = new Configuration();
      if (this.configurationProperties != null) {
        configuration.setVariables(this.configurationProperties);
      }
    }
	
	//将sqlSessionFactoryBean的objectFactory属性设置给configuration
    if (this.objectFactory != null) {
      configuration.setObjectFactory(this.objectFactory);
    }

	//将sqlSessionFactoryBean的objectWrapperFactory属性设置给configuration
    if (this.objectWrapperFactory != null) {
      configuration.setObjectWrapperFactory(this.objectWrapperFactory);
    }

    if (this.vfs != null) {
      configuration.setVfsImpl(this.vfs);
    }
	
    //将sqlSessionFactoryBean的typeAliasesPackage属性设置给configuration
    if (hasLength(this.typeAliasesPackage)) {
      String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeAliasPackageArray) {
        configuration.getTypeAliasRegistry().registerAliases(packageToScan,
                typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for aliases");
        }
      }
    }
	//将sqlSessionFactoryBean的typeAliasse属性设置给configuration
    if (!isEmpty(this.typeAliases)) {
      for (Class<?> typeAlias : this.typeAliases) {
        configuration.getTypeAliasRegistry().registerAlias(typeAlias);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered type alias: '" + typeAlias + "'");
        }
      }
    }

	//将sqlSessionFactoryBean的plugins属性设置给configuration
    if (!isEmpty(this.plugins)) {
      for (Interceptor plugin : this.plugins) {
        configuration.addInterceptor(plugin);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered plugin: '" + plugin + "'");
        }
      }
    }

	//将sqlSessionFactoryBean的typeHandlersPackage属性设置给configuration
    if (hasLength(this.typeHandlersPackage)) {
      String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeHandlersPackageArray) {
        configuration.getTypeHandlerRegistry().register(packageToScan);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for type handlers");
        }
      }
    }
	//将sqlSessionFactoryBean的typeHandlers属性设置给configuration
    if (!isEmpty(this.typeHandlers)) {
      for (TypeHandler<?> typeHandler : this.typeHandlers) {
        configuration.getTypeHandlerRegistry().register(typeHandler);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered type handler: '" + typeHandler + "'");
        }
      }
    }
	//将sqlSessionFactoryBean的databaseIdProvider属性设置给configuration
    if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
      try {
        configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
      } catch (SQLException e) {
        throw new NestedIOException("Failed getting a databaseId", e);
      }
    }

	//将sqlSessionFactoryBean的cache属性设置给configuration
    if (this.cache != null) {
      configuration.addCache(this.cache);
    }
	//利用xmlConfigBuilder解析配置文件获得configuration
    if (xmlConfigBuilder != null) {
      try {
        xmlConfigBuilder.parse();

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed configuration file: '" + this.configLocation + "'");
        }
      } catch (Exception ex) {
        throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
      } finally {
        ErrorContext.instance().reset();
      }
    }

	//设置默认transactionFactory
    if (this.transactionFactory == null) {
      this.transactionFactory = new SpringManagedTransactionFactory();
    }
	//设置configuration的environment
    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

    if (!isEmpty(this.mapperLocations)) {
    	//解析mapper配置文件到configuration
      for (Resource mapperLocation : this.mapperLocations) {
        if (mapperLocation == null) {
          continue;
        }

        try {
          XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
              configuration, mapperLocation.toString(), configuration.getSqlFragments());
          xmlMapperBuilder.parse();
        } catch (Exception e) {
          throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
        } finally {
          ErrorContext.instance().reset();
        }

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed mapper file: '" + mapperLocation + "'");
        }
      }
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Property 'mapperLocations' was not specified or no matching resources found");
      }
    }
	//利用sqlsessionFactoryBuilder创建sqlsessionFactory
    return this.sqlSessionFactoryBuilder.build(configuration);
  }

```

SqlSessionFactoryBuilder#build：

```
  public SqlSessionFactory build(Configuration config) {
  	//利用configuration创建DefaultSqlSessionFactory
    return new DefaultSqlSessionFactory(config);
  }
```

##### SqlSessionFactoryBean#getObject

```
  public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
      afterPropertiesSet();
    }
	//返回创建的sqlSessionFactory
    return this.sqlSessionFactory;
  }
```

SqlSessionFactoryBean中有configuration中的属性，既可以通过mybatis的配置文件配置mybatis的属性，也可以通过配置SqlSessionFactoryBean的属性，SqlSessionFactoryBean会将mybatis配置文件和自己的属性创建configuration对象。

#### MapperFactoryBean

MapperFactoryBean的继承体系：

![1654331076357](D:\学习\spring源码深度解析\assets\1654331076357.png)

MapperFactoryBean实现InitializingBean、FactoryBean接口。

##### DaoSupport#afterPropertiesSet

```
	public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
		// 配置校验
		checkDaoConfig();

		// Let concrete implementations initialize themselves.
		try {
			initDao();
		}
		catch (Exception ex) {
			throw new BeanInitializationException("Initialization of DAO failed", ex);
		}
	}
```

MapperFactoryBean#checkDaoConfig：

```
  protected void checkDaoConfig() {
    //调用父类的校验方法
    super.checkDaoConfig();
	//属性非空校验
    notNull(this.mapperInterface, "Property 'mapperInterface' is required");
	//利用sqlSession获得configuration
    Configuration configuration = getSqlSession().getConfiguration();
    //configuration中mapperInterface为注册，在次注册
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }
```

SqlSessionDaoSupport#getSqlSession：

```
  public SqlSession getSqlSession() {
    return this.sqlSession;
  }
```

###### 何时给MapperFactoryBean中的sqlSession实例化？

配置MapperFactoryBean时设置了sqlSessionFactory的属性，会通过setter依赖注入sqlSessionFactory实例，此时会调用MapperFactoryBean的setSqlSessionFactory方法。

```
	<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" >
		<property name="mapperInterface" value="com.zchuber.springsourcedeepdiving.mybatis.UserMapper"/>
		<property name="sqlSessionFactory" ref="sqlSessionFactory"/>
	</bean>
```

SqlSessionDaoSupport#setSqlSessionFactory：

```
  public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
    if (!this.externalSqlSession) {
      //利用sqlSessionFactory创建sqlSession实例，然后赋值给sqlSession属性。
      this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```

##### MapperFactoryBean#getObject：

```
  public T getObject() throws Exception {
  	//利用sqlSession获取mapperInterface的实现类
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

### MapperScannerConfigure

如果需要注册大量的mapper到spring中，使用上述在spring配置文件中注册MapperFactoryBean的效率比较低。可以使用MapperScannerConfigurer扫描指定路径下的所有mapper接口然后注册到spring里

#### 使用示例

mybatis配置文件中添加MapperScannerConfigurer，：

```
	<bean  class="org.mybatis.spring.mapper.MapperScannerConfigurer" >
       <property name="basePackage" value="com.zchuber.springsourcedeepdiving.mybatis"/>
	</bean>
	
	<!-- 注释调userMapper -->
	<!--	<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" >
		<property name="mapperInterface" value="com.zchuber.springsourcedeepdiving.mybatis.UserMapper"/>
		<property name="sqlSessionFactory" ref="sqlSessionFactory"/>
	</bean>-->
```

测试用例：

```
    @Test
    public void testSpringMybatis()  {
        ApplicationContext context = new ClassPathXmlApplicationContext("beanFactoryTest-mybatis.xml");
        UserMapper userDao = (UserMapper) context.getBean("userMapper");
        System.out.println(userDao.getUser(1));
    }
```

运行结果：

```
User(id=1, name=张三, age=20)
```

即使没有定义userMapper，仍旧可以从beanFactory中获取userMapper的bean

#### MapperScannerConfigurer

![1654340518003](D:\学习\spring源码深度解析\assets\1654340518003.png)

MapperScannerConfigurer实现InitializingBean、BeanDefinitionRegistryPostProcessor接口。

#### MapperScannerConfigurer#afterPropertiesSet

```
  public void afterPropertiesSet() throws Exception {
  	//非空校验
    notNull(this.basePackage, "Property 'basePackage' is required");
  }
```

#### MapperScannerConfigurer#postProcessBeanFactory

```
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 空实现
  }
```

#### MapperScannerConfigurer#postProcessBeanDefinitionRegistry

主要的处理过程在postProcessBeanDefinitionRegistry中：

```
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
  	//如果processPropertyPlaceHolders为true，调用processPropertyPlaceHolder
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }
	
	//利用ClassPathMapperScanner进行扫描，并为其设置属性
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    //注册扫描或者不扫扫描规则
    scanner.registerFilters();
    //扫描指定包
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
```

##### processPropertyPlaceHolder有何作用？

###### processPropertyPlaceHolder使用示例

新建文件test.properties：

```
basePackage=com.zchuber.springsourcedeepdiving.mybatis
```

mybatis配置文件添加PropertyPlaceholderConfigurer的bean，并修改MapperScannerConfigurer的bean配置：

```
	<bean  class="org.mybatis.spring.mapper.MapperScannerConfigurer" >
		<property name="basePackage" value="${basePackage}"/>
	</bean>
	
	<bean id="mesHandler" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
		<property name="locations">
			<list>
				<value>test.properties</value>
			</list>
		</property>
	</bean>	
```

运行结果：

```
java.lang.IllegalArgumentException: Could not resolve placeholder 'basePackage' in value "${basePackage}"

	at org.springframework.util.PropertyPlaceholderHelper.parseStringValue(PropertyPlaceholderHelper.java:178)
	at org.springframework.util.PropertyPlaceholderHelper.replacePlaceholders(PropertyPlaceholderHelper.java:124)
```

调用MapperScannerConfigurer的postProcessBeanDefinitionRegistry时PropertyPlaceholderConfigurer还为的postProcessBeanFactory还为调用，导致basePackage属性变量未被替换。

在配置MapperScannerConfigurer的bean中设置processPropertyPlaceHolders为true可以解决该问题

```
		<property name="processPropertyPlaceHolders" value="true"/>
```

##### MapperScannerConfigurer#processPropertyPlaceHolders

```
private void processPropertyPlaceHolders() {
  Map<String, PropertyResourceConfigurer> prcs = applicationContext.getBeansOfType(PropertyResourceConfigurer.class);

  if (!prcs.isEmpty() && applicationContext instanceof ConfigurableApplicationContext) {
  	//从beanFactory中获得mapperscannerConfigurer的beanDefinition
    BeanDefinition mapperScannerBean = ((ConfigurableApplicationContext) applicationContext)
        .getBeanFactory().getBeanDefinition(beanName);

    //新建一个beanFactory，在该beanFactory中解析mapperScannerBean中的占位符
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    //注册mapperScannerBean到模拟工厂
    factory.registerBeanDefinition(beanName, mapperScannerBean);
	
	//调用PropertyResourceConfigurer的psotProcessBeanFactory，在该方法中解析占位符
    for (PropertyResourceConfigurer prc : prcs.values()) {
      prc.postProcessBeanFactory(factory);
    }
	//获得解析后的mapperScannerBean中的属性值
    PropertyValues values = mapperScannerBean.getPropertyValues();
	
	//给成员属性赋解析后的属性值
    this.basePackage = updatePropertyValue("basePackage", values);
    this.sqlSessionFactoryBeanName = updatePropertyValue("sqlSessionFactoryBeanName", values);
    this.sqlSessionTemplateBeanName = updatePropertyValue("sqlSessionTemplateBeanName", values);
  }
}
```

#### ClassPathMapperScanner#registerFilters

通过该方法可以决定scanner扫描类，或者跳过类的规则

```
  public void registerFilters() {
    boolean acceptAllInterfaces = true;

    // 添加扫描注解的IncludeFilter，被该注解修饰的类都算是合法的bean
    if (this.annotationClass != null) {
      addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
      acceptAllInterfaces = false;
    }

    // 添加扫描指定接口的IncludeFilter，该接口或者实现类都算是合法的bean
    if (this.markerInterface != null) {
      addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
        @Override
        protected boolean matchClassName(String className) {
          return false;
        }
      });
      acceptAllInterfaces = false;
    }
	//如果acceptAllInterfaces为true，所有的接口都是合法的bean
    if (acceptAllInterfaces) {
      // default include filter that accepts all classes
      addIncludeFilter(new TypeFilter() {
        @Override
        public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
          return true;
        }
      });
    }

    // 排查package-info的java类
    addExcludeFilter(new TypeFilter() {
      @Override
      public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        String className = metadataReader.getClassMetadata().getClassName();
        return className.endsWith("package-info");
      }
    });
  }

```

#### ClassPathBeanDefinitionScanner#scan

ClassPathMapperScanner的继承体系如下：

![1654342273265](D:\学习\spring源码深度解析\assets\1654342376784.png)

ClassPathMapperScanner的父类为ClassPathBeanDefinitionScanner。

ClassPathBeanDefinitionScanner#scan：

```
public int scan(String... basePackages) {
   int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
   //调用ClassPathMapperScanner中的重写方法
   doScan(basePackages);

   // Register annotation config processors, if necessary.
   if (this.includeAnnotationConfig) {
      AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
   }

   return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```

ClassPathMapperScanner#doScan：

```
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
  //调用父类的doScan
  Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

  if (beanDefinitions.isEmpty()) {
    logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
  } else {
    //处理扫描到的BeanDefinitions
    processBeanDefinitions(beanDefinitions);
  }

  return beanDefinitions;
}
```

ClassPathBeanDefinitionScanner#doScan：

```
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		//遍历所有的包名
		for (String basePackage : basePackages) {
		   //从指定包中扫描BeanDefinition
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

ClassPathScanningCandidateComponentProvider#findCandidateComponents：

```
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
   if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
      return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
   }
   else {
      //调用scanCandidateComponents进行扫描
      return scanCandidateComponents(basePackage);
   }
}
```

ClassPathScanningCandidateComponentProvider#scanCandidateComponents：

```
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
   Set<BeanDefinition> candidates = new LinkedHashSet<>();
   try {
      //补齐扫描路径
      String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
      //获得扫描包下所有的resource   
      Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
      for (Resource resource : resources) {
         if (resource.isReadable()) {
            try {
              //获得resource中的元数据
               MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
               //如果是满足条件则算是合法的
               if (isCandidateComponent(metadataReader)) {
                  ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                  sbd.setResource(resource);
                  sbd.setSource(resource);
                  //在ClassPathMapperScanner中重新该方法，返回true则是合法的BeanDefinition
                  if (isCandidateComponent(sbd)) {
                     if (debugEnabled) {
                        logger.debug("Identified candidate component class: " + resource);
                     }
                     candidates.add(sbd);
                  }
                  else {
                     if (debugEnabled) {
                        logger.debug("Ignored because not a concrete top-level class: " + resource);
                     }
                  }
               }
               else {
                  if (traceEnabled) {
                     logger.trace("Ignored because not matching any filter: " + resource);
                  }
               }
            }
            catch (Throwable ex) {
               throw new BeanDefinitionStoreException(
                     "Failed to read candidate component class: " + resource, ex);
            }
         }
         else {
            if (traceEnabled) {
               logger.trace("Ignored because not readable: " + resource);
            }
         }
      }
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
   }
   return candidates;
}
```

ClassPathScanningCandidateComponentProvider#isCandidateComponent：

ClassPathMapperScanner注册的filters在次生效了

```
	protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
		//利用excludeFilter进行过滤，匹配则返回false
		for (TypeFilter tf : this.excludeFilters) {
			if (tf.match(metadataReader, getMetadataReaderFactory())) {
				return false;
			}
		}
        //利用includeFilter进行过滤，匹配则返回true
		for (TypeFilter tf : this.includeFilters) {
			if (tf.match(metadataReader, getMetadataReaderFactory())) {
				return isConditionMatch(metadataReader);
			}
		}
		return false;
	}
```

ClassPathMapperScanner#isCandidateComponent：

```
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
  //BeanDefinition必须是接口，并且不能是内部接口
  return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
}
```

##### ClassPathMapperScanner#processBeanDefinitions：

该方法将扫描到Mapper的BeanDefinition修改为MapperFactoryBean的BeanDefinition，mapper接口的信息保存在MapperFactoryBean的BeanDefinition的构造方法中

```
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    //遍历所有的BeanDefinitions
    for (BeanDefinitionHolder holder : beanDefinitions) {
      //获得扫描到mapper的BeanDefinition
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      //将definition中的beanClassName（即Mapper的全路径，在示例中则为：com.zchuber.springsourcedeepdiving.mybatis.UserMapper）保存在definition的构造参数中
      definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
      //设置definition的beanClass为mapperFactoryBean
      definition.setBeanClass(this.mapperFactoryBean.getClass());

      definition.getPropertyValues().add("addToConfig", this.addToConfig);
	  
	  //向definition中为sqlSessionFactory的属性赋值
      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }
      
	  //向definition中为sqlSessionTemplate属性赋值
      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }
		
	  //设置definition的AutowireMode为AUTOWIRE_BY_TYPE
      if (!explicitFactoryUsed) {
        if (logger.isDebugEnabled()) {
          logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        }
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }

```

###### debug说明

![1654343904465](D:\学习\spring源码深度解析\assets\1654343904465.png)

修改扫描到的BeanDefinition，最终的结果和在xml中添加MapperFactoryBean的配置相同，只是更加的方便。

