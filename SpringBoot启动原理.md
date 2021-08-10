一、SpringBoot启动原理思维导图

启动类的main方法，调用SpringApplication的run方法

SpringApplication的run方法：

1、初始化SpringApplication。

1.1、从spring.factories中获取ApplicationContextInitializer。

1.2、从spring.factories中获取ApplicationListener监听器。

2、运行run方法。

     2.1、读取并设置环境变量信息，如：configureHeadlessProperty。配置信息：。

     2.2、创建SpringBoot上下文：AnnotationConfigServletWebServerApplicationContext。

     2.3、准备SpringBoot上下文：

          -  循环调用1.1读取出来的ApplicationContextInitializer的initialize方法。

          - 发布一系列的监听器事件：比如ApplicationContextInitializedEvent监听器事件。

     2.4、刷新SpringBoot上下文：实际上就是调用了refresh方法：

          - 初始化并创建后置处理器。

          - 初始化国际化配置信息。

          - 发布一系列的监听器事件。

          - 获取Servlet容器并启动，比如：Tomcat。

    2.5、一些其他的操作。

二、XXXApplication启动类

    @SpringBootApplication
    @ServletComponentScan
    public class PractiseApplication {
        public static void main(String[] args) {
            SpringApplication.run(PractiseApplication.class, args);
        }
    }

三、SpringApplication.run方法

    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
       return run(new Class<?>[] { primarySource }, args);
    }
    
    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        // 1、new一个SpringApplication
        // 2、调用run方法
    	return new SpringApplication(primarySources).run(args);
    }

3.1、初始化SpringApplication

    // SpringApplication的构造函数
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    	this.resourceLoader = resourceLoader;
    	Assert.notNull(primarySources, "PrimarySources must not be null");
        // 将启动类放入primarySources
    	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        // 判断我们的服务是属于哪种App类型的：SERVLET/NONE/REACTIVE
    	this.webApplicationType = WebApplicationType.deduceFromClasspath();
        // 从spring.factories中获取ApplicationContextInitializer类型的全限定类路径。
    	setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 从spring.factories中获取ApplicationListener类型的全限定类路径。
    	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        // 获取栈信息，推算含有main方法的启动类，也就是XXXApplication启动类
    	this.mainApplicationClass = deduceMainApplicationClass();
    }

3.2、启动SpringBoot

    // SpringApplication的run方法函数
    public ConfigurableApplicationContext run(String... args) {
    	StopWatch stopWatch = new StopWatch();
    	stopWatch.start();
    	ConfigurableApplicationContext context = null;
    	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        // 设置java.awt.headless这个参数，如果我们的服务器缺失了部分外设，比如显示器，那么也需要允许服务启动。
    	configureHeadlessProperty();
        // 从spring.factories中获取SpringApplicationRunListener类型的组件，用来发布事件
    	SpringApplicationRunListeners listeners = getRunListeners(args);
        // 发布ApplicationStartingEvent监听器事件，表示SpringBoot开始启动
    	listeners.starting();
    	try {
    		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            // 获取JAVA环境变量和系统环境变量（基于监听器）
    		ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
            // 配置是否忽略所有继承BeanInfo的Bean
    		configureIgnoreBeanInfo(environment);
            // 控制台打印SpringBoot默认的banner
    		Banner printedBanner = printBanner(environment);
            // 创建SpringBoot上下文，实际上就是创建AnnotationConfigServletWebServerApplicationContext上下文
    		context = createApplicationContext();
            // 从spring.factories中获取SpringBootExceptionReporter类型的全限定类路径,主要是用于支持SpringApplication启动错误的自定义报告
    		exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
    				new Class[] { ConfigurableApplicationContext.class }, context);
            // 准备SpringBoot上下文： 创建了一些后置增强器，循环调用了ApplicationContextInitializer类型的类的initialize方法，发布了ApplicationContextInitializedEvent，ApplicationPreparedEvent监听器事件，将Beans加载到应用程序上下文中等。
    		prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            // 刷新SpringBoot上下文，也就是加载Spring IOC容器：
    		refreshContext(context);
             // 在刷新上下文后调用,这是一个空实现
    		afterRefresh(context, applicationArguments);
    		stopWatch.stop();
    		if (this.logStartupInfo) {
    			new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
    		}
    		listeners.started(context);
    		callRunners(context, applicationArguments);
    	}
    	catch (Throwable ex) {
    		handleRunFailure(context, ex, exceptionReporters, listeners);
    		throw new IllegalStateException(ex);
    	}
    	try {
    		listeners.running(context);
    	}
    	catch (Throwable ex) {
    		handleRunFailure(context, ex, exceptionReporters, null);
    		throw new IllegalStateException(ex);
    	}
    	return context;
    }

