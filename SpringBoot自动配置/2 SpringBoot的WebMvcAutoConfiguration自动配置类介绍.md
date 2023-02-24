# 1 `WebMvcAutoConfiguration`类

```java
@AutoConfiguration(after = { DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
    ValidationAutoConfiguration.class })
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebMvcAutoConfiguration {
```

## 1.1 `HiddenHttpMethodFilter`过滤器的注册

> ​		这个过滤器在`SpringMVC`中用于根据我们用户发起的`POST`请求携带的`_method`属性的值来将用户的请求类型改为`PUT/DELETE/PATCH`等等,用于解决浏览器只支持发送`GET/POST`请求的问题.使得我们能够很好地使用`RESTful`设计风格

```java
@Bean
@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
//这个注解使得我们
@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled")
public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
    return new OrderedHiddenHttpMethodFilter();
}
```

### `HiddenHttpMethodFilter`实现`RESTful`风格的原理

```java
public class OrderedHiddenHttpMethodFilter extends HiddenHttpMethodFilter implements OrderedFilter
public class HiddenHttpMethodFilter extends OncePerRequestFilter
```

> 在语雀的`SSM/SpringMVC`笔记的[`SpringMVC`源码解析](https://www.yuque.com/tangling0112/dsqc4k/id15p5#g6ihw)小节中有详细解释

## 1.2 `FormContentFilter`过滤器的注册

```java
@Bean
@ConditionalOnMissingBean(FormContentFilter.class)
@ConditionalOnProperty(prefix = "spring.mvc.formcontent.filter", name = "enabled", matchIfMissing = true)
public OrderedFormContentFilter formContentFilter() {
    return new OrderedFormContentFilter();
}
```

## 1.3 `WebMvcAutoConfigurationAdapter`内部类

```java
@SuppressWarnings("deprecation")
@Configuration(proxyBeanMethods = false)
@Import(EnableWebMvcConfiguration.class)
@EnableConfigurationProperties({ WebMvcProperties.class, WebProperties.class })
@Order(0)
public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer, ServletContextAware {
```

### 1.3. 1 `WebMvcAutoConfigurationAdapter`内部类的有参构造器方法

> ​		由于这个内部类是会被注册为`Bean`对象的,因此根据`Spring`的特性,在`Spring`实例化它时,他的有参构造器中的所有参数都会去我们的`Spring BeanFactory`中去找.

```java
public WebMvcAutoConfigurationAdapter(WebProperties webProperties, WebMvcProperties mvcProperties,
        ListableBeanFactory beanFactory, ObjectProvider<HttpMessageConverters> messageConvertersProvider,
        ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider,
        ObjectProvider<DispatcherServletPath> dispatcherServletPath,
        ObjectProvider<ServletRegistrationBean<?>> servletRegistrations) {
    //在这一步中获取了一个Resources类实例,而这个类实例中存储的读取文件的路径恰好就是我们SpringBoot读取的那四大静态文件存储路径CLASSPATH_RESOURCE_LOCATIONS = { "classpath:/META-INF/resources/","classpath:/resources/", "classpath:/static/", "classpath:/public/" };
    this.resourceProperties = webProperties.getResources();
    //这一步实际上就是读取了我们的SpringBoot的.properties以及.yaml配置文件中指定的一些属性,然后封装成一个WebMvcProperties实例
    this.mvcProperties = mvcProperties;
    this.beanFactory = beanFactory;
    //
    this.messageConvertersProvider = messageConvertersProvider;
    this.resourceHandlerRegistrationCustomizer = resourceHandlerRegistrationCustomizerProvider.getIfAvailable();
    //
    this.dispatcherServletPath = dispatcherServletPath;
    this.servletRegistrations = servletRegistrations;
    this.mvcProperties.checkConfiguration();
}
```

### 1.3.2 `HttpMessageConverter`的配置



> ​		若我们需要为我们`SpringBoot`的`SpringMVC`添加`HttpMessageConverter`,那么我只只需要实例化它们并存储在`List`集合类型中然后通过我们主程序的`run`方法返回的`ApplicationContext`的`getBean`获取到我们的`WebMvcConfigurationAdpter`类的`Bean`对象然后调用这个方法,将我们前面的那个`List`集合传入即可

```java
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    this.messageConvertersProvider.ifAvailable((customConverters) -> 		converters.addAll(customConverters.getConverters()));
}
```

### 1.3.3 `PathMatch`的配置

> 

```java
public void configurePathMatch(PathMatchConfigurer configurer) {
    if (this.mvcProperties.getPathmatch()
            .getMatchingStrategy() == WebMvcProperties.MatchingStrategy.PATH_PATTERN_PARSER) {
        configurer.setPatternParser(pathPatternParser);
    }
    configurer.setUseSuffixPatternMatch(this.mvcProperties.getPathmatch().isUseSuffixPattern());
    configurer.setUseRegisteredSuffixPatternMatch(
            this.mvcProperties.getPathmatch().isUseRegisteredSuffixPattern());
    this.dispatcherServletPath.ifAvailable((dispatcherPath) -> {
        String servletUrlMapping = dispatcherPath.getServletUrlMapping();
        if (servletUrlMapping.equals("/") && singleDispatcherServlet()) {
            UrlPathHelper urlPathHelper = new UrlPathHelper();
            urlPathHelper.setAlwaysUseFullPath(true);
            configurer.setUrlPathHelper(urlPathHelper);
        }
    });
}
```

### 1.3.4 `ContentNegotiation`的配置

> 

```java
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    WebMvcProperties.Contentnegotiation contentnegotiation = this.mvcProperties.getContentnegotiation();
    configurer.favorPathExtension(contentnegotiation.isFavorPathExtension());
    configurer.favorParameter(contentnegotiation.isFavorParameter());
    if (contentnegotiation.getParameterName() != null) {
        configurer.parameterName(contentnegotiation.getParameterName());
    }
    Map<String, MediaType> mediaTypes = this.mvcProperties.getContentnegotiation().getMediaTypes();
    mediaTypes.forEach(configurer::mediaType);
}
```

### 1.3.5 `InternalResourceViewResolver`类的注册

> - 在`SpringMVC`的视图机制那里我们提到了其三大视图`普通视图`,`转发视图`,`重定向视图`.每一种视图都有着它们对应的视图解析器,而我们的这个`InternalResourceViewResolver`恰好就是解析我们的`转发视图`的视图解析器

```java
@Bean
@ConditionalOnMissingBean
public InternalResourceViewResolver defaultViewResolver() {
    InternalResourceViewResolver resolver = new InternalResourceViewResolver();
    resolver.setPrefix(this.mvcProperties.getView().getPrefix());
    resolver.setSuffix(this.mvcProperties.getView().getSuffix());
    return resolver;
}
```

### 1.3.6 `BeanNameViewResolver`类的注册

```java
@Bean
@ConditionalOnBean(View.class)
@ConditionalOnMissingBean
public BeanNameViewResolver beanNameViewResolver() {
    BeanNameViewResolver resolver = new BeanNameViewResolver();
    resolver.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
    return resolver;
}
```

### 1.3.7 `ContentNegotiatingViewResolver`类的注册

```java
@Bean
@ConditionalOnBean(ViewResolver.class)
@ConditionalOnMissingBean(name = "viewResolver", value = ContentNegotiatingViewResolver.class)
public ContentNegotiatingViewResolver viewResolver(BeanFactory beanFactory) {
    ContentNegotiatingViewResolver resolver = new ContentNegotiatingViewResolver();
    resolver.setContentNegotiationManager(beanFactory.getBean(ContentNegotiationManager.class));
    // ContentNegotiatingViewResolver uses all the other view resolvers to locate
    // a view so it should have a high precedence
    resolver.setOrder(Ordered.HIGHEST_PRECEDENCE);
    return resolver;
}
```

### 1.3.8 `MessageCodesResolver`类的注册

```java
public MessageCodesResolver getMessageCodesResolver() {
    if (this.mvcProperties.getMessageCodesResolverFormat() != null) {
        DefaultMessageCodesResolver resolver = new DefaultMessageCodesResolver();
        resolver.setMessageCodeFormatter(this.mvcProperties.getMessageCodesResolverFormat());
        return resolver;
    }
    return null;
}
```

### 1.3.9 `addFormatters`

```java
public void addFormatters(FormatterRegistry registry) {
   ApplicationConversionService.addBeans(registry, this.beanFactory);
}
```

### 1.3.10 `addResourceHandlers`静态资源访问的配置

```java
public void addResourceHandlers(ResourceHandlerRegistry registry) {
   if (!this.resourceProperties.isAddMappings()) {
      logger.debug("Default resource handling disabled");
      return;
   }
   addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
   addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
      registration.addResourceLocations(this.resourceProperties.getStaticLocations());
      if (this.servletContext != null) {
         ServletContextResource resource = new ServletContextResource(this.servletContext, SERVLET_LOCATION);
         registration.addResourceLocations(resource);
      }
   });
}
```

```java
private void addResourceHandler(ResourceHandlerRegistry registry, String pattern, String... locations) {
    addResourceHandler(registry, pattern, (registration) -> registration.addResourceLocations(locations));
}
```

```java
private void addResourceHandler(ResourceHandlerRegistry registry, String pattern,
        Consumer<ResourceHandlerRegistration> customizer) {
    if (registry.hasMappingForPattern(pattern)) {
        return;
    }
    ResourceHandlerRegistration registration = registry.addResourceHandler(pattern);
    customizer.accept(registration);
    registration.setCachePeriod(getSeconds(this.resourceProperties.getCache().getPeriod()));
    registration.setCacheControl(this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl());
    registration.setUseLastModified(this.resourceProperties.getCache().isUseLastModified());
    customizeResourceHandlerRegistration(registration);
}
```

```java
private void customizeResourceHandlerRegistration(ResourceHandlerRegistration registration) {
    if (this.resourceHandlerRegistrationCustomizer != null) {
        this.resourceHandlerRegistrationCustomizer.customize(registration);
    }
}
```

```java
@Override
public void customize(ResourceHandlerRegistration registration) {
    Resources.Chain properties = this.resourceProperties.getChain();
    configureResourceChain(properties, registration.resourceChain(properties.isCache()));
}
```

```java
private void configureResourceChain(Resources.Chain properties, ResourceChainRegistration chain) {
    Strategy strategy = properties.getStrategy();
    if (properties.isCompressed()) {
        chain.addResolver(new EncodedResourceResolver());
    }
    if (strategy.getFixed().isEnabled() || strategy.getContent().isEnabled()) {
        chain.addResolver(getVersionResourceResolver(strategy));
    }
}
```



### 1.3.11 `RequestContextFilter`过滤器的注册

```java
@Bean
@ConditionalOnMissingBean({ RequestContextListener.class, RequestContextFilter.class })
@ConditionalOnMissingFilterBean(RequestContextFilter.class)
public static RequestContextFilter requestContextFilter() {
    return new OrderedRequestContextFilter();
}
```

## 1.4 `EnableWebMvcConfiguration`内部类

> ​		我们可以注意到我们的`EnableWebMvcConfiguration`内部类继承了,`DelegatingWebMvcConfiguration`类.我们应该知道后者是继承了`WebMvcConfigurationSupport`类的.因此我们的这个内部类应该算是`WebMvcConfigurationSupport`类的子类,

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(WebProperties.class)
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
```

### 1.4.1 `EnableWebMvcConfiguration`内部类的有参构造器

> ​		由于这个内部类是会被注册为`Bean`对象的,因此根据`Spring`的特性,在`Spring`实例化它时,他的有参构造器中的所有参数都会去我们的`Spring BeanFactory`中去找.

```java
public EnableWebMvcConfiguration(WebMvcProperties mvcProperties, WebProperties webProperties,
      ObjectProvider<WebMvcRegistrations> mvcRegistrationsProvider,
      ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider,
      ListableBeanFactory beanFactory) {
   this.resourceProperties = webProperties.getResources();
   this.mvcProperties = mvcProperties;
   this.webProperties = webProperties;
   this.mvcRegistrations = mvcRegistrationsProvider.getIfUnique();
   this.beanFactory = beanFactory;
}
```

### 1.4.2 `RequestMappingHandlerAdapter`类的注册

> ​		在`SpringMVC`的`DispatcherServlet`的`doDispatch`方法中,有一段是根据我们的获取到的控制器类方法去获取用于承载其运行的`HandlerAdapter`适配器.这个寻找的过程中就去找了我们应用程序所属的`BeanFactory`中实现了`HandlerAdapter`接口的`Bean`对象或是`ID`为`handlerAdapter`的`Bean`对象.
>
> ​		我们的`RequestMappingHandlerAdapter`类正是一个`HandlerAdapter`接口的实现类,因此就会在上面所述的情况下被获取.很显然从名字上看我们就知道,这个适配器是用来对`@RequestMapping`注解提供支持的,其会用于处理`@RequestMapping`注解映射的控制器类方法的调用
>
> ```java
> HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
> ```
>
> ```java
> protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
>     if (this.handlerAdapters != null) {
>         for (HandlerAdapter adapter : this.handlerAdapters) {
>             if (adapter.supports(handler)) {
>                 return adapter;
>             }
>         }
>     }
>     throw new ServletException("No adapter for handler [" + handler +
>             "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
> }
> ```
>
> ​		这里我们就获取了我们的当前应用的`BeanFactory`中的所有实现了`HandlerAdapter`接口的`Bean`对象或是`ID`为`handlerAdapter`的`Bean`对象
>
> ```java
> private void initHandlerAdapters(ApplicationContext context) {
>     this.handlerAdapters = null;
> 
>     if (this.detectAllHandlerAdapters) {
>         // Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
>         Map<String, HandlerAdapter> matchingBeans =
>                 BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
>         if (!matchingBeans.isEmpty()) {
>             this.handlerAdapters = new ArrayList<>(matchingBeans.values());
>             // We keep HandlerAdapters in sorted order.
>             AnnotationAwareOrderComparator.sort(this.handlerAdapters);
>         }
>     }
>     else {
>         try {
>             HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
>             this.handlerAdapters = Collections.singletonList(ha);
>         }
>         catch (NoSuchBeanDefinitionException ex) {
>             // Ignore, we'll add a default HandlerAdapter later.
>         }
>     }
> 
>     // Ensure we have at least some HandlerAdapters, by registering
>     // default HandlerAdapters if no other adapters are found.
>     if (this.handlerAdapters == null) {
>         this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
>         if (logger.isTraceEnabled()) {
>             logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
>                     "': using default strategies from DispatcherServlet.properties");
>         }
>     }
> }
> ```
>
> 

```java
@Bean
@Override
public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
        @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
        @Qualifier("mvcConversionService") FormattingConversionService conversionService,
        @Qualifier("mvcValidator") Validator validator) {
    RequestMappingHandlerAdapter adapter = super.requestMappingHandlerAdapter(contentNegotiationManager,
            conversionService, validator);
    adapter.setIgnoreDefaultModelOnRedirect(
            this.mvcProperties == null || this.mvcProperties.isIgnoreDefaultModelOnRedirect());
    return adapter;
}
```

### 1.4.3 `WelcomePageHandlerMapping`类的注册

> ​		在`SpringMVC`的`DispatcherServlet`的`doDispatch`方法中,有一段是根据我们的用户发来的`Request`请求的信息去寻找适合用于处理这个请求的`HandlerMapping`实例,而这个寻找的过程就去找了我们应用程序所属的`BeanFactory`中实现了`HandlerMapping`接口的`Bean`对象或是`ID`为`handlerMapping`的`Bean`对象.
>
> ```java
> mappedHandler = getHandler(processedRequest);
> ```
>
> ```java
> @Nullable
> protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
>     if (this.handlerMappings != null) {
>         for (HandlerMapping mapping : this.handlerMappings) {
>             HandlerExecutionChain handler = mapping.getHandler(request);
>             if (handler != null) {
>                 return handler;
>             }
>         }
>     }
>     return null;
> }
> ```
>
> ​		这里我们就获取了我们当前应用的`BeanFactory`中的所有实现了`HandlerMapping`接口的`Bean`对象或是`ID`为`handlerMapping`的`Bean`对象
>
> ```java
> private void initHandlerMappings(ApplicationContext context) {
>     this.handlerMappings = null;
> 
>     if (this.detectAllHandlerMappings) {
>         // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
>         Map<String, HandlerMapping> matchingBeans =
>                 BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
>         if (!matchingBeans.isEmpty()) {
>             this.handlerMappings = new ArrayList<>(matchingBeans.values());
>             // We keep HandlerMappings in sorted order.
>             AnnotationAwareOrderComparator.sort(this.handlerMappings);
>         }
>     }
>     else {
>         try {
>             HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
>             this.handlerMappings = Collections.singletonList(hm);
>         }
>         catch (NoSuchBeanDefinitionException ex) {
>             // Ignore, we'll add a default HandlerMapping later.
>         }
>     }
> 
>     // Ensure we have at least one HandlerMapping, by registering
>     // a default HandlerMapping if no other mappings are found.
>     if (this.handlerMappings == null) {
>         this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
>         if (logger.isTraceEnabled()) {
>             logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
>                     "': using default strategies from DispatcherServlet.properties");
>         }
>     }
> 
>     for (HandlerMapping mapping : this.handlerMappings) {
>         if (mapping.usesPathPatterns()) {
>             this.parseRequestPath = true;
>             break;
>         }
>     }
> }
> ```
>
> 

```java
@Bean
public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext,
        FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {
    WelcomePageHandlerMapping welcomePageHandlerMapping = new WelcomePageHandlerMapping(
            new TemplateAvailabilityProviders(applicationContext), applicationContext, getWelcomePage(),
            this.mvcProperties.getStaticPathPattern());
    welcomePageHandlerMapping.setInterceptors(getInterceptors(mvcConversionService, mvcResourceUrlProvider));
    welcomePageHandlerMapping.setCorsConfigurations(getCorsConfigurations());
    return welcomePageHandlerMapping;
}
```

```java
WelcomePageHandlerMapping(TemplateAvailabilityProviders templateAvailabilityProviders,
      ApplicationContext applicationContext, Resource welcomePage, String staticPathPattern) {
   if (welcomePage != null && "/**".equals(staticPathPattern)) {
      logger.info("Adding welcome page: " + welcomePage);
      setRootViewName("forward:index.html");
   }
   else if (welcomeTemplateExists(templateAvailabilityProviders, applicationContext)) {
      logger.info("Adding welcome page template: index");
      setRootViewName("index");
   }
}
```

#### 基于`WelcomePageHandlerMapping`的欢迎页面的获取原理

> ​		首先获取到我们`SpringBoot`静态文件的存储路径,也就是类路径下的`META-INF/resources`,`/static`这些.然后就会去遍历每一个位置查找是否有我们的`index.html`文件,找到了就会返回它.没找到就换一种方式找
>
> ​		第二种方式就会去找我们的项目的根路径下的`index.html`文件,注意是项目根路径,不是`classpath`的根路径.
>
> ​		如果我们找到了`index.html`文件,这个文件就会以被用于构造`WelcomePageHandlerMapping`实例,并且如果用`@RequestMapping`注解形容的话,其`@RequestMapping="/**"`

```java
private Resource getWelcomePage() {
   for (String location : this.resourceProperties.getStaticLocations()) {
      Resource indexHtml = getIndexHtml(location);
      if (indexHtml != null) {
         return indexHtml;
      }
   }
   ServletContext servletContext = getServletContext();
   if (servletContext != null) {
      return getIndexHtml(new ServletContextResource(servletContext, SERVLET_LOCATION));
   }
   return null;
}

