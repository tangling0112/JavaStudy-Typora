# `SpringBoot`的基于`ErrorMvcAutoConfiguration`默认异常处理器的自动配置

## 总体概括

### `ErrorMvcAutoConfiguration`类包含的组件汇总

- `DefualtErrorAttribute`异常处理器

- `BasicErrorController`基础错误处理控制器
- `ErrorPageCustomizer`错误页面定义器
- `DefaultErrorViewResolverConfiguration`默认错误视图解析器配置类
- `WhitelabelErrorViewConfiguration`错误视图配置注册器
- `StaticView`错误视图

### `ErrorMvcAutoConfiguration`类组件关系分析

- `DefualtErrorAttribute`异常处理器实现了我们的`HandlerExceptionResolver`接口,是一个异常处理器

- `WhitelabelErrorViewConfiguration`错误视图配置注册器实例化了一个**`StaticView`错误视图的实例**,并将这个错误视图实例借助`@Bean`注解注册为了`Bean`对象
- `DefaultErrorViewResolverConfiguration`默认错误视图解析器配置类实例化了一个`DefaultErrorViewResolver`**错误视图解析器**,并且将其注册为了一个`Bean`对象
    - **注意**:这个错误视图解析器并没有实现我们的`ViewResolver`接口,因此其并不会被注册成为我们的`DispatcherServlet`的视图解析器,其作用仅仅是支持我们的`BasicErrorController`的运行

- `ErrorPageCustomizer`错误页面定义器告知了我们的`SpringBoot`当遇到错误时去找`/error`映射的控制器类方法来处理
- `BasicErrorController`**基础错误处理控制器**就定义了`/error`映射来处理错误,并且**其`viewResolver`属性中存储的视图解析器就是我们前面的`DefaultErrorViewResolver`错误视图解析器**.

## `ErrorMvcAutoConfiguration`类

```java
@AutoConfiguration(before = WebMvcAutoConfiguration.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class })
@EnableConfigurationProperties({ ServerProperties.class, WebMvcProperties.class })
public class ErrorMvcAutoConfiguration {
```

> ​		我们的`ErrorMvcAutoConfiguration`会被我们的`Spring`自动加载为`Bean`对象,由于其没有无参构造器,因此会调用其这个有参构造器去实例化它的`Bean`对象.这里的`ServerProperties`参数会由我们的`Spring`的机制找到实现类`ServerProperties`接口的`Bean`对象来传入

```java
public ErrorMvcAutoConfiguration(ServerProperties serverProperties) {
   this.serverProperties = serverProperties;
}
```

****

### `DefualtErrorAttribute`异常处理器的注册

```java
@Bean
@ConditionalOnMissingBean(value = ErrorAttributes.class, search = SearchStrategy.CURRENT)
public DefaultErrorAttributes errorAttributes() {
   return new DefaultErrorAttributes();
}
```

