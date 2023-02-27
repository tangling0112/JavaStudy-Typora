# 3.`DispatcherServlet`解析控制器类方法的形式参数的工作原理

## 3.1`DispatcherServlet`的`26`个控制器类方法参数解析器

<img src="https://cdn.nlark.com/yuque/0/2023/png/32659331/1675788070750-b06edb3a-565c-4b4a-945d-0f7f67effdfc.png?x-oss-process=image%2Fresize%2Cw_550%2Climit_0" alt="image.png" style="zoom:67%;" />

## 3.2 `DispatcherServlet`为每个我们的控制器类方法的形式参数确定用于解析它的参数解析器的工作原理

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230211014757984.png" alt="image-20230211014757984" style="zoom:80%;" />

> `DispatcherServlet`类

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

> `AbstractHandlerMethodAdapter`类
>
> - 我们的`ha`是一个`RequestMappingHandlerAdapter`实例,而`AbstractHandlerMethodAdapter`是其父类

```java
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
        throws Exception {
    return handleInternal(request, response, (HandlerMethod) handler);
}
```

> `RequestMappingHandlerAdapter`类

```java
protected ModelAndView handleInternal(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ModelAndView mav;
    checkRequest(request);

    // 当我们的RequestMappingHandlerAdapter的Bean对象的synchronizeOnSession为true时以加锁的方式处理我们的业务吗,当然这个属性默认为false
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        }
        else {
            // No HttpSession available -> no mutex necessary
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
    }
    else {
        // 不加锁只需
        mav = invokeHandlerMethod(request, response, handlerMethod);
    }

    if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
        if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
            applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
        }
        else {
            prepareResponse(response);
        }
    }

    return mav;
}
```

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
    HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

ServletWebRequest webRequest = new ServletWebRequest(request, response);
try {
    WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
    ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
	//在这一步将我们的封装了我们要调用的控制器类方法的Method反射实例的HandlerMethod类的类实例handlerMethod又封装成了ServletInvocableHandlerMethod类的类实例
    ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
    //在这一步为我们封装好的ServletInvocableHandlerMethod类的类实例invocableMethod的resolvers属性进行了初始化,这个属性初始化后保存着我们SpringMVC的26个参数解析器
    if (this.argumentResolvers != null) {
        invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
    }
    if (this.returnValueHandlers != null) {
        invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
    }
    invocableMethod.setDataBinderFactory(binderFactory);
    invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

    ModelAndViewContainer mavContainer = new ModelAndViewContainer();
    mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
    modelFactory.initModel(webRequest, mavContainer, invocableMethod);
    mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

    AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
    asyncWebRequest.setTimeout(this.asyncRequestTimeout);

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    asyncManager.setTaskExecutor(this.taskExecutor);
    asyncManager.setAsyncWebRequest(asyncWebRequest);
    asyncManager.registerCallableInterceptors(this.callableInterceptors);
    asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

    if (asyncManager.hasConcurrentResult()) {
        Object result = asyncManager.getConcurrentResult();
        mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
        asyncManager.clearConcurrentResult();
        LogFormatUtils.traceDebug(logger, traceOn -> {
            String formatted = LogFormatUtils.formatValue(result, !traceOn);
            return "Resume with async result [" + formatted + "]";
        });
        invocableMethod = invocableMethod.wrapConcurrentResult(result);
    }
    //这个invocableMethod就是我们前面提到的封装了HandlerMethod类实例的参数,并且在上面的环节我们还知道,他的resolver属性还存储了26中参数解析器
	//在这一步就是真正地去决定我们的控制器类方法的每个参数要使用什么参数解析器解析,并应用这些解析得到每个形参应该被赋予的实际参数值
    //注意这里我们传入了一个mavContainer和webRequest参数其是一个ModelAndViewContainer的实例,后续他会带着我们的控制类方法成功执行后的相关数据回来,以便我们后续获取ModelAndView实例
    invocableMethod.invokeAndHandle(webRequest, mavContainer);
    if (asyncManager.isConcurrentHandlingStarted()) {
        return null;
    }
	//获取我们的ModelAndView实例
    return getModelAndView(mavContainer, modelFactory, webRequest);
}
finally {
    webRequest.requestCompleted();
}
}
```

### 为`invocableMethod`变量的`resolvers`属性初始化的部分

> `InvocableHandlerMethod`类
>
> 为`invocableMethod`变量的`resolvers`属性存入存储着我们的`26`个参数解析器的`HandlerMethodArgumentResolverComposite`实例

```java
private HandlerMethodArgumentResolverComposite resolvers
public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver
    
public void setHandlerMethodArgumentResolvers(HandlerMethodArgumentResolverComposite argumentResolvers) {
		this.resolvers = argumentResolvers;
	}
