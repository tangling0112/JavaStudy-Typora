# 0 `SpringBoot`自带的四大默认异常处理器

> 以下异常处理器是按照默认情况下的优先级排序的

- **`DefualtErrorAttribute`异常处理器**
    - 由于默认情况下这个异常处理器的优先级是最高的,因此在默认情况下任何异常都会要经过这个异常处理器的处理.
    - 而这个异常处理器只是把一些异常相关的数据存储到我们的`Request`请求域中.并且任何情况下都只会返回`null`而不会是返回一个`ModelAndView`
- **`ExceptionHandlerExceptionResolver`异常处理器**
    - 这个异常处理器主要是负责处理我们使用`@ControllerAdvice+@ExceptionHandle`注解定义的异常处理器.
    - 我们的这个异常处理器在处理异常时,会拿到我们的异常,以及引起这个异常的控制器类方法的`HandlerMethod`实例,然后就会去判断是否存在一个使用`@ExceptionHandle`注解注册的异常处理器的异常处理方法能够处理这个控制器类方法引起的对应异常类型的错误.
    - 如果没有任何一个使用`@ControllerAdvice+@ExceptionHandle`注解定义的异常处理器能够处理当前异常,那么我们的这个异常处理器会返回`null`
- **`ResponseStatusExceptionResolver`异常处理器**
    - 实现`@ResponseStatus`注解实现的异常处理器功能的异常处理器
- **`DefualtHandlerExceptionResolver`异常处理器**
    - 这个异常处理器会去判断我们的异常类型是否是我们的异常处理器中指定的一些类型,如果是就会根据不同的异常类型进行异常处理,并给出不同的`ModelAndView`实例
    - 当然我们应该知道基本上对于支持的异常类型进行处理时,我们的这个异常处理器都会在返回`ModelAndView`之前调用`response.sendError()`方法
    - 如果不是我们的这个异常处理器能够解决的一些异常类型,那么我们的这个异常处理器会直接返回`null`

![image-20230223112703230](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302231127291.png)

****

# 1 使用`SpringBoot`默认异常处理器处理异常的工作流程(`DefaultErrorAttribute`)

