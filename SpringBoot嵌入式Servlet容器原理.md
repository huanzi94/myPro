
1、嵌入式Servlet容器加载原理

1.1、ServletWebServerFactoryAutoConfiguration自动配置类。

		ServletWebServerFactoryAutoConfiguration是SpringBoot嵌入式Servlet容器的自动配置类。

    @Configuration
    @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
    /**
     * 当javax.servlet.ServletRequest类存在的时候，ServletWebServerFactoryAutoConfiguration才会生效。当我们不引入任何Servlet容器的时候，那么这个
     * javax.servlet.ServletRequest类就会找不到。在pom.xml中增加以下内容就可以证明这点：
     * <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-web</artifactId>
          <exclusions>
            <exclusion>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
          </exclusions>
        </dependency>
     */
    @ConditionalOnClass(ServletRequest.class)
    @ConditionalOnWebApplication(type = Type.SERVLET)
    // 将yml文件中所有的以server开头的配置加载到ServerProperties类中。比如：server.port=8081
    @EnableConfigurationProperties(ServerProperties.class)
    /**
     * BeanPostProcessorsRegistrar类实现了ImportBeanDefinitionRegistrar接口，这个接口中的方法会在SpringBoot启动的时候调用。
     * 默认导入EmbeddedTomcat、EmbeddedJetty、EmbeddedUndertow三种servlet容器。
     */
    @Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
          ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
          ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
          ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
    public class ServletWebServerFactoryAutoConfiguration {
    
       @Bean
       public ServletWebServerFactoryCustomizer servletWebServerFactoryCustomizer(ServerProperties serverProperties) {
          return new ServletWebServerFactoryCustomizer(serverProperties);
       }
    
       @Bean
       @ConditionalOnClass(name = "org.apache.catalina.startup.Tomcat")
       public TomcatServletWebServerFactoryCustomizer tomcatServletWebServerFactoryCustomizer(
             ServerProperties serverProperties) {
          return new TomcatServletWebServerFactoryCustomizer(serverProperties);
       }
    }

1.2、ServletWebServerFactoryCustomizer Bean

		ServletWebServerFactoryCustomizer Bean：通用的Servlet容器定制器，实现了WebServerFactoryCustomizer接口，重写了customize()方法，里面设置了Servlet容器的通用的配置。

    public class ServletWebServerFactoryCustomizer
          implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>, Ordered {
    
       private final ServerProperties serverProperties;
    
       public ServletWebServerFactoryCustomizer(ServerProperties serverProperties) {
          this.serverProperties = serverProperties;
       }
    
       @Override
       public int getOrder() {
          return 0;
       }
    
        // 设置了Servlet容器的通用的配置
       @Override
       public void customize(ConfigurableServletWebServerFactory factory) {
          PropertyMapper map = PropertyMapper.get().alwaysApplyingWhenNonNull();
          map.from(this.serverProperties::getPort).to(factory::setPort);
          map.from(this.serverProperties::getAddress).to(factory::setAddress);
          map.from(this.serverProperties.getServlet()::getContextPath).to(factory::setContextPath);
          map.from(this.serverProperties.getServlet()::getApplicationDisplayName).to(factory::setDisplayName);
          map.from(this.serverProperties.getServlet()::getSession).to(factory::setSession);
          map.from(this.serverProperties::getSsl).to(factory::setSsl);
          map.from(this.serverProperties.getServlet()::getJsp).to(factory::setJsp);
          map.from(this.serverProperties::getCompression).to(factory::setCompression);
          map.from(this.serverProperties::getHttp2).to(factory::setHttp2);
          map.from(this.serverProperties::getServerHeader).to(factory::setServerHeader);
          map.from(this.serverProperties.getServlet()::getContextParameters).to(factory::setInitParameters);
       }
    
    }

