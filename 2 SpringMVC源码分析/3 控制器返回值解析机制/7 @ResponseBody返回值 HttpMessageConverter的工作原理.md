# `HttpMessageConverter`的作用(只有在解析被`@ResponseBody`注解标识的返回值时会被使用到,其他的不会使用到,是`RequestResponseBodyMethodOrocessor`专属)

> 我们知道我们的控制器类方法可以有多种不同的返回值
>
> - 当不使用`@ResponseBody`注解时
>     - 我们的控制器类方法只能传递如下三种字符串类型的返回值
>         - `“index”`普通视图
>         - `“forward:index.html”`请求转发视图
>         - `“redirect:/test”`重定向视图
>
> - 当我们使用`@ResponseBody`注解时
>     - 我们的控制器类方法能够传递任何`java`数据类型的返回值,并且我们的`SpringMVC`会自动将其转换为客户端可以接收的某种格式,然后放入我们的响应体中传回给我们的用户
>
>
> **我们的`HttpMessageConverter`就是用于实现上面我们提到的对于使用`@ResponseBody`注解时将任何`Java`数据类型解析为客户端可以接收的某种格式的一个转换器.**
>
> <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230213004316429.png" alt="image-20230213004316429" style="zoom:80%;" />

# `HttpMassageConverter`的被使用到的流程

<img src="./../../../AppData/Roaming/Typora/typora-user-images/image-20230213000521777.png" alt="image-20230213000521777" style="zoom:80%;" />

> ​		前面的过程在我们的解析控制器返回值那一节已经呈现,我们不再重复展示.直接从返回值解析器获取,返回值真正解析部分开始

## `InvocableHandlerMethod`实例

> `ServletInvocableHandlerMethod`类

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {
	//调用我们的控制器类方法并获取到控制器类方法调用后的返回值
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
   try {
       //这里的this是我们的InvocableHandlerMethod实例
       //returnValueHandlers是其内的属性,其存储的是一个存储着所有的返回值解析器的HandlerMethodReturnValueHandlerComposite类的实例
       //对我们的控制器类方法的返回值进行解析
      this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
   }
}
```

## `HandlerMethodReturnValueHandlerComposite`实例

> `HandlerMethodReturnValueHandlerComposite`类

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
	//根据我们的返回值以及返回值类型去查找我们的所有返回值解析器,找到用于解析这个类型的返回值的解析器的HandlerMethodReturnValueHandler类实例
   HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
   if (handler == null) {
      throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
   }
    //使用我们得到的返回值解析器去解析我们的返回值
    //returnValue是我们的返回值
    //returnType是包含了我们的返回值对应的控制器类方法的详细信息的封装类
   handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

## 寻找返回值解析器的部分

> `HandlerMethodReturnValueHandlerComposite`类

```java
private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
   boolean isAsyncValue = isAsyncReturnValue(value, returnType);
    //遍历HandlerMethodReturnValueHandlerComposite类实例中的returnValueHandlers属性所存储的所有返回值参数解析器
   for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
      if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
         continue;
      }
       //调用参数解析器的supportsReturnType方法,看当前返回值解析器是否支持解析我们的返回值
       //我们这里使用了@ResponseBody注解,因此一定会使用RequestResponseBodyMethodProcessor返回值解析器
      if (handler.supportsReturnType(returnType)) {
         return handler;
      }
   }
   return null;
}
```

## `RequestResponseBodyMethodProcessor`实例

> `RequestResponseBodyMethodProcessor`类

```java
public boolean supportsReturnType(MethodParameter returnType) {
    //若我们的返回值对应的控制器类方法使用了@ResponseBody注解修饰就返回true
    	//这里两个一个是处理类方法直接被@ResponseBody修饰的情况,一个是处理类方法所在类使用@RestController修饰的情况
   return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
         returnType.hasMethodAnnotation(ResponseBody.class));
}
```

## 真正解析返回值的部分

> `RequestResponseBodyMethodProcessor`类

```java
//returnValue是我们的返回值
//returnType是包含了我们的返回值对应的控制器类方法的详细信息的封装类
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   mavContainer.setRequestHandled(true);
    //这里调用的两个方法都是我们的RequestResponseBodyMethodProcessor返回值解析器内部的方法,当然,是从其父类继承得到的
    //通过这两个方法,从我们的webRequest变量中获取了我们的一些数据,并且是引用类型的,从而生成了inputMessage和outputMessage两个实例.我们对这两个实例的修改会由于数据的引用,同时作用到我们的webRequest变量上
   ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
   ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

   // Try even with null return value. ResponseBodyAdvice could get involved.
    //returnValue是我们的返回值
    //returnType是包含了我们的返回值对应的控制器类方法的详细信息的封装类
   writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

### 获取`inputMessage`和`outputMessage`部分

> `AbstractMessageConverterMethodArgumentResolver`类

```java
protected ServletServerHttpRequest createInputMessage(NativeWebRequest webRequest) {
   HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
   Assert.state(servletRequest != null, "No HttpServletRequest");
   return new ServletServerHttpRequest(servletRequest);
}
```

> `AbstractMessageConverterMethodProcessor`类

```java
protected ServletServerHttpResponse createOutputMessage(NativeWebRequest webRequest) {
   HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
   Assert.state(response != null, "No HttpServletResponse");
   return new ServletServerHttpResponse(response);
}
```

### 真正去解析应用我们的返回值的部分

> `AbstractMessageConverterMethodProcessor`类

