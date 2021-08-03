
一、WebMvcAutoConfiguration类

		上一篇博客已经讲解了SpringBoot的自动配置原理，其中，SpringBoot自动配置的过程中会帮我们加载一个叫做WebMvcAutoConfiguration的类，这个类就是SpringMVC的自动配置类。那么如果在SpringBoot下边开发一个MVC的WEB应用，那么必须了解这类的原理，才能得心应手的处理静态资源的问题，即使我们的SpringBoot项目仅作为一个纯后台的应用。

WebMvcAutoConfiguration类的源码如下：

    @Configuration
    // 当我们的应用是一个基于servlet的Web应用程序时，这个自动配置才会生效。
    @ConditionalOnWebApplication(
        type = Type.SERVLET
    )
    @ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
    // 当IOC容器中没有这个WebMvcConfigurationSupport.class时，这个自动配置才会生效。所以我们如果想让SPringBoot自动帮我们处理资源问题的时候，我们不能自己去创建一个基于WebMvcConfigurationSupport类的实现。同时也不能使用@EnableWebMvc去注解一个我们自己创建的组件，因为@EnableWebMvc中导入了一个实现了WebMvcConfigurationSupport类的DelegatingWebMvcConfiguration.class类。
    @ConditionalOnMissingBean({WebMvcConfigurationSupport.class})
    @AutoConfigureOrder(-2147483638)
    // 该注解会保障WebMvcAutoConfiguration类加载完之后再去初始化它引入的几个类，也就是说主类加载完之后才能做一些事情，可以用这个注解。
    @AutoConfigureAfter({DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class})
    public class WebMvcAutoConfiguration {
    			……省略其他代码……
    }

二、WebMvcConfigurer类

		WebMvcAutoConfiguration中有一个实现了WebMvcConfigurer类的WebMvcAutoConfigurationAdapter类，已经帮我们实现了一些默认的功能，比如：默认的视图解析器配置，LocaleResolver配置等，如果实现了WebMvcConfigurer接口，就可以扩展SpringMVC的一些功能。

    @Configuration
    @Import(EnableWebMvcConfiguration.class)
    @EnableConfigurationProperties({ WebMvcProperties.class, ResourceProperties.class })
    @Order(0)
    public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer, ResourceLoaderAware {
        ……省略其他代码……
    }

1、@Import(EnableWebMvcConfiguration.class)注解

		EnableWebMvcConfiguration.class的父类中有一个@Autowired注解的方法，那么他就会将所有的实现了WebMvcConfigurer接口的类从容器中拿出来添加到configurer变量中保存起来。

    @Configuration
    public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
        private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
        @Autowired(
            required = false
        )
        public void setConfigurers(List<WebMvcConfigurer> configurers) {
            if (!CollectionUtils.isEmpty(configurers)) {
                this.configurers.addWebMvcConfigurers(configurers);
            }
        }
    }

		configurer变量底层实际上就是将这些configurers保存在一个List<WebMvcConfigurer>的delegates变量里面，当我们在实现了WebMvcConfigurer接口的类中调用WebMvcConfigurer方法的时候，就是循环delegates变量，依次调用。

    @Configuration
    public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {
        ……省略其他代码……
    }
    
    @Configuration
    public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
        private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
        public void setConfigurers(List<WebMvcConfigurer> configurers) {
            if (!CollectionUtils.isEmpty(configurers)) {
                // 调用addWebMvcConfigurers()方法
                this.configurers.addWebMvcConfigurers(configurers);
            }
        }
        ……省略其他代码……
    }
    
    class WebMvcConfigurerComposite implements WebMvcConfigurer {
        private final List<WebMvcConfigurer> delegates = new ArrayList();
        // addWebMvcConfigurers()方法实现，将所有的configurers添加到delegates变量中。
        public void addWebMvcConfigurers(List<WebMvcConfigurer> configurers) {
            if (!CollectionUtils.isEmpty(configurers)) {
                this.delegates.addAll(configurers);
            }
        }
         ……省略其他代码……
    }
     
    class WebMvcConfigurerComposite implements WebMvcConfigurer {	
        // 调用addInterceptors()方式时，就是循环delegates变量，依次调用。
        public void addInterceptors(InterceptorRegistry registry) {
                Iterator var2 = this.delegates.iterator();
    
                while(var2.hasNext()) {
                    WebMvcConfigurer delegate = (WebMvcConfigurer)var2.next();
                    delegate.addInterceptors(registry);
                }
    
            }
    }

三、视图解析器