1.3、TomcatServletWebServerFactoryCustomizer Bean

		TomcatServletWebServerFactoryCustomizer Bean：Tomcat容器的定制器，实现了WebServerFactoryCustomizer接口，重写了customize()方法，里面设置了Tomcat容器的特有的配置。

    public class TomcatServletWebServerFactoryCustomizer
          implements WebServerFactoryCustomizer<TomcatServletWebServerFactory>, Ordered {
    
       private final ServerProperties serverProperties;
    
       public TomcatServletWebServerFactoryCustomizer(ServerProperties serverProperties) {
          this.serverProperties = serverProperties;
       }
    
        // 设置了Tomcat容器的特殊的配置
       @Override
       public void customize(TomcatServletWebServerFactory factory) {
           // 获取配置中所有的server.tomcat开头的配置
          ServerProperties.Tomcat tomcatProperties = this.serverProperties.getTomcat();
          if (!ObjectUtils.isEmpty(tomcatProperties.getAdditionalTldSkipPatterns())) {
             factory.getTldSkipPatterns().addAll(tomcatProperties.getAdditionalTldSkipPatterns());
          }
          if (tomcatProperties.getRedirectContextRoot() != null) {
             customizeRedirectContextRoot(factory, tomcatProperties.getRedirectContextRoot());
          }
          if (tomcatProperties.getUseRelativeRedirects() != null) {
             customizeUseRelativeRedirects(factory, tomcatProperties.getUseRelativeRedirects());
          }
       }
        ………省略其他代码……
    }

1.4、BeanPostProcessorsRegistrar.class

		ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class：实现了ImportBeanDefinitionRegistrar接口，调用registerBeanDefinitions方法以便提前注册 WebServerFactoryCustomizerPostProcessors，在SpringBoot启动的时候，经过一系列的调用，最终会通过这个WebServerFactoryCustomizerPostProcessors后置处理器找到TomcatServletWebServerFactoryCustomizer定制器去执行customize方法，定制Tomcat容器。

    public static class BeanPostProcessorsRegistrar implements ImportBeanDefinitionRegistrar, BeanFactoryAware {
    
       private ConfigurableListableBeanFactory beanFactory;
    
       @Override
       public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
          if (beanFactory instanceof ConfigurableListableBeanFactory) {
             this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
          }
       }
    
        // 注册WebServerFactoryCustomizerPostProcessors后置处理器
       @Override
       public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
             BeanDefinitionRegistry registry) {
          if (this.beanFactory == null) {
             return;
          }
          registerSyntheticBeanIfMissing(registry, "webServerFactoryCustomizerBeanPostProcessor",
                WebServerFactoryCustomizerBeanPostProcessor.class);
          registerSyntheticBeanIfMissing(registry, "errorPageRegistrarBeanPostProcessor",
                ErrorPageRegistrarBeanPostProcessor.class);
       }
    
        // 具体注册WebServerFactoryCustomizerPostProcessors后置处理器的逻辑。
       private void registerSyntheticBeanIfMissing(BeanDefinitionRegistry registry, String name, Class<?> beanClass) {
          if (ObjectUtils.isEmpty(this.beanFactory.getBeanNamesForType(beanClass, true, false))) {
             RootBeanDefinition beanDefinition = new RootBeanDefinition(beanClass);
             beanDefinition.setSynthetic(true);
             registry.registerBeanDefinition(name, beanDefinition);
          }
       }
    
    }

