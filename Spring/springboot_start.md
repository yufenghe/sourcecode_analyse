### SpringBoot启动流程

SpringBoot的启动从main方法开始：
```java
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```
整个流程从这一行代码开始，先执行SpringApplication的构造函数。
## initialize
此方法用于web环境的判断、初始化器、监听器的配置获取，是启动前的准备工作。
```java
private void initialize(Object[] sources) {
		if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
        //检测是否web环境，根据此参数决定使用上下文applicationContext
		this.webEnvironment = deduceWebEnvironment();
        //设置ApplicationContextInitializer实现类
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
        //设置监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        //获取主类，通过线程栈得到
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```
### deduceWebEnvironment
此方法通过判断当前环境中是否包含**javax.servlet.Servlet**,
**org.springframework.web.context.ConfigurableWebApplicationContext**这两个类来判断是否为web环境；
```java
private boolean deduceWebEnvironment() {
		for (String className : WEB_ENVIRONMENT_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return false;
			}
		}
		return true;
	}
```
### setInitializers和setListeners
其中Initializer和Listeners实现，是通过SpringFactoriesLoader加载器加载各jar包中`META-INF/spring.factories`文件的配置信息。
spring.factories配置结构如下：
```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer
```
### deduceMainApplicationClass
检索线程栈查找main方法，其结构图如下：
## run方法
```java
public ConfigurableApplicationContext run(String... args) {
	StopWatch stopWatch = new StopWatch();
	//启动计时器
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	FailureAnalyzers analyzers = null;
	configureHeadlessProperty();
	//获得SpringApplicationRunListener实例，然后使用SimpleApplicationEventMulticaster
	//ApplicationStartedEvent事件
	SpringApplicationRunListeners listeners = getRunListeners(args);
	listeners.starting();
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(
				args);
		//配置环境变量，加载application.properties配置文件信息，包括spring.profile.active配置
		//及相关配置文件的读取
		ConfigurableEnvironment environment = prepareEnvironment(listeners,
				applicationArguments);
		//banner打印
		Banner printedBanner = printBanner(environment);
		//实例化applicationContext，如果为web环境，则为AnnotationConfigEmbeddedWebApplicationContext，
		//否则为AnnotationConfigApplicationContext
		context = createApplicationContext();
		analyzers = new FailureAnalyzers(context);
		//进行applicationContext的装配工作，比如设置Evironment环境变量，beanNameGenerator和resourceLoader，
		//
		prepareContext(context, environment, listeners, applicationArguments,
				printedBanner);
		refreshContext(context);
		afterRefresh(context, applicationArguments);
		listeners.finished(context, null);
		stopWatch.stop();
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass)
					.logStarted(getApplicationLog(), stopWatch);
		}
		return context;
	}
	catch (Throwable ex) {
		handleRunFailure(context, listeners, analyzers, ex);
		throw new IllegalStateException(ex);
	}
}
```

### getRunListeners
获取spring.factories中关于`SpringApplicationRunListeners`的所有实现，即：
```java
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListen
```

### ApplicationArguments
将run方法传入的参数封装成此对象，用户环境变量的设置和context的设置。

### prepareEnvironment

```java
private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// 创建环境变量对象，如果是web环境，则创建StandardServletEnvironment，否则StandardEnvironment
		ConfigurableEnvironment environment = getOrCreateEnvironment();

		//1、将defaultProperties和commandLineArgs更新到ConfigurableEnvironment中，有则更新，没有则添加；
		//2、根据设置的spring.profiles.active属性设置active profiles
		configureEnvironment(environment, applicationArguments.getSourceArgs());

		//观察者模式：发出准备启动事件
		listeners.environmentPrepared(environment);
		if (!this.webEnvironment) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertToStandardEnvironmentIfNecessary(environment);
		}
		return environment;
	}

	protected void configureEnvironment(ConfigurableEnvironment environment,
			String[] args) {
		//1、将defaultProperties和commandLineArgs更新到ConfigurableEnvironment中，有则更新，没有则添加
		configurePropertySources(environment, args);
		//2、根据设置的spring.profiles.active属性设置active profiles
		configureProfiles(environment, args);
	}
```

### printBanner
打印springboot的banner，bannerMode默认为控制台打印Console
```java
private Banner printBanner(ConfigurableEnvironment environment) {
		if (this.bannerMode == Banner.Mode.OFF) {
			return null;
		}

		//设置资源加载器
		ResourceLoader resourceLoader = this.resourceLoader != null ? this.resourceLoader
				: new DefaultResourceLoader(getClassLoader());
		SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter(
				resourceLoader, this.banner);
			//banner为日志模式：即打印到日志文件中
		if (this.bannerMode == Mode.LOG) {
			return bannerPrinter.print(environment, this.mainApplicationClass, logger);
		}
		//默认为Console打印
		return bannerPrinter.print(environment, this.mainApplicationClass, System.out);
	}

```