## `DispatcherServlet`

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   HttpServletRequest processedRequest = request;
   HandlerExecutionChain mappedHandler = null;
   boolean multipartRequestParsed = false;

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

   try {
      ModelAndView mv = null;
      Exception dispatchException = null;

      try {
         processedRequest = checkMultipart(request);
         multipartRequestParsed = (processedRequest != request);

         // Determine handler for the current request.
         mappedHandler = getHandler(processedRequest);
         if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);
            return;
         }

         // Determine handler adapter for the current request.
         HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

         // Process last-modified header, if supported by the handler.
         String method = request.getMethod();
         boolean isGet = HttpMethod.GET.matches(method);
         if (isGet || HttpMethod.HEAD.matches(method)) {
            long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
            if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
               return;
            }
         }

         if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
         }

         // Actually invoke the handler.
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

         if (asyncManager.isConcurrentHandlingStarted()) {
            return;
         }

         applyDefaultViewName(processedRequest, mv);
         mappedHandler.applyPostHandle(processedRequest, response, mv);
      }
      //如果上面的过程发生了异常报错那么就会捕获异常实例并交由我们后续的processDispatchResult方法处理
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {
         // As of 4.3, we're processing Errors thrown from handler methods as well,
         // making them available for @ExceptionHandler methods and other scenarios.
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
       //由于上面发生了报错因此实际上我们的mv变量的值为null
       //dispatchException为我们的错误实例
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {
      if (asyncManager.isConcurrentHandlingStarted()) {
         // Instead of postHandle and afterCompletion
         if (mappedHandler != null) {
            mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
         }
      }
      else {
         // Clean up any resources used by a multipart request.
         if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
         }
      }
   }
}
```

```java
//我们要注意到,我们的这个方法中是没有使用try-catch结构的,因此我们的processHandlerException方法一旦抛出异常就会直接异常传递到我们的doDispatch方法
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
      @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
      @Nullable Exception exception) throws Exception {

   boolean errorView = false;

   if (exception != null) {
      if (exception instanceof ModelAndViewDefiningException) {
         logger.debug("ModelAndViewDefiningException encountered", exception);
         mv = ((ModelAndViewDefiningException) exception).getModelAndView();
      }
      else {
         Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
          //在这一步就会调用我们的DispatcherServlet的processHandlerException方法去处理我们的异常
          //这里会有三种可能性
          	//没有任何异常处理器可以处理我们的异常,因此mv依旧为空
          	//只有我们的SpringBoot的DefaultErrorAttribute异常处理器可以处理我们的异常,但是我们知道,这个异常处理器的返回值为null
          	//存在异常处理器能够处理我们的异常,并最终返回一个ModelAndView实例
         mv = processHandlerException(request, response, handler, exception);
         errorView = (mv != null);
      }
   }
	//如果我们的mv变量不为空且是一个有实际意义的ModelAndView实例,那么就会根据我们的ModelAndView去渲染我们的视图
   if (mv != null && !mv.wasCleared()) {
      render(mv, request, response);
      if (errorView) {
         WebUtils.clearErrorRequestAttributes(request);
      }
   }
   else {
      if (logger.isTraceEnabled()) {
         logger.trace("No view rendering, null ModelAndView returned.");
      }
   }

   if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
      // Concurrent handling started during a forward
      return;
   }

   if (mappedHandler != null) {
      // Exception (if any) is already handled..
      mappedHandler.triggerAfterCompletion(request, response, null);
   }
}
```

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
      @Nullable Object handler, Exception ex) throws Exception {

   // Success and error responses may use different content types
   request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

   // Check registered HandlerExceptionResolvers...
   ModelAndView exMv = null;
   if (this.handlerExceptionResolvers != null) {
       //遍历我们的DispatcherServlet的handlerExceptionResolvers存储的所有的异常处理器,找到第一个能够给出一个ModelAndView实例而不是返回空的异常处理器,并接收这个ModelAndView实例然后处理
       //由于我们讨论的就是DefaultErrorAttribute异常处理器,因此后面的代码讲述的就是这个异常处理器的resolveException方法做的事情
       //由于我们的DefaultErrorAttribute异常处理器在任何情况下都会返回null,因此经过这个异常处理器处理完毕后还是会遍历后续的异常处理器
      for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
         exMv = resolver.resolveException(request, response, handler, ex);
         if (exMv != null) {
            break;
         }
      }
   }
   if (exMv != null) {
      if (exMv.isEmpty()) {
         request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
         return null;
      }
      // We might still need view name translation for a plain error model...
      if (!exMv.hasView()) {
         String defaultViewName = getDefaultViewName(request);
         if (defaultViewName != null) {
            exMv.setViewName(defaultViewName);
         }
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Using resolved error view: " + exMv, ex);
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Using resolved error view: " + exMv);
      }
      WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
      return exMv;
   }
	//当没有任何一个异常处理器能够处理我们的异常的时候就会抛出异常给我们的processDispatchResult方法,然后其又会把错误抛出给我们的doDispatch方法,然后我们的请求处理流程就会不正常结束,并且后续我们的SpringMVC会将我们的请求转发给我们的/error映射
    //这是一个复杂的过程,我们只要知道会有这样的请求转发到/error映射的流程即可
   throw ex;
}
```

## `DefualtErrorAttribute`

```java
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler,
      Exception ex) {
   storeErrorAttributes(request, ex);
   return null;
}
```

```java
private void storeErrorAttributes(HttpServletRequest request, Exception ex) {
   request.setAttribute(ERROR_INTERNAL_ATTRIBUTE, ex);
}
```

```java
private static final String ERROR_INTERNAL_ATTRIBUTE = DefaultErrorAttributes.class.getName() + ".ERROR";
```

****

# 2 使用基于`SpringMVC`的`@ControllerAdvice+@ExceptionHandler`注解配置的异常处理器处理异常的工作流程(`ExceptionHandlerExceptionResolver`)

> 前面的内容我们不再多说,直接正题

