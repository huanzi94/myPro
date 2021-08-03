
一、SpringBoot-Jackson引擎

1、HttpMessageConvertersAutoConfiguration自动配置类

		这个是SpringBoot的消息转化器，默认支持三种JSON框架@Import({JacksonHttpMessageConvertersConfiguration.class, GsonHttpMessageConvertersConfiguration.class, JsonbHttpMessageConvertersConfiguration.class})：

- Jackson：SpringBoot默认开启的JSON框架，简单易用，所用的依赖相对较少，性能相对较高。但是对复杂的转化会出现问题。

- Gson：Gson是目前功能最全的JSON解析解析器，基本无依赖，功能上面无可挑剔。
- Jsonb：Jsonb在功能上和性能上有好多缺陷，依赖的第三方jar包较多，对复杂的转化会出现问题。

    @Configuration
    @ConditionalOnClass({HttpMessageConverter.class})
    @AutoConfigureAfter({GsonAutoConfiguration.class, JacksonAutoConfiguration.class, JsonbAutoConfiguration.class})
    @Import({JacksonHttpMessageConvertersConfiguration.class, GsonHttpMessageConvertersConfiguration.class, JsonbHttpMessageConvertersConfiguration.class})
    public class HttpMessageConvertersAutoConfiguration {
        …………………………
    }

2、jackson常用注解使用

		jackson的这些注解可以配合org.springframework.validation.annotation做参数校验，@Validated通过建立分组来区分不同场景下参数校验规则，基本上可以满足大部分需求。

-  @JsonProperty("userName")：给属性设置别名
-  @JsonInclude(JsonInclude.Include.NON_NULL)：该属性不为空的时候，才包含这个属性
-  @JsonIgnore：忽略这个属性，返回给客户端的时候不包含这个字段。
- @JsonFormat(pattern = "yyyy-mm-dd HH:MM:ss")：时间格式化。

    public class User {
        @JsonProperty("userName")
        private String name;
        @JsonInclude(JsonInclude.Include.NON_NULL)
        @Null(groups = Query.class, message = "性别为空")
        @NotBlank(groups = Add.class, message = "性别不能为空")
        private String sex;
        @JsonIgnore
        private int age;
        @JsonFormat(pattern = "yyyy-mm-dd HH:MM:ss")
        @NotNull(groups = Add.class, message = "出生日期不能为空")
        private Date birthday = new Date();
    }

3、JSON序列化和反序列化

		jackson的@JsonComponent注解可以帮助我们定制自己的序列化规则，SpringBoot提供了已经分装好的序列化类JsonObjectSerializer<T>和反序列化JsonObjectDeserializer<T>类。

注意：使用序列化和反序列化的时候，实体类上的Jsckson就会失效。

    @JsonComponent
    public class UserJsonConsumer {
    
        public static class Serializer extends JsonObjectSerializer<User> {
    
            @Override
            protected void serializeObject(User user, JsonGenerator jgen, SerializerProvider provider) throws IOException {
                jgen.writeObjectField("userName", user.getName());
                // 这里可以序列化其他的字段。当然这里也可以做一些其他的业务逻辑处理。比如：根据用户的角色，返回部分字段，例如admin用户可以查看所有字段，而ordinary用户只可以查看部分字段。
            }
        }
    
        public static class DSerializer extends JsonObjectDeserializer<User> {
    
            @Override
            protected User deserializeObject(JsonParser jsonParser, DeserializationContext context, ObjectCodec codec,
                JsonNode tree) throws IOException {
                User user = new User();
                user.setName(tree.findValue("userName").asText());
                user.setSex(tree.findValue("sex").asText());
                return user;
            }
        }
    }

二、SpringBoot-BasicErrorController全局异常