1、InternalResourceViewResolver默认的视图解析器

2、BeanNameViewResolver

3、ContentNegotiatingViewResolver

四、静态资源配置原理

1、LocaleResolver获取语言方式配置

    @Bean
    @ConditionalOnMissingBean
    /**
     * 如果设置了yml文件spring.mvc.locale才会进来执行。
     * spring:
     *   mvc:
     *     locale-resolver: fixed(获取固定参数)/accept_header(从请求头中获取:Accept-Language)
     *	   // 如果设置为fiexd，那么直接会根据这里配置的locale获取语种，如果这里没有配置，那么直接会被Accept-Language覆盖
     *     locale: zh_CN 
     */
    @ConditionalOnProperty(prefix = "spring.mvc", name = "locale")
    public LocaleResolver localeResolver() {
        // 如果当前配置的是fiexd的，那么直接获取spring.mvc.locale的值
       if (this.mvcProperties.getLocaleResolver() == WebMvcProperties.LocaleResolver.FIXED) {
          return new FixedLocaleResolver(this.mvcProperties.getLocale());
       }
       // 如果设置的是accept_header，则获取设置的spring.mvc.locale的值。并且以后每一次请求都会调用AcceptHeaderLocaleResolver类中的
       // resolveLocale去获取请求头中的Accept-Language的值.
       AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
       localeResolver.setDefaultLocale(this.mvcProperties.getLocale());
       return localeResolver;
    }

1.1、LocaleResolver的四种实现

- CookieLocaleResolver：将语言存放在cookie中，这样每次请求都从cokkie获取。
- SessionLocaleResolver：将语言存放在session中，这样每次请求都从session获取。
- FixedLocaleResolver：直接会根据yml中配置的spring.mvc.locale获取语种，如果这里没有配置，那么直接会被Accept-Language覆盖，如果有，则一直就不会变，一直保持设置的值。
- AcceptHeaderLocaleResolver：每一次请求都会调用AcceptHeaderLocaleResolver类中的resolveLocale去获取请求头中的Accept-Language的值。
  以CookieLocaleResolver为例，自定义获取语种的方式为从请求连接的传递的参数上获取（http://localhost:8081/practise/user/getUserInfor?locale=en_US）：自定义一个WebMvcConfigurer类，创建一个LocaleResolverbeen，规定使用哪种LocaleResolver，配合LocaleChangeInterceptor获取语种。
      @Configuration
      // 自定义一个WebMvcConfigurer类
      public class ConsumerWebMvcConfigurer implements WebMvcConfigurer {
          @Bean
          // 创建一个LocaleResolverbeen,也就是说，规定使用哪种LocaleResolver
          public LocaleResolver localeResolver() {
              return new CookieLocaleResolver();
          }
      
          // 一般CookieLocaleResolver配合LocaleChangeInterceptor使用，通过拦截器获取请求中的语言，默认是参数是locale名，可以自定义参数名。
          public void addInterceptors(InterceptorRegistry registry) {
              LocaleChangeInterceptor localeChangeInterceptor = new LocaleChangeInterceptor();
              // 自定义参数名
              // localeChangeInterceptor.setParamName("lang"); 
              registry.addInterceptor(localeChangeInterceptor);
          }
      }

2、图片、文本等静态资源的访问

		SpringBoot默认会在classpath:[/META-INF/resources/,/resources/, /static/, /public/]下查找静态资源，在yml中可以使用spring.resources指定静态资源路径。如果我们将静态资源路径存放在以上四个路径下，那么我们无需做任何配置就可以直接访问我们的静态资源文件（http://localhost:8081/practise/test.jpg）。

    @ConfigurationProperties(prefix = "spring.resources", ignoreUnknownFields = false)
    public class ResourceProperties {
    
       private static final String[] CLASSPATH_RESOURCE_LOCATIONS = { "classpath:/META-INF/resources/",
             "classpath:/resources/", "classpath:/static/", "classpath:/public/" };
    
       /**
        * Locations of static resources. Defaults to classpath:[/META-INF/resources/,
        * /resources/, /static/, /public/].
        */
       private String[] staticLocations = CLASSPATH_RESOURCE_LOCATIONS;
       ****省略其他代码*****
     }

3、SpringBoot的全局配置文件加载

		SpringBoot提供了三种格式的全局配置文件，application.yml、application.yaml、application.propreties。如果项目没有三者中的一个，那么SpringBoot就会使用默认的配置，比如：sevlet容器就会使用Tomcat容器，端口自然会使用Tomcat的默认端口8080。