1.5、WebServerFactoryCustomizerBeanPostProcessor.class

		WebServerFactoryCustomizerBeanPostProcessor.class：实现了BeanPostProcessor接口，由SpringBoot内部调用。

    public class WebServerFactoryCustomizerBeanPostProcessor implements BeanPostProcessor, BeanFactoryAware {
    
       private ListableBeanFactory beanFactory;
    
       private List<WebServerFactoryCustomizer<?>> customizers;
    
       @Override
       public void setBeanFactory(BeanFactory beanFactory) {
          Assert.isInstanceOf(ListableBeanFactory.class, beanFactory,
                "WebServerCustomizerBeanPostProcessor can only be used " + "with a ListableBeanFactory");
          this.beanFactory = (ListableBeanFactory) beanFactory;
       }
    
        /**
         * 初始化WebServerFactoryCustomizerBeanPostProcessor之前做的一些事情
         */
       @Override
       public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
          if (bean instanceof WebServerFactory) {
             postProcessBeforeInitialization((WebServerFactory) bean);
          }
          return bean;
       }
    
        /**
         * 初始化WebServerFactoryCustomizerBeanPostProcessor之后做的一些事情
         */
       @Override
       public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
          return bean;
       }
    
       /**
        * LambdaSafe.callbacks().withLogger().invoke()：支持泛型回调回调的一种写法。
        * - getCustomizers：获取所有的实现了WebServerFactoryCustomizer的实现类集合。
        * - withLogger：这个就是记录日志的一个方法。
        * - invoke：实际上是循环调用getCustomizers返回来的实现类的customize方法。
        */
       @SuppressWarnings("unchecked")
       private void postProcessBeforeInitialization(WebServerFactory webServerFactory) {
          LambdaSafe.callbacks(WebServerFactoryCustomizer.class, getCustomizers(), webServerFactory)
                .withLogger(WebServerFactoryCustomizerBeanPostProcessor.class)
                .invoke((customizer) -> customizer.customize(webServerFactory));
       }
    
       // 返回WebServerFactoryCustomizer的实现类集合。
       private Collection<WebServerFactoryCustomizer<?>> getCustomizers() {
          if (this.customizers == null) {
             // Look up does not include the parent context
             this.customizers = new ArrayList<>(getWebServerFactoryCustomizerBeans());
             this.customizers.sort(AnnotationAwareOrderComparator.INSTANCE);
             this.customizers = Collections.unmodifiableList(this.customizers);
          }
          return this.customizers;
       }
    
        // 从beanFactory中获取WebServerFactoryCustomizer的实现类集合。
       @SuppressWarnings({ "unchecked", "rawtypes" })
       private Collection<WebServerFactoryCustomizer<?>> getWebServerFactoryCustomizerBeans() {
          return (Collection) this.beanFactory.getBeansOfType(WebServerFactoryCustomizer.class, false, false).values();
       }
    }

2、嵌入式Servlet容器启动原理（以EmbeddedTomcat为例）

		ServletWebServerFactoryAutoConfiguration在加载的时候，导入了三个内置Servlet容器，例如：EmbeddedTomcat

    @Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
          ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
          ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
          ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })

2.1、EmbeddedTomcat.class

		Tomcat容器的配置类，该类在常规自动配置类中@Import，以保证它的执行。

    @Configuration
    @ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
    @ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
    public static class EmbeddedTomcat {
    
        // 创建Tomcat容器服务工厂TomcatServletWebServerFactory Bean
       @Bean
       public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
          return new TomcatServletWebServerFactory();
       }
    }

2.2、TomcatServletWebServerFactory Bean

		TomcatServletWebServerFactory是用来创建TomcatWebServer的抽象web服务的抽象工厂类。此工厂类默认创建监听端口8080的HTTP请求的容器。

    public class TomcatServletWebServerFactory extends AbstractServletWebServerFactory
          implements ConfigurableTomcatWebServerFactory, ResourceLoaderAware {
        @Override
    	public WebServer getWebServer(ServletContextInitializer... initializers) {
    		/**
    		   这里先new了一个Tomcat对象，默认端口8080，请求地址localhost
                public class Tomcat {
                    protected int port = 8080;
                    protected String hostname = "localhost";
                    ……………省略其他代码……………
               }
    		 */
            Tomcat tomcat = new Tomcat();
    		File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
    		tomcat.setBaseDir(baseDir.getAbsolutePath());
    		Connector connector = new Connector(this.protocol);
    		tomcat.getService().addConnector(connector);
    		customizeConnector(connector);
    		tomcat.setConnector(connector);
    		tomcat.getHost().setAutoDeploy(false);
    		configureEngine(tomcat.getEngine());
    		for (Connector additionalConnector : this.additionalTomcatConnectors) {
    			tomcat.getService().addConnector(additionalConnector);
    		}
    		prepareContext(tomcat.getHost(), initializers);
            // 根据tomcat对象调用getTomcatWebServer获取TomcatWebServe的具体实例。
    		return getTomcatWebServer(tomcat);
    	}
        
        // 实际上new了一个TomcatWebServer对象
        protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
    		return new TomcatWebServer(tomcat, getPort() >= 0);
    	}
    }