### createApplicationContext
实例化spring应用上下文
```java
protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				//如果是web环境，则设置contextClass为org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext，
				//否则为org.springframework.context.annotation.AnnotationConfigApplicationContext
				contextClass = Class.forName(this.webEnvironment
						? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, "
								+ "please specify an ApplicationContextClass",
						ex);
			}
		}
		//实例化
		return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
	}
```

### prepareContext
准备上下文：设置上下文的环境变量
```java
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		context.setEnvironment(environment);
		//注册单例ConfigurationBeanNameGenerator，设置资源加载器resourceLoader
		postProcessApplicationContext(context);
		applyInitializers(context);

		//处理资context完成准备的事件
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}

		//省略

		// Load the sources
		Set<Object> sources = getSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		//加载bean到spring的beanFactory中
		load(context, sources.toArray(new Object[sources.size()]));

		//执行context加载完成事件
		listeners.contextLoaded(context);
	}
```
#### load
加载应用中的bean
```java
protected void load(ApplicationContext context, Object[] sources) {
	//创建BeanDefinitionLoader
		BeanDefinitionLoader loader = createBeanDefinitionLoader(
				getBeanDefinitionRegistry(context), sources);
		if (this.beanNameGenerator != null) {
			loader.setBeanNameGenerator(this.beanNameGenerator);
		}
		if (this.resourceLoader != null) {
			loader.setResourceLoader(this.resourceLoader);
		}
		if (this.environment != null) {
			loader.setEnvironment(this.environment);
		}
		loader.load();
	}
```

##### createBeanDefinitionLoader
```java
BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
		Assert.notNull(registry, "Registry must not be null");
		Assert.notEmpty(sources, "Sources must not be empty");
		this.sources = sources;
		//JavaConfig形式bean的读取
		this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
		//xml形式注册的bean的读取
		this.xmlReader = new XmlBeanDefinitionReader(registry);
		if (isGroovyPresent()) {
			this.groovyReader = new GroovyBeanDefinitionReader(registry);
		}
		//资源扫描器：从当前classPath中读取
		this.scanner = new ClassPathBeanDefinitionScanner(registry);
		//设置过滤项
		this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
	}
```
##### AnnotatedBeanDefinitionReader
javaConfig读取bean的注册，设置起始bean
```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
```

##### registerAnnotationConfigProcessors
主要功能时注册几个主要的bean：ConfigurationAnnotationBeanProcessor、AutowiredAnnotationBeanProcessor、
RequiredAnnotationBeanProcessor、CommonAnnotationBeanPostProcessor等BeanPostProcessor，用于bean实例的
前后置处理。
```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {

		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);

		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(4);

		//注册ConfigurationAnnotationBeanProcessor
		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		//注册AutowiredAnnotationBeanProcessor
		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		//注册RequiredAnnotationBeanProcessor
		if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// 注册CommonAnnotationBeanPostProcessor
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		//注册PersistenceAnnotationBeanPostProcessor
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		//注册EventListenerBeanProcessor
		if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}
		if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
		}

		return beanDefs;
	}
```
##### load方法
```java
private int load(Object source) {
		Assert.notNull(source, "Source must not be null");
		//source为应用的class对象执行此步
		if (source instanceof Class<?>) {
			return load((Class<?>) source);
		}
		if (source instanceof Resource) {
			return load((Resource) source);
		}
		if (source instanceof Package) {
			return load((Package) source);
		}
		if (source instanceof CharSequence) {
			return load((CharSequence) source);
		}
		throw new IllegalArgumentException("Invalid source type " + source.getClass());
	}

	private int load(Class<?> source) {
		//判断class是否有Component注解
		if (isComponent(source)) {
			//注册当前主类的bean对象到beanFactory中
			this.annotatedReader.register(source);
			return 1;
		}
		return 0;
	}
```
### refreshContext
进行应用内bean实例的创建、属性设置、针对bean扩展点的处理
```java
private void refreshContext(ConfigurableApplicationContext context) {
	 //调用父类AbstractApplicationContext的refresh方法
		refresh(context);
		//注册shutdownhook
		if (this.registerShutdownHook) {
			try {
				context.registerShutdownHook();
			}
			catch (AccessControlException ex) {
				// Not allowed in some environments.
			}
		}
	}
```

#### refresh

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
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
