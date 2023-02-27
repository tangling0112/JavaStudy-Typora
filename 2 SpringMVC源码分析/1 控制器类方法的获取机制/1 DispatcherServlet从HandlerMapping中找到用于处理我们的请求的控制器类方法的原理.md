# 1.`DispatcherServlet`从`HandlerMapping`中找到用于处理我们的请求的控制器类方法的原理

## 1.1 `SpringBoot`中`SpringMVC`组件的五大`HandlerMapping`实现类

- `RequestMappingHandlerMapping`实现类
    - 这个实现类的实例中会存储我们当前项目中所有标注出来的`@RequestMapping`注解的详细信息及其所属的方法,我们用户的请求会经过该实现类实例判断,如果有匹配的上的`@RequestMapping`注解就会使用其对应的控制器类方法处理
- `WelcomPageHandlerMapping`实现类
    - 这个实现类是由我们的`SpringBoot`在`WebMvcConfiguration`类中注册为`Bean`的.当我们的`RequestMappingHandlerMapping`无法处理我们用户的请求,就会交由我们的这个实现类,如果这个实现类能够处理,那么就会由这个实现类给出的处理器进行处理
- `BeanNameUrlHandlerMapping`实现类
- `RounterFunctionMapping`实现类
- `SimpleUrlHandlerMapping`实现类

## 1.2 `RequestMappingHandlerMapping`寻找能够处理用户请求的处理器的原理源码分析

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230210231834133.png" alt="image-20230210231834133" style="zoom: 80%;" />

> `DispatcherServlet`类

```java
//DispatcherServlet的doDispatch方法调用自身的getHandler方法,想要根据我么用户的请求去找到对应的处理器
mappedHandler = getHandler(processedRequest);
```

```java
//DispatcherServlet要去遍历我们所有的HandlerMapping实现类去找到那个能处理我们用户请求的处理器
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

> `AbstractHandlerMapping`类

```java
//AbstractHandlerMapping的获取处理器的getHandler方法,这个方法用于寻找我们当前handlermapping实现类实例中有没有能够处理我们的用户请求的处理器,如果有就把它与它对应的拦截器等存入一个HandlerExecutionChain类的类实例中,并返回这个类实例,如果没有就返回null,null就会触发我们上面那个方法对下一个HandlerMapping实例的遍历
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    //这一步就去调用AbstractHandlerMapping的方法去获取处理器,实际上获取到的就是封装了我们的控制器类方法的Method反射的HandlerMethod类的类实例
    Object handler = getHandlerInternal(request);
    if (handler == null) {
        handler = getDefaultHandler();
    }
    if (handler == null) {
        return null;
    }
    //这一步主要是当我们使用遍历到BeanNameHandlerMapping时生效
    // Bean name or resolved handler?
    if (handler instanceof String) {
        String handlerName = (String) handler;
        handler = obtainApplicationContext().getBean(handlerName);
    }

    // Ensure presence of cached lookupPath for interceptors and others
    if (!ServletRequestPathUtils.hasCachedPath(request)) {
        initLookupPath(request);
    }

    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

    if (logger.isTraceEnabled()) {
        logger.trace("Mapped to " + handler);
    }
    else if (logger.isDebugEnabled() && !DispatcherType.ASYNC.equals(request.getDispatcherType())) {
        logger.debug("Mapped to " + executionChain.getHandler());
    }

    if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
        CorsConfiguration config = getCorsConfiguration(handler, request);
        if (getCorsConfigurationSource() != null) {
            CorsConfiguration globalConfig = getCorsConfigurationSource().getCorsConfiguration(request);
            config = (globalConfig != null ? globalConfig.combine(config) : config);
        }
        if (config != null) {
            config.validateAllowCredentials();
        }
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }

    return executionChain;
}
```

> `RequestMappingHandlerMapping`类

```java
//RequestMappingHandlerMapping实例的方法,用于寻找处理器
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    request.removeAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
    try {
        return super.getHandlerInternal(request);
    }
    finally {
        ProducesRequestCondition.clearMediaTypesAttribute(request);
    }
}
```

> `AbstractHandlerMethodMapping`类

```java
//RequestMappingHandlerMapping实例的方法的父类`RequestMappingInfoHandlerMapping的父类AbstractHandlerMethodMapping的获取处理器的方法
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    //获取我们的Request请求的请求URL
    String lookupPath = initLookupPath(request);
    this.mappingRegistry.acquireReadLock();
    try {
        //根据请求URL与我们的请求去寻找处理器
        HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
        return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
    }
    finally {
        this.mappingRegistry.releaseReadLock();
    }
}
```

![image-20230210232200198](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230210232200198.png)

```java
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
    List<Match> matches = new ArrayList<>();
    List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
    if (directPathMatches != null) {
        addMatchingMappings(directPathMatches, matches, request);
    }
    if (matches.isEmpty()) {
        //在这一步我们获取了我们的RequestMappingHandler实例中存储的所有的@RequestMapping,然后去依次查询它们找到能够对应上我们的请求URL的控制器类方法
        addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
    }
    //我们找到的matchs属性中就存储了所有的能够处理我们的用户请求的处理器封装成的Match对象,下面我们的SpringMVC将直接使用经过排序之后,matches集合中排第一个的Match对象
    if (!matches.isEmpty()) {
        Match bestMatch = matches.get(0);
        if (matches.size() > 1) {
            Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
            matches.sort(comparator);
            bestMatch = matches.get(0);
            if (logger.isTraceEnabled()) {
                logger.trace(matches.size() + " matching mappings: " + matches);
            }
            if (CorsUtils.isPreFlightRequest(request)) {
                for (Match match : matches) {
                    if (match.hasCorsConfig()) {
                        return PREFLIGHT_AMBIGUOUS_MATCH;
                    }
                }
            }
            else {
                Match secondBestMatch = matches.get(1);
                if (comparator.compare(bestMatch, secondBestMatch) == 0) {
                    Method m1 = bestMatch.getHandlerMethod().getMethod();
                    Method m2 = secondBestMatch.getHandlerMethod().getMethod();
                    String uri = request.getRequestURI();
                    throw new IllegalStateException(
                            "Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
                }
            }
        }
        request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.getHandlerMethod());
        handleMatch(bestMatch.mapping, lookupPath, request);
        //通过调用Match类的类实例的getHandlerMethod方法获取到封装了我们都控制器类方法的MeThod反射对象的HandlerMethod实例
        return bestMatch.getHandlerMethod();
    }
    else {
        return handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), lookupPath, request);
    }
}
```



## 1.3 `RequestMappingHandlerMapping`获取所有`@RequestMapping`注解的原理源码分析
