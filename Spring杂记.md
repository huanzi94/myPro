一、Aware接口是用来帮助我们获取Spring容器的对象。

二、beanPostProcess实际上是AOP的实现，是对bean的增强器。

- AutowiredAnnotationBeanPostProcessor：@Autowired、@Value注解的实现类。
- CommonAnnotationBeanPostProcessor：@PostConstuct、@PreDestory、@Resource注解实现类。
- ConfigurationClassParser：
  入口：ConfigurationClassPostProcessor.processConfigBeanDefinitions-->ConfigurationClassParser.parse|doProcessConfigurationClass
  @Component、@ComponentScans、@PropertySources、@ImportResource、@Import注解的实现类。