## `DispatcherServlet`类

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
      @Nullable Object handler, Exception ex) throws Exception {

   // Success and error responses may use different content types
   request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

   // Check registered HandlerExceptionResolvers...
   ModelAndView exMv = null;
   if (this.handlerExceptionResolvers != null) {
       //遍历我们的DispatcherServlet的handlerExceptionResolvers存储的所有的异常处理器,找到第一个能够给出一个ModelAndView实例而不是返回空的异常处理器,并接收这个ModelAndView实例然后处理
       //由于我们讨论的就是DefaultErrorAttribute异常处理器,因此后面的代码讲述的就是这个异常处理器的resolveException方法做的事情
       //由于我们的DefaultErrorAttribute异常处理器在任何情况下都会返回null,因此经过这个异常处理器处理完毕后还是会遍历后续的异常处理器
      for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
         exMv = resolver.resolveException(request, response, handler, ex);
         if (exMv != null) {
            break;
         }
      }
   }
   if (exMv != null) {
      if (exMv.isEmpty()) {
         request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
         return null;
      }
      // We might still need view name translation for a plain error model...
      if (!exMv.hasView()) {
         String defaultViewName = getDefaultViewName(request);
         if (defaultViewName != null) {
            exMv.setViewName(defaultViewName);
         }
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Using resolved error view: " + exMv, ex);
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Using resolved error view: " + exMv);
      }
      WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
      return exMv;
   }
	//当没有任何一个异常处理器能够处理我们的异常的时候就会抛出异常给我们的processDispatchResult方法,然后其又会把错误抛出给我们的doDispatch方法,然后我们的请求处理流程就会不正常结束,并且后续我们的SpringMVC会将我们的请求转发给我们的/error映射
    //这是一个复杂的过程,我们只要知道会有这样的请求转发到/error映射的流程即可
   throw ex;
}
```

![image-20230223155246267](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302231552390.png)

## `AbstractHandlerExceptionResolver`类

```java
//这个方法是我们的ExceptionhandlerExceptionResolver实例从AbstractHandlerExceptionResolver继承的
public ModelAndView resolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   if (shouldApplyTo(request, handler)) {
      prepareResponse(ex, response);
       //调用doResolveException进行异常处理
      ModelAndView result = doResolveException(request, response, handler, ex);
      if (result != null) {
         if (logger.isDebugEnabled() && (this.warnLogger == null || !this.warnLogger.isWarnEnabled())) {
            logger.debug(buildLogMessage(ex, request) + (result.isEmpty() ? "" : " to " + result));
         }
         logException(ex, request);
      }
      return result;
   }
   else {
      return null;
   }
}
```

## `AbstractHandlerMethodExceptionResolver`类

```java
//这个方法是ExceptionhandlerExceptionResolver实例从AbstractHandlerMethodExceptionResolver类继承的
protected final ModelAndView doResolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   HandlerMethod handlerMethod = (handler instanceof HandlerMethod ? (HandlerMethod) handler : null);
    //调用doResolveHandlerMethodException进行异常处理
   return doResolveHandlerMethodException(request, response, handlerMethod, ex);
}
```

## `ExceptionHandlerExceptionResolver`类

```java
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
      HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {
	//获取到我们使用@ControllerAdvice+@ExceptionHandler注册的异常处理器的异常处理方法的ServletInvocableHandlerMethod实例,以便于我们后续进行异常处理
   ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
   if (exceptionHandlerMethod == null) {
      return null;
   }
	//给我们的ServletInvocableHandlerMethod实例设置用于处理异常处理器的异常处理方法的参数解析以及返回值解析的参数解析器与返回值解析器
   if (this.argumentResolvers != null) {
      exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
   }
   if (this.returnValueHandlers != null) {
      exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
   }

   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   ModelAndViewContainer mavContainer = new ModelAndViewContainer();

   ArrayList<Throwable> exceptions = new ArrayList<>();
   try {
      // Expose causes as provided arguments as well
      Throwable exToExpose = exception;
      while (exToExpose != null) {
         exceptions.add(exToExpose);
         Throwable cause = exToExpose.getCause();
         exToExpose = (cause != exToExpose ? cause : null);
      }
       //构建我们的异常数组
       //为什么我们的异常只有一个,我们还要构建一个数组去容纳他呢?
       		//原因在于我们的ServletInvocableHandlerMethod类的invokeAndHandle方法是通过Object...这个可变形参来接收我们的这个参数的,因此我们必须要传递一个数组过去才能保证实现正确的接收
      Object[] arguments = new Object[exceptions.size() + 1];
      exceptions.toArray(arguments);  // efficient arraycopy call in ArrayList
      arguments[arguments.length - 1] = handlerMethod;
       
       //这个invokeAndHandle就是我们前面分析我们的控制器类方法的获取,调用,返回值处理的那一部分的那个invokeAndHandle方法
       //这个方法会解析我们的异常处理器的异常处理方法的形参列表,使用参数解析器依次得到这些形参应该传入的实际参数的值,并且调用我们的异常处理器的异常处理方法,并且对调用得到的返回值像调用控制器类方法的返回值一样使用返回值解析器进行解析
       //注意:这里的arguments是一个存储了我们的异常实例的一个数组,传递这个参数的目的在于,我们通过@ControllerAdvice+@ExceptionHandler注解构建的异常处理器的异常处理方法是能够接收到我们引起异常处理的异常实例的.因此需要传入这个数据来保证这个功能得以实现.
       
       //由于这个方法的工作流程我们前面讨论控制器类方法的调用时就分析过,因此这里我们不再分析
      exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, arguments);
       
   }
    
    //当我们的异常处理器的异常处理方法发生报错时就会直接return null,也就是说我们的异常没有被我们的ExceptionHandlerExceptionResolver所处理
   catch (Throwable invocationEx) {
      // Any other than the original exception (or a cause) is unintended here,
      // probably an accident (e.g. failed assertion or the like).
      if (!exceptions.contains(invocationEx) && logger.isWarnEnabled()) {
         logger.warn("Failure in @ExceptionHandler " + exceptionHandlerMethod, invocationEx);
      }
      // Continue with default processing of the original exception...
      return null;
   }
    //正常情况下这一部分是不执行的
   if (mavContainer.isRequestHandled()) {
      return new ModelAndView();
   }
    //这一步就会根据我们的异常处理器的异常处理方法的处理结果去生成我们的ModelAndView实例,然后去返回给我们的doDispatch方法,以完成异常处理
   else {
      ModelMap model = mavContainer.getModel();
      HttpStatus status = mavContainer.getStatus();
      ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, status);
      mav.setViewName(mavContainer.getViewName());
      if (!mavContainer.isViewReference()) {
         mav.setView((View) mavContainer.getView());
      }
      if (model instanceof RedirectAttributes) {
         Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
         RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
      }
      return mav;
   }
}
```

****

# 3 使用基于`SpringMVC`的`@ResponseStatus`注解或`ResponseStatusException`类配置的异常处理器处理异常的工作流程(`ResponseStatusExceptionResolver`)

> 前面的内容我们不再多说,直接正题

## `DispatcherServlet`类

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
      @Nullable Object handler, Exception ex) throws Exception {

   // Success and error responses may use different content types
   request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

   // Check registered HandlerExceptionResolvers...
   ModelAndView exMv = null;
   if (this.handlerExceptionResolvers != null) {
       //遍历我们的DispatcherServlet的handlerExceptionResolvers存储的所有的异常处理器,找到第一个能够给出一个ModelAndView实例而不是返回空的异常处理器,并接收这个ModelAndView实例然后处理
       //由于我们讨论的就是DefaultErrorAttribute异常处理器,因此后面的代码讲述的就是这个异常处理器的resolveException方法做的事情
       //由于我们的DefaultErrorAttribute异常处理器在任何情况下都会返回null,因此经过这个异常处理器处理完毕后还是会遍历后续的异常处理器
      for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
         exMv = resolver.resolveException(request, response, handler, ex);
         if (exMv != null) {
            break;
         }
      }
   }
   if (exMv != null) {
      if (exMv.isEmpty()) {
         request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
         return null;
      }
      // We might still need view name translation for a plain error model...
      if (!exMv.hasView()) {
         String defaultViewName = getDefaultViewName(request);
         if (defaultViewName != null) {
            exMv.setViewName(defaultViewName);
         }
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Using resolved error view: " + exMv, ex);
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Using resolved error view: " + exMv);
      }
      WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
      return exMv;
   }
	//当没有任何一个异常处理器能够处理我们的异常的时候就会抛出异常给我们的processDispatchResult方法,然后其又会把错误抛出给我们的doDispatch方法,然后我们的请求处理流程就会不正常结束,并且后续我们的SpringMVC会将我们的请求转发给我们的/error映射
    //这是一个复杂的过程,我们只要知道会有这样的请求转发到/error映射的流程即可
   throw ex;
}
```