> d

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
public class DefaultErrorAttributes implements ErrorAttributes, HandlerExceptionResolver, Ordered {
```

****

### `BasicErrorController`基础错误处理控制器的注册

```java
@Bean
@ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
public BasicErrorController basicErrorController(ErrorAttributes errorAttributes,
      ObjectProvider<ErrorViewResolver> errorViewResolvers) {
   return new BasicErrorController(errorAttributes, this.serverProperties.getError(),
         errorViewResolvers.orderedStream().collect(Collectors.toList()));
}
```

****

### `ErrorPageCustomizer`错误页面自定义器的注册

```java
@Bean
public ErrorPageCustomizer errorPageCustomizer(DispatcherServletPath dispatcherServletPath) {
   return new ErrorPageCustomizer(this.serverProperties, dispatcherServletPath);
}
```

****

### `DefaultErrorViewResolverConfiguration`内部类

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties({ WebProperties.class, WebMvcProperties.class })
static class DefaultErrorViewResolverConfiguration {
```

> ​		和我们的`ErrorMvcAutoConfiguration`的加载`Bean`对象一样,我们的`DefaultErrorViewResolverConfiguration`也会借助有参构造器实例化为`Bean`对象

```java
DefaultErrorViewResolverConfiguration(ApplicationContext applicationContext, WebProperties webProperties) {
   this.applicationContext = applicationContext;
   this.resources = webProperties.getResources();
}
```

#### `DefaultErrorViewResolver`默认错误视图解析器的注册

```java
@Bean
@ConditionalOnBean(DispatcherServlet.class)
@ConditionalOnMissingBean(ErrorViewResolver.class)
DefaultErrorViewResolver conventionErrorViewResolver() {
   return new DefaultErrorViewResolver(this.applicationContext, this.resources);
}
```

****

### `WhitelabelErrorViewConfiguration`内部类

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(prefix = "server.error.whitelabel", name = "enabled", matchIfMissing = true)
@Conditional(ErrorTemplateMissingCondition.class)
protected static class WhitelabelErrorViewConfiguration {
```

> ​		这个内部类没有设置任何构造器,因此我们的`Spring`会直接使用`Java`给该类自动设置的无参构造器来实例化其`Bean`对象

#### `defaultErrorView`默认错误视图的注册

```java
private final StaticView defaultErrorView = new StaticView();

@Bean(name = "error")
@ConditionalOnMissingBean(name = "error")
public View defaultErrorView() {
   return this.defaultErrorView;
}
```

#### `BeanNameViewResolver`的注册

```java
@Bean
@ConditionalOnMissingBean
public BeanNameViewResolver beanNameViewResolver() {
   BeanNameViewResolver resolver = new BeanNameViewResolver();
   resolver.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
   return resolver;
}
```

****

### `StaticView`内部类

```java
private static class StaticView implements View 
```

> `StaticView`视图的`render`渲染方法
>
> - 这个方法我们前面讨论视图解析机制的时候知道,我们的`DispatcherServlet`首先会去调用自己的`render`方法去调用`ContentNegotiationViewResolver`视图解析器去尝试解析我们的视图,而我们的这个视图解析器中存储了其他的视图解析器,因此它会依次调用各个视图解析器的`resolverViewName`方法,我们的`defualtErrorViewResolver`视图解析器的这个方法就会生成一个`defualtErrorView`默认错误视图,而这个视图实际上就是一个`StaticView`视图实例
> - 然后就会调用各个视图实例的`render`方法找到最合适的渲染结果,因此就会调用到我们的这个`StaticView`的`render`方法来渲染我们的视图

```java
@Override
public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
      throws Exception {
   if (response.isCommitted()) {
      String message = getMessage(model);
      logger.error(message);
      return;
   }
   response.setContentType(TEXT_HTML_UTF8.toString());
   StringBuilder builder = new StringBuilder();
   Object timestamp = model.get("timestamp");
   Object message = model.get("message");
   Object trace = model.get("trace");
   if (response.getContentType() == null) {
      response.setContentType(getContentType());
   }
   builder.append("<html><body><h1>Whitelabel Error Page</h1>").append(
         "<p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>")
         .append("<div id='created'>").append(timestamp).append("</div>")
         .append("<div>There was an unexpected error (type=").append(htmlEscape(model.get("error")))
         .append(", status=").append(htmlEscape(model.get("status"))).append(").</div>");
   if (message != null) {
      builder.append("<div>").append(htmlEscape(message)).append("</div>");
   }
   if (trace != null) {
      builder.append("<div style='white-space:pre-wrap;'>").append(htmlEscape(trace)).append("</div>");
   }
   builder.append("</body></html>");
   response.getWriter().append(builder.toString());
}
```

****

### `ErrorTemplateMissingCondition`内部类

```java
private static class ErrorTemplateMissingCondition extends SpringBootCondition {