private Resource getIndexHtml(String location) {
   return getIndexHtml(this.resourceLoader.getResource(location));
}

private Resource getIndexHtml(Resource location) {
   try {
      Resource resource = location.createRelative("index.html");
      if (resource.exists() && (resource.getURL() != null)) {
         return resource;
      }
   }
   catch (Exception ex) {
   }
   return null;
}
```

### 1.4.4 `LocaleResolver`类的注册

```java
@Override
@Bean
@ConditionalOnMissingBean(name = DispatcherServlet.LOCALE_RESOLVER_BEAN_NAME)
public LocaleResolver localeResolver() {
    if (this.webProperties.getLocaleResolver() == WebProperties.LocaleResolver.FIXED) {
        return new FixedLocaleResolver(this.webProperties.getLocale());
    }
    AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
    localeResolver.setDefaultLocale(this.webProperties.getLocale());
    return localeResolver;
}
```

### 1.4.5 `ThemeResolver`类以及`FlashManager`类的注册

```java
@Override
@Bean
@ConditionalOnMissingBean(name = DispatcherServlet.THEME_RESOLVER_BEAN_NAME)
public ThemeResolver themeResolver() {
	return super.themeResolver();
}

@Override
@Bean
@ConditionalOnMissingBean(name = DispatcherServlet.FLASH_MAP_MANAGER_BEAN_NAME)
public FlashMapManager flashMapManager() {
    return super.flashMapManager();
}
```

### 1.4.6 `FormattingConversionService`类的注册

```java
@Bean
@Override
public FormattingConversionService mvcConversionService() {
   Format format = this.mvcProperties.getFormat();
   WebConversionService conversionService = new WebConversionService(new DateTimeFormatters()
         .dateFormat(format.getDate()).timeFormat(format.getTime()).dateTimeFormat(format.getDateTime()));
   addFormatters(conversionService);
   return conversionService;
}
```

### 1.4.7 `Validator`类的注册

```java
@Bean
@Override
public Validator mvcValidator() {
   if (!ClassUtils.isPresent("javax.validation.Validator", getClass().getClassLoader())) {
      return super.mvcValidator();
   }
   return ValidatorAdapter.get(getApplicationContext(), getValidator());
}
```

### 1.4.8 `ContentNegotiationManager`类的注册

```java
@Bean
@Override
@SuppressWarnings("deprecation")
public ContentNegotiationManager mvcContentNegotiationManager() {
   ContentNegotiationManager manager = super.mvcContentNegotiationManager();
   List<ContentNegotiationStrategy> strategies = manager.getStrategies();
   ListIterator<ContentNegotiationStrategy> iterator = strategies.listIterator();
   while (iterator.hasNext()) {
      ContentNegotiationStrategy strategy = iterator.next();
      if (strategy instanceof org.springframework.web.accept.PathExtensionContentNegotiationStrategy) {
         iterator.set(new OptionalPathExtensionContentNegotiationStrategy(strategy));
      }
   }
   return manager;
}
```

## 1.5 `ResourceChainCustomizerConfiguration`内部类

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnEnabledResourceChain
static class ResourceChainCustomizerConfiguration {

   @Bean
   ResourceChainResourceHandlerRegistrationCustomizer resourceHandlerRegistrationCustomizer(
         WebProperties webProperties) {
      return new ResourceChainResourceHandlerRegistrationCustomizer(webProperties.getResources());
   }

}
```

### 1.5.1 `ResourceChainResourceHandlerRegistrationCustomizer`类的介绍

# `DispatcherServletAutoConfiguration`类

# `ErrorMvcAutoConfiguration`类

# `HttpEncodingAutoConfiguration`类

# `MultipartAutoConfiguration`类