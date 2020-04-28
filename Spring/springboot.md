## 1、 @SpringBootApplication注解分析

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { 
@Filter(type = FilterType.CUSTOM,classes = TypeExcludeFilter.class),
@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
```

列举下各个注解的含义：
1、@Target Target通过ElementType来指定注解可使用范围的枚举集合（FIELD/METHOD/PARAMETER...）
2、@Retention Retention(保留)注解说明,这种类型的注解会被保留到那个阶段. 有三个值:
​	RetentionPolicy.SOURCE —— 这种类型的Annotations只在源代码级别保留,编译时就会被忽略
​	RetentionPolicy.CLASS —— 这种类型的Annotations编译时被保留,在class文件中存在,但JVM将会忽略
​	RetentionPolicy.RUNTIME —— 这种类型的Annotations将被JVM保留,所以他们能在运行时被JVM或其他使
3、@Documented 注解表明这个注解应该被 javadoc工具记录. 默认情况下,javadoc是不包括注解的. 但如果声明注解时指定了 @Documented,则它会被 javadoc 之类的工具处理, 所以注解类型信息也会被包括在生成的文档中
4、@Inherited 允许子类继承父类的注解，仅限于**类**注解有用，对于方法和属性无效。
5、@SpringBootConfiguration 注解实际上和@Configuration有相同的作用，配备了该注解的类就能够以JavaConfig的方式完成一些配置，可以不再使用XML配置。
6、@ComponentScan 这个注解完成的是自动扫描的功能，相当于Spring XML配置文件中的：、、\<context:component-scan\>,可使用basePackages属性指定要扫描的包，及扫描的条件。如果不设置则默认扫描@ComponentScan注解所在类的同级类和同级目录下的所有类，所以我们的Spring Boot项目，一般会把入口类放在顶层目录中，这样就能够保证源码目录下的所有类都能够被扫描到。
7、@EnableAutoConfiguration 这个注解是让Spring Boot的配置能够如此简化的关键性注解。



## 2、 具体源码实现
最初的入口是:
```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
```

### 2.1、实例化SpringApplication对象
```java
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		//1、初始化资源加载器
		this.resourceLoader = resourceLoader;
		//2、校验启动类，可以制指定多个
		Assert.notNull(primarySources, "PrimarySources must not be null");
		//3、初始化资源类集合，并去重
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		//4、判断应用类型：NONE；SERVLET；REACTIVE
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		//5、设置应用初始化上下文容器
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		//6、设置监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		//7、推断出项目启动类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

#### 2.1.1、设置上下文容器和监听器的 getSpringFactoriesInstances 方法
```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
		//1、获得当前的类加载器，如果没有的话，得到默认的类加载器
		ClassLoader classLoader = getClassLoader();
		//2、从META-INF/spring.factories 下读取对应的类配置的默认初始化的类
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		//3、把第二步中拿到的className的set初始化成对应的类
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
		//4、排个序
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
```




### 2.2、run方法
```java
public ConfigurableApplicationContext run(String... args) {
		//1、计时监控类
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		
		//2、初始化应用上下文和异常报告集合
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		
		//3、去除 java.awt.headless 模式
		configureHeadlessProperty();
		
		// 4、创建所有 Spring 运行监听器并发出开始执行的事件
		//getRunListeners 方法目测会得到如下的运行监听器
		//org.springframework.boot.context.event.EventPublishingRunListener
		//并启动它
		//关于SpringApplicationRunListeners 和 SpringApplicationRunListener 和 ApplicationListener 三者之间的关系，可以参考别的文章
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		
		try {
			//5、初始化一些main方法的入参，args不能为null
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			
			// 6、根据SpringApplicationRunListeners和应用参数来准备 Spring 环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			
			// 7、准备Banner打印器 - 就是启动Spring Boot的时候打印在console上的ASCII艺术字体
			Banner printedBanner = printBanner(environment);
			// 8、创建Spring上下文
            //要么走：自定义ApplicationContext的实现类
			//要么走：根据当前应用的类型webApplicationType来匹配对应的ApplicationContext，是servlet、reactive或者非web应用
			context = createApplicationContext();
			// 9、准备异常报告器
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			// 10、Spring上下文前置处理
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			// 11、刷新Spring上下文
			refreshContext(context);
			// 12、Spring上下文后置处理
			afterRefresh(context, applicationArguments);
			// 13、停止计时监控类
			stopWatch.stop();
			// 14、输出日志记录执行主类名、时间信息
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			// 15、发布应用上下文启动完成事件
			listeners.started(context);
			// 16、执行所有 Runner 运行器
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			// 17、发布应用上下文就绪事件
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		// 18、返回应用上下文
		return context;
	}
```





### 2.2.1、prepareEnvironment方法的具体实现
```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// 1、创建一个environment对象，根据是SERVLET还是REACTIVE还是NONE
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		//2、配置environment，配置PropertySources和activeProfiles
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		//3、配置PropertySources对它自己的递归依赖
		ConfigurationPropertySources.attach(environment);
	    //4、listeners环境准备(就是广播ApplicationEnvironmentPreparedEvent事件)。
    	listeners.environmentPrepared(environment);
        //5、将环境绑定到SpringApplication
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
		}
    	//6、配置PropertySources对它自己的递归依赖 和 3 一致，不知道为啥要整两份
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```


### 2.2.2、第16步中执行所有 Runner 运行器
Runner 运行器用于在服务启动时进行一些业务初始化操作，这些操作只在服务启动后执行一次。
Spring Boot提供了ApplicationRunner和CommandLineRunner两种服务接口CommandLineRunner、ApplicationRunner

####  对比：

**相同点 **

两者均在服务启动完成后执行，并且只执行一次。
两者都能获取到应用的命令行参数。
两者在执行时机上是一致的（可以通过Ordered相关的接口或注解来实现自定义执行优先级。）。

**不同点**

虽然两者都是获取到应用的命令行参数，但是ApplicationRunner获取到的是封装后的ApplicationArguments对象，而CommandLine获取到的是ApplicationArguments中的sourceArgs属性（List<String>）,即原始参数字符串列表（命令行参数列表）。



https://segmentfault.com/a/1190000020359093?utm_source=tag-newest







