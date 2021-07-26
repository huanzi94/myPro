Spring Boot自动配置原理

---

1、Spring Boot自动配置

		Spring Boot最大的特点就是省略了繁杂的配置工作。以前我们使用Spring 、Spring MVC的时候，需要配置web.xml、application.xml等一些配置文件，来加载一些我们需要的组件，比如Spring整合Mybatis，我们需要手动配置大量的配置信息。而使用Spring Boot帮我们省去大量繁杂的配置工作，我们的工作就是在pom.xml中添加Mybatis的场景依赖（mybatis-spring-boot-starter），在application.yml文件中设置数据数据库的地址、用户名以及其他的设置即可。		

 2、Spring Boot自动配置原理思维导图

<img src="C:\Users\lwx5317690\AppData\Roaming\Typora\typora-user-images\image-20210726164708452.png" alt="image-20210726164708452" style="zoom:100%" />

3、主程序入口、启动类

		Spring Boot项目的都有一个启动类，该类一般以XXXAppApplication命名，并且需要有@SpringBootApplication注解，表是这是一个启动类。雷必须包含main()方法，所以，Spring Boot项目可以直接从这个类启动项目，右键，run main或者debug main即可启动。

    @SpringBootApplication
    public class OjAppApplication {
        public static void main(String []args){
            SpringApplication.run(OjAppApplication.class, args);
        }
    }

3.1 、@SpringBootApplication  注解

		这是一个复合注解，自动配置就是从这里开始处理，其中最主要的是@EnableAutoConfiguration注解。

    @Target({ElementType.TYPE}) // 
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @SpringBootConfiguration
    @EnableAutoConfiguration
    @ComponentScan(
        excludeFilters = {@Filter(
        type = FilterType.CUSTOM,
        classes = {TypeExcludeFilter.class}
    ), @Filter(
        type = FilterType.CUSTOM,
        classes = {AutoConfigurationExcludeFilter.class}
    )}
    )

@SpringBootConfiguration：这个注解表明这个类是一个SpringBoot配置类，他底层下边使用@Configuration注解。

@EnableAutoConfiguration：这个注解表明开启自动配置，也是自动配置的关键。

3.2 、@EnableAutoConfiguration注解

	  这是一个复合注解。

    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage
    @Import({AutoConfigurationImportSelector.class})
    public @interface EnableAutoConfiguration {
        String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    
        Class<?>[] exclude() default {};
    
        String[] excludeName() default {};
    }

@AutoConfigurationPackage：这个注解帮我们获取到当前类所在的包路劲，这个路劲将会作为Spring Boot管理Been的package进行使用。也就是说，Spring Boot会扫描这个包下的所有的类，并且将这些类注册为Been，交给Spring IOC容器来管理。