## `AbstractHandlerExceptionResolver`类

```java
//这个方法是我们的ResponseStatusExceptionResolver实例从AbstractHandlerExceptionResolver继承的
public ModelAndView resolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   if (shouldApplyTo(request, handler)) {
      prepareResponse(ex, response);
       //调用doResolveException进行异常处理
      ModelAndView result = doResolveException(request, response, handler, ex);
       
      if (result != null) {
         if (logger.isDebugEnabled() && (this.warnLogger == null || !this.warnLogger.isWarnEnabled())) {
            logger.debug(buildLogMessage(ex, request) + (result.isEmpty() ? "" : " to " + result));
         }
         logException(ex, request);
      }
      return result;
   }
   else {
      return null;
   }
}
```

## `ResponseStatusExceptionResolver`类

```java
protected ModelAndView doResolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   try {
      if (ex instanceof ResponseStatusException) {
         return resolveResponseStatusException((ResponseStatusException) ex, request, response, handler);
      }

      ResponseStatus status = AnnotatedElementUtils.findMergedAnnotation(ex.getClass(), ResponseStatus.class);
      if (status != null) {
         return resolveResponseStatus(status, request, response, handler, ex);
      }

      if (ex.getCause() instanceof Exception) {
         return doResolveException(request, response, handler, (Exception) ex.getCause());
      }
   }
   catch (Exception resolveEx) {
      if (logger.isWarnEnabled()) {
         logger.warn("Failure while trying to resolve exception [" + ex.getClass().getName() + "]", resolveEx);
      }
   }
   return null;
}
```