1、ErrorMvcAutoConfiguration自动配置类

		这个是SpringBoot全局异常自动配置类。ErrorMvcAutoConfiguration配置类中注入了DefaultErrorAttributes和BasicErrorController的两个Bean。

    @Configuration
    @ConditionalOnWebApplication(type = Type.SERVLET)
    @ConditionalOnClass({ Servlet.class, DispatcherServlet.class })
    // Load before the main WebMvcAutoConfiguration so that the error View is available
    @AutoConfigureBefore(WebMvcAutoConfiguration.class)
    @EnableConfigurationProperties({ ServerProperties.class, ResourceProperties.class, WebMvcProperties.class })
    public class ErrorMvcAutoConfiguration {
       private final ServerProperties serverProperties;
        
       @Bean
       @ConditionalOnMissingBean(value = ErrorAttributes.class, search = SearchStrategy.CURRENT)
       public DefaultErrorAttributes errorAttributes() {
          return new DefaultErrorAttributes(this.serverProperties.getError().isIncludeException());
       }
        
       @Bean
       @ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
       public BasicErrorController basicErrorController(ErrorAttributes errorAttributes) {
          return new BasicErrorController(errorAttributes, this.serverProperties.getError(), this.errorViewResolvers);
       }
    }

1.1、DefaultErrorAttributes Bean

DefaultErrorAttributes 实际上是封装了ServerProperties里面的配置内容，

1.2、DefaultErrorViewResolverConfiguration Bean

1.3、BasicErrorController Bean

		BasicErrorController 实际上就是一个全局异常的控制器。

    @Controller
    // 如果自定义了异常映射那么就使用自定义的异常映射，默认/error注解，
    @RequestMapping("${server.error.path:${error.path:/error}}")
    public class BasicErrorController extends AbstractErrorController {
        // 如果是页面请求produces = MediaType.TEXT_HTML_VALUE，则返回一个默认的ModelAndView视图模板。
        @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
    	public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
            // 这里获取的就是Http请求的状态码
    		HttpStatus status = getStatus(request);
    		Map<String, Object> model = Collections
    				.unmodifiableMap(getErrorAttributes(request, isIncludeStackTrace(request, MediaType.TEXT_HTML)));
    		response.setStatus(status.value());
    		ModelAndView modelAndView = resolveErrorView(request, response, status, model);
    		return (modelAnmadView != null) ? modelAndView : new ModelAndView("error", model);
    	}
    
        // 如果是其他请求方式（Ajax请求），就返回封装的JSON数据。
    	@RequestMapping
    	public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
    		Map<String, Object> body = getErrorAttributes(request, isIncludeStackTrace(request, MediaType.ALL));
    		HttpStatus status = getStatus(request);
    		return new ResponseEntity<>(body, status);
    	}
    }

- getErrorAttributes()：实际上就是调用了DefaultErrorAttributes 的getErrorAttributes方法，返回了异常需要的各种属性。
      @Override
      public Map<String, Object> getErrorAttributes(WebRequest webRequest, boolean includeStackTrace) {
         Map<String, Object> errorAttributes = new LinkedHashMap<>();
         errorAttributes.put("timestamp", new Date()); // 时间戳
         addStatus(errorAttributes, webRequest); // 错误状态
         addErrorDetails(errorAttributes, webRequest, includeStackTrace); // 错误详情，可以控制是否允许堆栈信息返回
         addPath(errorAttributes, webRequest); // 错误请求的路劲
         return errorAttributes;
      }
- resolveErrorView()：查找错误视图处理器，默认委派给ErrorViewResolver视图处理器处理，也就是DefaultErrorViewResolver处理。实际上也就是调用了DefaultErrorViewResolver的resolveErrorView()方法。该方法根据状态码去/error下寻找xxx.html，如果我们没有配置，那么直接返回一个null。最终errorHtml()就会帮我们new ModelAndView("error", model)返回到客户端，而error()会帮我们直接将错误信息封装到ResponseEntity响应实体中返回。
      @Override
      public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
         ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
         if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
            modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
         }
         return modelAndView;
      }
  2、定制全局配置错误页面
  		了解了上述的全局错误配置原理之后，那么我们可以自己定制自己的错误页面或者返回信息。
  
  