```

### 真正解析参数的部分

> `ServletInvocableHandlerMethod`类
>
> - 决定参数解析

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {
	//在这一步就使用ServletInvocableHandlerMethod类的invokeForRequest方法去解析我们的控制类类方法的参数并调用了它,并且获取到了这个控制类方法调用后的返回值,存储到returnValue变量中
   Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
   setResponseStatus(webRequest);

   if (returnValue == null) {
      if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
         disableContentCachingIfNecessary(webRequest);
         mavContainer.setRequestHandled(true);
         return;
      }
   }
   else if (StringUtils.hasText(getResponseStatusReason())) {
      mavContainer.setRequestHandled(true);
      return;
   }

   mavContainer.setRequestHandled(false);
   Assert.state(this.returnValueHandlers != null, "No return value handlers");
   try {
       //就是在这一步我们的mavContianer存储好了我们的的生成ModelAndView的一些数据
      this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
   }
   catch (Exception ex) {
      if (logger.isTraceEnabled()) {
         logger.trace(formatErrorForReturnValue(returnValue), ex);
      }
      throw ex;
   }
}
```

```java
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {
	// 获取每个形参应该传入的实际参数的值
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    if (logger.isTraceEnabled()) {
        logger.trace("Arguments: " + Arrays.toString(args));
    }
    //真正地通过反射去调用我们的控制器类方法
    return doInvoke(args);
}
```

### 获取每个形参应该传入的实际参数的值的部分

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {
	//获取我们的控制器类方法的各个形参的详细信息,并分别封装在MethodParameter实例中
    MethodParameter[] parameters = getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    }
	//这个args数组后续会存储我们每一个参数的实际参数的值
    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }
        //遍历所有的参数解析器,看存不存在一个能够解析我们的这个参数的参数解析器,如果有就进行后续解析操作,如果没有就抛异常
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
        }
        try {
		//使用参数解析器去解析我们的这个参数,并将其解析完成后得到的应该传入的实际参数的值存储在args数组的指定位置
            args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
        }
        catch (Exception ex) {
            if (logger.isDebugEnabled()) {
                String exMsg = ex.getMessage();
                if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
                    logger.debug(formatArgumentError(parameter, exMsg));
                }
            }
            throw ex;
        }
    }
    //返回我们实际应该传入的实际参数的值的数组
    return args;
}
```

> `HandlerMethodArgumentResolverComposite`类
>
> - 我们的`ServletInvocalbleMethod`的`resolver`属性就是这个类的类实例,这个实例存储着所有的`26`个参数解析器
> - `supportsParameter`是查看是否存在任何一个解析器能够解析我们的这个参数

```java
public boolean supportsParameter(MethodParameter parameter) {
    return getArgumentResolver(parameter) != null;
}
```

```java
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
    if (result == null) {
        for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, result);
                break;
            }
        }
    }
    return result;
}
```

> `RequestParamMethodArgumentResolver`类

```java
public boolean supportsParameter(MethodParameter parameter) {
    if (parameter.hasParameterAnnotation(RequestParam.class)) {
        if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
            RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
            return (requestParam != null && StringUtils.hasText(requestParam.name()));
        }
        else {
            return true;
        }
    }
    else {
        if (parameter.hasParameterAnnotation(RequestPart.class)) {
            return false;
        }
        parameter = parameter.nestedIfOptional();
        if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
            return true;
        }
        else if (this.useDefaultResolution) {
            return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
        }
        else {
            return false;
        }
    }
}
```

> `HandlerMethodArgumentResolverComposite`类

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
   if (resolver == null) {
      throw new IllegalArgumentException("Unsupported parameter type [" +
            parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
   }
   return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

> 循环结束,每个形式参数都使用其对应的解析器解析好了,并且将其应该被赋予的实际参数值存入了`args`数组中

![image-20230211020244854](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230211020244854.png)

### 真正地去调用我们的控制器类方法的部分

> `InvocableHandlerMethod`类

```java
protected Object doInvoke(Object... args) throws Exception {
   Method method = getBridgedMethod();
   try {
      if (KotlinDetector.isSuspendingFunction(method)) {
         return CoroutinesUtils.invokeSuspendingFunction(method, getBean(), args);
      }
      return method.invoke(getBean(), args);
   }
   catch (IllegalArgumentException ex) {
      assertTargetBean(method, getBean(), args);
      String text = (ex.getMessage() != null ? ex.getMessage() : "Illegal argument");
      throw new IllegalStateException(formatInvokeError(text, args), ex);
   }
   catch (InvocationTargetException ex) {
      // Unwrap for HandlerExceptionResolvers ...
      Throwable targetException = ex.getTargetException();
      if (targetException instanceof RuntimeException) {
         throw (RuntimeException) targetException;
      }
      else if (targetException instanceof Error) {
         throw (Error) targetException;
      }
      else if (targetException instanceof Exception) {
         throw (Exception) targetException;
      }
      else {
         throw new IllegalStateException(formatInvokeError("Invocation failure", args), targetException);
      }
   }
}
```

# 4.`DispatcherServlet`解析控制器类方法的返回值的工作原理

# 5.`HttpMessageConverter`的工作原理

# 6.内容协商原理