```java
protected ModelAndView resolveResponseStatusException(ResponseStatusException ex,
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler) throws Exception {

   ex.getResponseHeaders().forEach((name, values) ->
         values.forEach(value -> response.addHeader(name, value)));

   return applyStatusAndReason(ex.getRawStatusCode(), ex.getReason(), response);
}
```

```java
protected ModelAndView resolveResponseStatus(ResponseStatus responseStatus, HttpServletRequest request,
      HttpServletResponse response, @Nullable Object handler, Exception ex) throws Exception {

   int statusCode = responseStatus.code().value();
   String reason = responseStatus.reason();
   return applyStatusAndReason(statusCode, reason, response);
}
```

****

# 4 `DefaultHandlerExceptionResolver`

> 前面的内容我们不再多说,直接正题

## `DispatcherServlet`类

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
      @Nullable Object handler, Exception ex) throws Exception {

   // Success and error responses may use different content types
   request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

   // Check registered HandlerExceptionResolvers...
   ModelAndView exMv = null;
   if (this.handlerExceptionResolvers != null) {
       //遍历我们的DispatcherServlet的handlerExceptionResolvers存储的所有的异常处理器,找到第一个能够给出一个ModelAndView实例而不是返回空的异常处理器,并接收这个ModelAndView实例然后处理
       //由于我们讨论的就是DefaultErrorAttribute异常处理器,因此后面的代码讲述的就是这个异常处理器的resolveException方法做的事情
       //由于我们的DefaultErrorAttribute异常处理器在任何情况下都会返回null,因此经过这个异常处理器处理完毕后还是会遍历后续的异常处理器
      for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
         exMv = resolver.resolveException(request, response, handler, ex);
         if (exMv != null) {
            break;
         }
      }
   }
   if (exMv != null) {
      if (exMv.isEmpty()) {
         request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
         return null;
      }
      // We might still need view name translation for a plain error model...
      if (!exMv.hasView()) {
         String defaultViewName = getDefaultViewName(request);
         if (defaultViewName != null) {
            exMv.setViewName(defaultViewName);
         }
      }
      if (logger.isTraceEnabled()) {
         logger.trace("Using resolved error view: " + exMv, ex);
      }
      else if (logger.isDebugEnabled()) {
         logger.debug("Using resolved error view: " + exMv);
      }
      WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
      return exMv;
   }
	//当没有任何一个异常处理器能够处理我们的异常的时候就会抛出异常给我们的processDispatchResult方法,然后其又会把错误抛出给我们的doDispatch方法,然后我们的请求处理流程就会不正常结束,并且后续我们的SpringMVC会将我们的请求转发给我们的/error映射
    //这是一个复杂的过程,我们只要知道会有这样的请求转发到/error映射的流程即可
   throw ex;
}
```

## `AbstractHandlerExceptionResolver`类

```java
//这个方法是我们的ResponseStatusExceptionResolver实例从AbstractHandlerExceptionResolver继承的
public ModelAndView resolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   if (shouldApplyTo(request, handler)) {
      prepareResponse(ex, response);
       //调用doResolveException进行异常处理
      ModelAndView result = doResolveException(request, response, handler, ex);
       
      if (result != null) {
         if (logger.isDebugEnabled() && (this.warnLogger == null || !this.warnLogger.isWarnEnabled())) {
            logger.debug(buildLogMessage(ex, request) + (result.isEmpty() ? "" : " to " + result));
         }
         logException(ex, request);
      }
      return result;
   }
   else {
      return null;
   }
}
```

## `DefualtHandlerExceptionResolver`

```java
protected ModelAndView doResolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   try {
      if (ex instanceof HttpRequestMethodNotSupportedException) {
         return handleHttpRequestMethodNotSupported(
               (HttpRequestMethodNotSupportedException) ex, request, response, handler);
      }
      else if (ex instanceof HttpMediaTypeNotSupportedException) {
         return handleHttpMediaTypeNotSupported(
               (HttpMediaTypeNotSupportedException) ex, request, response, handler);
      }
      else if (ex instanceof HttpMediaTypeNotAcceptableException) {
         return handleHttpMediaTypeNotAcceptable(
               (HttpMediaTypeNotAcceptableException) ex, request, response, handler);
      }
      else if (ex instanceof MissingPathVariableException) {
         return handleMissingPathVariable(
               (MissingPathVariableException) ex, request, response, handler);
      }
      else if (ex instanceof MissingServletRequestParameterException) {
         return handleMissingServletRequestParameter(
               (MissingServletRequestParameterException) ex, request, response, handler);
      }
      else if (ex instanceof ServletRequestBindingException) {
         return handleServletRequestBindingException(
               (ServletRequestBindingException) ex, request, response, handler);
      }
      else if (ex instanceof ConversionNotSupportedException) {
         return handleConversionNotSupported(
               (ConversionNotSupportedException) ex, request, response, handler);
      }
      else if (ex instanceof TypeMismatchException) {
         return handleTypeMismatch(
               (TypeMismatchException) ex, request, response, handler);
      }
      else if (ex instanceof HttpMessageNotReadableException) {
         return handleHttpMessageNotReadable(
               (HttpMessageNotReadableException) ex, request, response, handler);
      }
      else if (ex instanceof HttpMessageNotWritableException) {
         return handleHttpMessageNotWritable(
               (HttpMessageNotWritableException) ex, request, response, handler);
      }
      else if (ex instanceof MethodArgumentNotValidException) {
         return handleMethodArgumentNotValidException(
               (MethodArgumentNotValidException) ex, request, response, handler);
      }
      else if (ex instanceof MissingServletRequestPartException) {
         return handleMissingServletRequestPartException(
               (MissingServletRequestPartException) ex, request, response, handler);
      }
      else if (ex instanceof BindException) {
         return handleBindException((BindException) ex, request, response, handler);
      }
      else if (ex instanceof NoHandlerFoundException) {
         return handleNoHandlerFoundException(
               (NoHandlerFoundException) ex, request, response, handler);
      }
      else if (ex instanceof AsyncRequestTimeoutException) {
         return handleAsyncRequestTimeoutException(
               (AsyncRequestTimeoutException) ex, request, response, handler);
      }
   }
   catch (Exception handlerEx) {
      if (logger.isWarnEnabled()) {
         logger.warn("Failure while trying to resolve exception [" + ex.getClass().getName() + "]", handlerEx);
      }
   }
   return null;
}
```

```java
protected ModelAndView handleHttpRequestMethodNotSupported(HttpRequestMethodNotSupportedException ex,
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler) throws IOException {

   String[] supportedMethods = ex.getSupportedMethods();
   if (supportedMethods != null) {
      response.setHeader("Allow", StringUtils.arrayToDelimitedString(supportedMethods, ", "));
   }
   response.sendError(HttpServletResponse.SC_METHOD_NOT_ALLOWED, ex.getMessage());
   return new ModelAndView();
}
```

```java
protected ModelAndView handleHttpMediaTypeNotSupported(HttpMediaTypeNotSupportedException ex,
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler) throws IOException {

   List<MediaType> mediaTypes = ex.getSupportedMediaTypes();
   if (!CollectionUtils.isEmpty(mediaTypes)) {
      response.setHeader("Accept", MediaType.toString(mediaTypes));
      if (request.getMethod().equals("PATCH")) {
         response.setHeader("Accept-Patch", MediaType.toString(mediaTypes));
      }
   }
   response.sendError(HttpServletResponse.SC_UNSUPPORTED_MEDIA_TYPE);
   return new ModelAndView();
}
```