2.3、TomcatWebServer

		用于创建Tomcat web服务器的WebServer。通常，这个类应该由TomcatServletWebServerFactory来调用创建，而不是直接创建。

    public class TomcatWebServer implements WebServer {
        // new TomcatWebServer对象的时候，调用initialize()来启动Tomcat
        public TomcatWebServer(Tomcat tomcat, boolean autoStart) {
            Assert.notNull(tomcat, "Tomcat Server must not be null");
            this.tomcat = tomcat;
            this.autoStart = autoStart;
            initialize();
        }
    
        private void initialize() throws WebServerException {
            logger.info("Tomcat initialized with port(s): " + getPortsDescription(false));
            synchronized (this.monitor) {
                …………省略其他代码…………
                try {
                    // 启动Tomcat服务
                     // Start the server to trigger initialization listeners
                    this.tomcat.start();
                    // Unlike Jetty, all Tomcat threads are daemon threads. We create a
                    // blocking non-daemon to stop immediate shutdown
                    startDaemonAwaitThread();
                }
                catch (Exception ex) {
                    stopSilently();
                    destroySilently();
                    throw new WebServerException("Unable to start embedded Tomcat", ex);
                }
            }
    }

3、使用外部Servlet容器，以Tomcat为例。

- 更改打包方式为war
      <packaging>war</packaging>
- 将SpringBoot内置的Tomcat容器的依赖排除掉，让它不参与打包，因为外置的Tomcat容器中已经有相关的jar。

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <scope>provided</scope>
    </dependency>

- 选择在IDEA总选择用Tomcat启动。

3.1、Servlt三大组件及注册方式

3.1.1、Servlt三大组件：Servlet、Filter、Listener

3.1.2、以Servelt为例，讲解在SpringBoot中注册Servelt的方式

- 基于Servlet3.0的规范注册Servlet，如果是以SpringBoot的启动类启动，那么必须在启动类上加@ServletComponentScan才能扫描到consumerServlet
      // Servlet3.0规范注册Servlet
      @WebServlet(name = "consumerServlet", urlPatterns = "/consumerServlet")
      // Servlet3.0规范注册Listener
      // @WebListener
      // Servlet3.0规范注册Filter
      // @WebFilter
      public class ConsumerServlet extends HttpServlet {
          @Override
          protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
              resp.getWriter().write("@WebServlet");
          }
      }
- 基于SpringBoot注解注册Servlet
      public class ConsumerSpringBootServlet extends HttpServlet {
      
          @Override
          protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
              resp.getWriter().write("ConsumerSpringBootServlet");
          }
      }
      
      @Configuration
      public class ConsumerConfiguration implements WebMvcConfigurer {
          @Bean
          public ServletRegistrationBean registrationBean() {
              ServletRegistrationBean registrationBean = new ServletRegistrationBean();
              registrationBean.setServlet(new ConsumerSpringBootServlet());
              registrationBean.addUrlMappings("/consumerSpringBootServlet");
              registrationBean.setName("consumerSpringBootServlet");
              return registrationBean;
          }
      }
      
      // 如果以SPringBoot启动类启动的项目，那么这个类可以不用。如果是以外部tomcat的方式启动，那么必须要有这个类，才能在外部tomcat启动的方式下，启动SpringBoot。
      public class TomcatSupportSpringBoot extends SpringBootServletInitializer {
          @Override
          protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
              return builder.sources(PractiseApplication.class);
          }
      }
