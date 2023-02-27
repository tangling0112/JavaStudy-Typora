# 2.`DispatcherServlet`获取`HandlerAdapter`处理器适配器的工作原理

## 2.1 `SpringBoot`中`SpringMVC`组件的四大`HandlerAdapter`实现类

- `RequestMappingHandlerAdapter`
    - 用于处理我们的控制器类方法类型的处理器的调用的适配器
- `HandlerFunctionAdapter`
- `HttpRequestHandlerAdapter`
- `SimpleControllerHandlerAdapter`

## 2.2 各个`HandlerAdapter`被加载到`DispatcherServlet`的原理

> `DispatcherServlet`类

```java
private void initHandlerAdapters(ApplicationContext context) {
   this.handlerAdapters = null;
	//detectAllHandlerAdapters是DispatcherServlet的属性,其默认值为true
   if (this.detectAllHandlerAdapters) {
      // 我们可以理解为BeanFactory.getBean(HandlerAdapter.class),其获取了我们的BeanFactory中的所有实现了HandlerAdapter的实现类Bean对象
      Map<String, HandlerAdapter> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.handlerAdapters = new ArrayList<>(matchingBeans.values());
         // We keep HandlerAdapters in sorted order.
         AnnotationAwareOrderComparator.sort(this.handlerAdapters);
      }
   }
   else {
      try {
         HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
         this.handlerAdapters = Collections.singletonList(ha);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default HandlerAdapter later.
      }
   }

   // Ensure we have at least some HandlerAdapters, by registering
   // default HandlerAdapters if no other adapters are found.
   if (this.handlerAdapters == null) {
      this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

## 2.3 `RequestMappingHandlerAdapter`被我们的处理器成功匹配的原理

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230211005933724.png" alt="image-20230211005933724" style="zoom: 80%;" />

> `DispatcherServlet`类

```java
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler +
            "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

> `AbstractHandlerMethodAdapter`抽象类

```java
public final boolean supports(Object handler) {
    //1.我们前面就提到了,如果我们从RequestMappingHandlerMapping中顺利获取到了对应的控制器类方法,那么最终就会获得一个封装了这个控制器类方法的反射Method实例的HandlerMethod类的类实例,所以有一个instanceof HandlerMethod
    //2.我们的这个supportsInternal实际上是this.supportsInternal,很显然我们当前是在判断RequestMappingHandlerAdapter,因此调用的就是RequestMappingHandlerAdapter类实例的supportsInternal方法,这个方法返回值恒为true
    return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}
```

> `RequestMappingHandlerAdapter`类

```java
protected boolean supportsInternal(HandlerMethod handlerMethod) {
    return true;
}
```
