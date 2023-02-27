## 4 `DispatcherServlet`解析控制器类方法的返回值的工作原理

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230211014757984.png" alt="image-20230211014757984" style="zoom:80%;" />

## `SpringBoot`的`15`个返回值解析器

![image.png](https://cdn.nlark.com/yuque/0/2023/png/32659331/1675788244515-52fb92f8-2d68-4fb7-bcc3-04f0f433342f.png?x-oss-process=image%2Fresize%2Cw_515%2Climit_0)

## 前置工作部分

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
    //在这一步为我们封装好的ServletInvocableHandlerMethod类的类实例invocableMethod的returnValueHandlers属性进行了初始化,这个属性初始化后保存着我们SpringMVC的15个返回值解析器
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
    return getModelAndView(mavContainer, modelFactory, webRequest)
}
finally {
    webRequest.requestCompleted();
}
}
```

> `InvocableHandlerMethod`类

```java
private HandlerMethodArgumentResolverComposite resolvers
public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver
    
public void setHandlerMethodArgumentResolvers(HandlerMethodArgumentResolverComposite argumentResolvers) {
		this.resolvers = argumentResolvers;
	}
```

> `ServletInvocableHandlerMethod`类

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
       //就是在这一步我们的mavContianer存储好了我们后面要用于生成ModelAndView的一些数据
       //也是在这一步调用了我们returnValueHandlers属性这个HandlerMethodReturnValueHandlerComposite类的类实例的handleReturnValue方法来处理我们的返回值
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

## 真正解析封装我们的控制器类方法的返回值的部分

> `HandlerMethodReturnValueHandlerComposite`类

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
	//寻找解析器
    HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
    if (handler == null) {
        throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
    }
    //找到解析器后解析的过程
    handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

### 寻找解析器的部分

```java
private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
    boolean isAsyncValue = isAsyncReturnValue(value, returnType);
    for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
        if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
            continue;
        }
        if (handler.supportsReturnType(returnType)) {
            return handler;
        }
    }
    return null;
}
```

> `ModelAndViewMethodReturnValueHandler`类

```java
public boolean supportsReturnType(MethodParameter returnType) {
    return ModelAndView.class.isAssignableFrom(returnType.getParameterType());
}
```

### 找到了用于解析我们的返回值的解析器后的部分

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
        throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

    mavContainer.setRequestHandled(true);
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

    // Try even with null return value. ResponseBodyAdvice could get involved.
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

```java
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
        ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
        throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

    Object body;
    Class<?> valueType;
    Type targetType;

    if (value instanceof CharSequence) {
        body = value.toString();
        valueType = String.class;
        targetType = String.class;
    }
    else {
        body = value;
        valueType = getReturnValueType(body, returnType);
        targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
    }

    if (isResourceType(value, returnType)) {
        outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
        if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
                outputMessage.getServletResponse().getStatus() == 200) {
            Resource resource = (Resource) value;
            try {
                List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
                outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
                body = HttpRange.toResourceRegions(httpRanges, resource);
                valueType = body.getClass();
                targetType = RESOURCE_REGION_LIST_TYPE;
            }
            catch (IllegalArgumentException ex) {
                outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
                outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
            }
        }
    }

    MediaType selectedMediaType = null;
    MediaType contentType = outputMessage.getHeaders().getContentType();
    boolean isContentTypePreset = contentType != null && contentType.isConcrete();
    if (isContentTypePreset) {
        if (logger.isDebugEnabled()) {
            logger.debug("Found 'Content-Type:" + contentType + "' in response");
        }
        selectedMediaType = contentType;
    }
    else {
        HttpServletRequest request = inputMessage.getServletRequest();
        List<MediaType> acceptableTypes;
        try {
            acceptableTypes = getAcceptableMediaTypes(request);
        }
        catch (HttpMediaTypeNotAcceptableException ex) {
            int series = outputMessage.getServletResponse().getStatus() / 100;
            if (body == null || series == 4 || series == 5) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Ignoring error response content (if any). " + ex);
                }
                return;
            }
            throw ex;
        }
        List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

        if (body != null && producibleTypes.isEmpty()) {
            throw new HttpMessageNotWritableException(
                    "No converter found for return value of type: " + valueType);
        }
        List<MediaType> mediaTypesToUse = new ArrayList<>();
        for (MediaType requestedType : acceptableTypes) {
            for (MediaType producibleType : producibleTypes) {
                if (requestedType.isCompatibleWith(producibleType)) {
                    mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
                }
            }
        }
        if (mediaTypesToUse.isEmpty()) {
            if (logger.isDebugEnabled()) {
                logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
            }
            if (body != null) {
                throw new HttpMediaTypeNotAcceptableException(producibleTypes);
            }
            return;
        }

        MediaType.sortBySpecificityAndQuality(mediaTypesToUse);

        for (MediaType mediaType : mediaTypesToUse) {
            if (mediaType.isConcrete()) {
                selectedMediaType = mediaType;
                break;
            }
            else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
                selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
                break;
            }
        }

        if (logger.isDebugEnabled()) {
            logger.debug("Using '" + selectedMediaType + "', given " +
                    acceptableTypes + " and supported " + producibleTypes);
        }
    }

    if (selectedMediaType != null) {
        selectedMediaType = selectedMediaType.removeQualityValue();
        for (HttpMessageConverter<?> converter : this.messageConverters) {
            GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
                    (GenericHttpMessageConverter<?>) converter : null);
            if (genericConverter != null ?
                    ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
                    converter.canWrite(valueType, selectedMediaType)) {
                body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
                        (Class<? extends HttpMessageConverter<?>>) converter.getClass(),
                        inputMessage, outputMessage);
                if (body != null) {
                    Object theBody = body;
                    LogFormatUtils.traceDebug(logger, traceOn ->
                            "Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
                    addContentDispositionHeader(inputMessage, outputMessage);
                    if (genericConverter != null) {
                        genericConverter.write(body, targetType, selectedMediaType, outputMessage);
                    }
                    else {
                        ((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
                    }
                }
                else {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Nothing to write: null body");
                    }
                }
                return;
            }
        }
    }

    if (body != null) {
        Set<MediaType> producibleMediaTypes =
                (Set<MediaType>) inputMessage.getServletRequest()
                        .getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

        if (isContentTypePreset || !CollectionUtils.isEmpty(producibleMediaTypes)) {
            throw new HttpMessageNotWritableException(
                    "No converter for [" + valueType + "] with preset Content-Type '" + contentType + "'");
        }
        throw new HttpMediaTypeNotAcceptableException(getSupportedMediaTypes(body.getClass()));
    }
}
```