- prepareContext方法

    private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
    			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
        	// 将环境信息设置到spring上下文中
    		context.setEnvironment(environment);
        	// 获取上下文后置处理器，
    		postProcessApplicationContext(context);
        	// 循环调用了ApplicationContextInitializer类型的类的initialize方法
    		applyInitializers(context);
        	// 发布了ApplicationContextInitializedEvent
    		listeners.contextPrepared(context);
    		if (this.logStartupInfo) {
    			logStartupInfo(context.getParent() == null);
    			logStartupProfileInfo(context);
    		}
    		// 获取了spring上下文的BeanFactory,用来负责创建Bean
    		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        	// 将applicationArguments以单例模式注册到BeanFactory中
    		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    		if (printedBanner != null) {
                // 将printedBanner以单例模式注册到BeanFactory中
    			beanFactory.registerSingleton("springBootBanner", printedBanner);
    		}
    		if (beanFactory instanceof DefaultListableBeanFactory) {
                // 设置相同的Bean是否允许覆盖，在spring中，同名的Bean会被后者覆盖(@Bean会覆盖其他的同名Bean),在SpringBoot中，默认是不允许覆盖的，即不允许出现同名的Bean。通过spring.main.allow-bean-definition-overriding=true设置
    			((DefaultListableBeanFactory) beanFactory)
    					.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    		}
    		// 获取到主启动类，主启动类也就是配置类：XXXApplication启动类
    		Set<Object> sources = getAllSources();
    		Assert.notEmpty(sources, "Sources must not be empty");
        	// 读取主启动类，后续会根据主启动类读取配置的Bean,比如：主启动类中的配置中有一个@ComponentScan注解
    		load(context, sources.toArray(new Object[0]));
        	// 读取完配置类后发布ApplicationPreparedEvent监听器事件
    		listeners.contextLoaded(context);
    	}

- refresh方法

    public void refresh() throws BeansException, IllegalStateException {
            synchronized(this.startupShutdownMonitor) {
                // 设置启动时间，是否激活状态标志，加载了有一系列的PropertySources文件等
                this.prepareRefresh();
                // 获得BeanFactory
                ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
                // 给beanFactory设置了一些属性，注册了一些组件。
                this.prepareBeanFactory(beanFactory);
    
                try {
                    // 在设置的basePackages包路径下边扫描并注册bean
                    this.postProcessBeanFactory(beanFactory);
                    // 自动配置类就是在这个时候加载的
                    this.invokeBeanFactoryPostProcessors(beanFactory);
                    // 注册一系列的BeanPostProcessor类型的Bean
                    this.registerBeanPostProcessors(beanFactory);
                    // 初始化国际化的资源信息并放入容器中
                    this.initMessageSource();
                    // 注册ApplicationEventMulticaster
                    this.initApplicationEventMulticaster();
                    // 调用了ServletWebServerApplicationContext的onRefresh，创建了内置的Servlet容器（Tomcat），并启动。
                    this.onRefresh();
                    // 将容器中的事件监听器添加到事件广播中，并发布一系列的早期事件
                    this.registerListeners();
                    // 创建并实例化所有的Bean
                    this.finishBeanFactoryInitialization(beanFactory);
                    // 清除资源文件缓存，调用生命周期处理器的onRefresh，发布一些监听器事件
                    this.finishRefresh();
                } catch (BeansException var9) {
                    if (this.logger.isWarnEnabled()) {
                        this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                    }
    			   // 销毁已创建的单例，以避免挂起资源。
                    this.destroyBeans();
                    // 重置激活状态标志为false。
                    this.cancelRefresh(var9);
                    throw var9;
                } finally {
                    // 清除缓存
                    this.resetCommonCaches();
                }
    
            }
        }
    

- prepareContext方法

    private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
    			ApplicationArguments applicationArguments) {
    		// 根据之前获取到的webApplicationType创建一个ConfigurableEnvironment
    		ConfigurableEnvironment environment = getOrCreateEnvironment();
        	// 将命令行参数读取到环境变量中
    		configureEnvironment(environment, applicationArguments.getSourceArgs());
        	// 发布ApplicationEnvironmentPreparedEvent监听器事件：这个监听器就会去读取约定的配置文件，包括全局配置文件application.yml
    		listeners.environmentPrepared(environment);
        	// 将application.yml中spring.mian开头的配置绑定到SpringApplication中。
    		bindToSpringApplication(environment);
    		if (!this.isCustomEnvironment) {
    			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
    					deduceEnvironmentClass());
    		}
        	// 根据environment更新配置PropertySources文件
    		ConfigurationPropertySources.attach(environment);
    		return environment;
    }

总结：

1、初始化了SpringApplication：读取了ApplicationListener, 

2、运行run()：

- 读取环境变量信息、配置信息……
- 创建了SpringApplication 上下文AnnotationConfigServletWebServerApplicationContext。
- 准备上下文：读取启动类。
- 调用refresh()：加载IOC容器、创建了Servlet容器、获取了所有的自动配置类……

3、期间发布了一系列的监听器，对外进行了扩展。