```java
//Value是我们的返回值
//returnType是包含了我们的返回值对应的控制器类方法的详细信息的封装类
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
      ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   Object body;
   Class<?> valueType;
   Type targetType;
	//看看我们的返回值是不是CharSequence类或其子类.并根据是与不是两种可能做两种不同的处理,我们假设我们的返回值不满足条件
   if (value instanceof CharSequence) {
      body = value.toString();
      valueType = String.class;
      targetType = String.class;
   }
    //进入这个代码块
   else {
      body = value;
       //获取到我们的返回值的实际类型
      valueType = getReturnValueType(body, returnType);
       //获取到我们的???
      targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
   }
    //isResourceType
	//return returnType != InputStreamResource.class && Resource.class.isAssignableFrom(returnValueType);
    //这里被跳过了
   if (isResourceType(value, returnType)) {
   }
	
   MediaType selectedMediaType = null;
    //获取我们的响应头中的ContentType响应头
   MediaType contentType = outputMessage.getHeaders().getContentType();
    //看看有没有响应头
   boolean isContentTypePreset = contentType != null && contentType.isConcrete();
    //没有
   if (isContentTypePreset) {
      if (logger.isDebugEnabled()) {
         logger.debug("Found 'Content-Type:" + contentType + "' in response");
      }
      selectedMediaType = contentType;
   }
    
    //内容协商开始
    //进入这个代码块
   else {
       //获取到我们用户的请求Request实例
      HttpServletRequest request = inputMessage.getServletRequest();
      List<MediaType> acceptableTypes;
      try {
          //获取我们的用户端在请求头中声明的可以接收的数据类型
         acceptableTypes = getAcceptableMediaTypes(request);
      }
       //获取当前我们的程序能够将我们的返回值类型转换成哪些数据类型
      List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
		//返回值不为空且producibleTypes为空时进行
      if (body != null && producibleTypes.isEmpty()) {
         throw new HttpMessageNotWritableException(
               "No converter found for return value of type: " + valueType);
      }
       
       
      List<MediaType> mediaTypesToUse = new ArrayList<>();
       //遍历我们的用户端可接受数据类型与我们的服务端可转换数据类型
       //看看是否存在这样的两个类型它们可以互相兼容
       //如果可以互相兼容就将实际的数据类型保存到mediaTypesToUse集合中
      for (MediaType requestedType : acceptableTypes) {
         for (MediaType producibleType : producibleTypes) {
             //看看当前服务端可转换类型与当前客户端可接收数据类型是否兼容
            if (requestedType.isCompatibleWith(producibleType)) {
                //如果兼容就将该兼容的数据类型添加到mediaTypesToUse中
               mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
            }
         }
      }
   //内容协商完成
       
       //看看我们的mediaTypesToUse是否为空,为空则表明我们的服务端无法生成响应数据满足我们的客户端的要求
       //如果不为空则证明可以进行转换
      if (mediaTypesToUse.isEmpty()) {
         if (logger.isDebugEnabled()) {
            logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
         }
         if (body != null) {
            throw new HttpMediaTypeNotAcceptableException(producibleTypes);
         }
         return;
      }
		//对mediaTypesToUse进行排序
      MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
		//选取我们将要使用的代转换数据类型
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
	//代转换数据类型选好了,准备进行转换
   if (selectedMediaType != null) {
      selectedMediaType = selectedMediaType.removeQualityValue();
       //这里的this是我们的RequestResponseBodyMethodProcessor返回值解析器实例
       //messageConverters是其一个存储着我们的所有实现了HttpMessageConverter接口的类的类实例的集合的类属性
      for (HttpMessageConverter<?> converter : this.messageConverters) {
          //获取到我们的集合中的第i个HttpMessageConverter
          //若其是GenericHttpMessageConverter类或其子类,则强转其为GenericHttpMessageConverter,若不是则返回null
         GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
               (GenericHttpMessageConverter<?>) converter : null);
          //看看我们的converter能不能处理这种数据类型转换
         if (genericConverter != null ?
               ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
               converter.canWrite(valueType, selectedMediaType)) {
             //做转换前的预处理工作
            body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
                  (Class<? extends HttpMessageConverter<?>>) converter.getClass(),
                  inputMessage, outputMessage);
            if (body != null) {
               Object theBody = body;
               LogFormatUtils.traceDebug(logger, traceOn ->
                     "Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
               addContentDispositionHeader(inputMessage, outputMessage);
               if (genericConverter != null) {
                   //开始转换,并将转换结果写入我们的Response响应的响应体
                  genericConverter.write(body, targetType, selectedMediaType, outputMessage);
               }
               else {
                   //开始转换,并将转换结果写入我们的Response响应的响应体
                  ((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
               }
            }
             //转换好并将数据写入了我们的Response响应体,结束我们的函数
             return;
         }
      }
   }
	//到了这里说明没有转换器可以进行转换,而body又不为空说明不是因为返回值为空而无需转换,而是无法转换,因此报错.不过一般不到这一步来
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

```java
protected boolean isResourceType(@Nullable Object value, MethodParameter returnType) {
   Class<?> clazz = getReturnValueType(value, returnType);
   return clazz != InputStreamResource.class && Resource.class.isAssignableFrom(clazz);
}
```

# `HttpMessageConverter`接口介绍

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230213004432199.png" alt="image-20230213004432199" style="zoom:80%;" />

> - `canRead`:看看我们是否能将指定的`MediaType`类型数据读出来存储到我们的给定类中
> - `canWrite`:看看我们是否能将传入的类以指定的`MediaType`写出去
> - `read`:将我们的`HttpInputMessage`的请求体内容读取并存储到指定类实例中
> - `write`:将我们的传入类实例写为指定的`MediaType`格式的数据,并写入指定的`HttpOutputMessage`实例的响应体中

## 常用的`HttpMessageConverter`实现类

![image-20230213005321216](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230213005321216.png)

![image-20230213005534692](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230213005534692.png)

# `RequestResponseBodyMethodProcessor`类
