## 1.`FormContentFilter`与`HiddenHttpMethodFilter`的配置

## 2.基于`WebMvcAutoConfigurationAdapter`内部类的配置原理

### `WebMvcAutoConfigurationAdapter`的声明体

```java
@Configuration(proxyBeanMethods = false)
@Import(EnableWebMvcConfiguration.class)
@EnableConfigurationProperties({ WebMvcProperties.class, WebProperties.class })
@Order(0)
public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer, ServletContextAware {
```

### `WebMvcAutoConfigurationAdapter`中未使用`@Bean`这种直接注册`Bean`对象的方式的方法

```java
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
   this.messageConvertersProvider
         .ifAvailable((customConverters) -> converters.addAll(customConverters.getConverters()));
}
```

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

```java
public void addFormatters(FormatterRegistry registry) {
   ApplicationConversionService.addBeans(registry, this.beanFactory);
}
```

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

## 3.基于`EnableWebMvcConfiguration`内部类的配置原理

### `EnableWebMvcConfiguration`的声明体

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(WebProperties.class)
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
```

### `WebMvcAutoConfigurationAdapter`中未使用`@Bean`这种直接注册`Bean`对象的方式的方法

```java
protected RequestMappingHandlerMapping createRequestMappingHandlerMapping() {
   if (this.mvcRegistrations != null) {
      RequestMappingHandlerMapping mapping = this.mvcRegistrations.getRequestMappingHandlerMapping();
      if (mapping != null) {
         return mapping;
      }
   }
   return super.createRequestMappingHandlerMapping();
}
```

```java
protected RequestMappingHandlerAdapter createRequestMappingHandlerAdapter() {
   if (this.mvcRegistrations != null) {
      RequestMappingHandlerAdapter adapter = this.mvcRegistrations.getRequestMappingHandlerAdapter();
      if (adapter != null) {
         return adapter;
      }
   }
   return super.createRequestMappingHandlerAdapter();
}
```

```java
protected ConfigurableWebBindingInitializer getConfigurableWebBindingInitializer(
      FormattingConversionService mvcConversionService, Validator mvcValidator) {
   try {
      return this.beanFactory.getBean(ConfigurableWebBindingInitializer.class);
   }
   catch (NoSuchBeanDefinitionException ex) {
      return super.getConfigurableWebBindingInitializer(mvcConversionService, mvcValidator);
   }
}
```

```java
protected ExceptionHandlerExceptionResolver createExceptionHandlerExceptionResolver() {
   if (this.mvcRegistrations != null) {
      ExceptionHandlerExceptionResolver resolver = this.mvcRegistrations
            .getExceptionHandlerExceptionResolver();
      if (resolver != null) {
         return resolver;
      }
   }
   return super.createExceptionHandlerExceptionResolver();
}
```

```java
protected void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> exceptionResolvers) {
   super.extendHandlerExceptionResolvers(exceptionResolvers);
   if (this.mvcProperties.isLogResolvedException()) {
      for (HandlerExceptionResolver resolver : exceptionResolvers) {
         if (resolver instanceof AbstractHandlerExceptionResolver) {
            ((AbstractHandlerExceptionResolver) resolver).setWarnLogCategory(resolver.getClass().getName());
         }
      }
   }
}
```