   @Override
   public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
      ConditionMessage.Builder message = ConditionMessage.forCondition("ErrorTemplate Missing");
      TemplateAvailabilityProviders providers = new TemplateAvailabilityProviders(context.getClassLoader());
      TemplateAvailabilityProvider provider = providers.getProvider("error", context.getEnvironment(),
            context.getClassLoader(), context.getResourceLoader());
      if (provider != null) {
         return ConditionOutcome.noMatch(message.foundExactly("template from " + provider));
      }
      return ConditionOutcome.match(message.didNotFind("error template view").atAll());
   }

}
```

****

### `ErrorPageCustomizer`内部类

```java
static class ErrorPageCustomizer implements ErrorPageRegistrar, Ordered {
```

> ​		不同于前面的一些类会被直接注册为`Bean`对象,我们的这个内部类的实例会在上面的`errorPageCustomizer`方法上通过`@Bean`注解被注册为`Bean`对象

```java
protected ErrorPageCustomizer(ServerProperties properties, DispatcherServletPath dispatcherServletPath) {
   this.properties = properties;
   this.dispatcherServletPath = dispatcherServletPath;
}
```

> ​		这里就是为什么我们的`SpringBoot`会默认在遇到错误时去寻找我们的`/error`映射来解决我们的错误的原因.

```java
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {
```

```java
public void registerErrorPages(ErrorPageRegistry errorPageRegistry) {
   ErrorPage errorPage = new ErrorPage(
         this.dispatcherServletPath.getRelativePath(this.properties.getError().getPath()));
   errorPageRegistry.addErrorPages(errorPage);
}
```

```java
public ErrorProperties getError() {
   return this.error;
}
```

```java
@NestedConfigurationProperty
private final ErrorProperties error = new ErrorProperties();
```

```java
public String getPath() {
   return this.path;
}
```

```java
public String getPath() {
   return this.path;
}
```

```java
@Value("${error.path:/error}")
private String path = "/error";
```

****

## `BasicErrorController`类

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
```

> 这个类就是一个给出了`/error`映射的控制器类 ,我们上面的`ErrorPageCustomizer`内部类设置了当我们的`SpringBoot`在产生异常时会去寻找`/error`映射的控制器类方法来处理我们的异常
>
> ​		而我们的`BasicErrorController`类,刚好就定义了一个处理这个`/error`映射的控制器类方法

> ​		注意,这个`BasicErrorController`类一两个被`@RequestMapping`修饰的方法
>
> - 第一个方法通过指定`produces`属性使得只有浏览器发出的请求才会被这个控制器类方法处理
> - 第二个方法什么也没指定,也就意味着只要前面的方法无法处理的请求就会由这个请求来处理也就是所谓的处理机器发出的请求

​		如果我们的`DefualtErrorViewResolver`错误解析处理器成功地找到了合适的页面去响应我们的用户,那么我们的`BasicErrorController`就会返回一个携带着指定的页面的`ModelAndView`,如果不能找到就会返回一个`ModelAndView("error", model)`的实例

- 第一个实例会被我们的视图解析器再度解析然后产生合适的视图发送给我们的用户
- 第二个实例会被我们的`BeanNameViewResolver`解析器解析,会获取我们的`BeanFactory`中的一个`ID=error`的`Bean`对象作为其视图,而显然我们的`SpringBoot`通过`WhitelabelErrorViewConfiguration`恰好就注册了一个`ID=error`的`Bean`对象,这个对象是`StaticView`类的实例

```java
@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
   HttpStatus status = getStatus(request);
   Map<String, Object> model = Collections
         .unmodifiableMap(getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML)));
   response.setStatus(status.value());
   ModelAndView modelAndView = resolveErrorView(request, response, status, model);
   return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
}

@RequestMapping
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
   HttpStatus status = getStatus(request);
   if (status == HttpStatus.NO_CONTENT) {
      return new ResponseEntity<>(status);
   }
   Map<String, Object> body = getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.ALL));
   return new ResponseEntity<>(body, status);
}
```

```java
protected ModelAndView resolveErrorView(HttpServletRequest request, HttpServletResponse response, HttpStatus status,
      Map<String, Object> model) {
   for (ErrorViewResolver resolver : this.errorViewResolvers) {
      ModelAndView modelAndView = resolver.resolveErrorView(request, status, model);
      if (modelAndView != null) {
         return modelAndView;
      }
   }
   return null;
}
```

> ​		`BasicErrorController`是被我们的`ErrorMvcAutoConfiguration`注册为`Bean`对象的,其`this.errorViewResolvers`属性是直接在我们的`BeanFactory`中加载的所有实现了`ErrorViewResolver`接口的`Bean`对象,而我们上面的`DefaultErrorViewResolver`所注册的错误视图解析器`Bean`对象正好是实现了`ErrorViewResolver`接口的.因此默认情况下就是调用的`DefaultErrorViewResolver`类的`resolveErrorView`方法

## `DefaultErrorViewResolver`错误视图解析器作用详解

> ​		这个视图解析器会根据我们的`Http`响应状态码去我们的`SpringBoot`的静态资源存储地址下的`error`目录下去查找与我们的`Http`响应状态码相对应的`html`文件
>
> ​		注意:除常规的静态目录外`templates`目录也会查找

- `/<templates>/error/404.<ext>`
- `/<static>/error/404.html`
- `/<templates>/error/4xx.<ext>`
- `/<static>/error/4xx.html`
- `/META-INF/resources/error/404.html`

```java
//这个方法被我们的BasicErrorViewController控制器的控制器类方法调用
public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
   //调用自身的resolve方法去解析我们的视图
   ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
    //上面的方式找不到合适的视图,因此采用另一种方式
   if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
      modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
   }
    //当找得到文件时就会返回ModelAndView实例,找不到就会返回null
   return modelAndView;
}
```

```java
//viewName形参接收的实际上就是我们的Http响应状态码的字符串,如"500","404"等等
//第二次调用该方法时传参就是"5xx","4xx"
private ModelAndView resolve(String viewName, Map<String, Object> model) {
   //拼接出我们的查找路径为"error/500"
   //第二次拼接我们的查找路径为"error/5xx"
   String errorViewName = "error/" + viewName;
   TemplateAvailabilityProvider provider = this.templateAvailabilityProviders.getProvider(errorViewName,
         this.applicationContext);
   if (provider != null) {
      return new ModelAndView(errorViewName, model);
   }
   //查找静态资源以获取我们的ModelAndView实例
   return resolveResource(errorViewName, model);
}
```

```java
private ModelAndView resolveResource(String viewName, Map<String, Object> model) {
    //获取到我们的SpringBoot存储静态资源的路径,遍历它们
   for (String location : this.resources.getStaticLocations()) {
      try {
         Resource resource = this.applicationContext.getResource(location);
          //查看静态资源路径/error/500.html是否存在
          //第二次查看静态资源路径/error/5xx.html
         resource = resource.createRelative(viewName + ".html");
         if (resource.exists()) {
            return new ModelAndView(new HtmlResourceView(resource), model);
         }
      }
      catch (Exception ex) {
      }
   }
   return null;
}
```
