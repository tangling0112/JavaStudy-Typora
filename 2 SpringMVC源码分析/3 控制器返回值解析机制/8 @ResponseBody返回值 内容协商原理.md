# 什么是内容协商?

> ​		内容协商即用户端与服务器端就响应体中的内容的传输数据类型进行协商指定.使得用户端能接收到其能够处理的数据类型

> ​		我们知道,在我们的`Request`请求头中有着`Accept`这个请求头,而这个请求头就是用于指定我们的用户端能够接收的数据类型.我们的服务端就是借助这个请求头来获知用户端可以接收的数据类型,然后根据服务端自己的处理能力,尽最大可能地满足我们的用户端对于数据类型的要求.而这就是借助我们的内容协商完成的
>

![image-20230214024848794](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230214024848794.png)

# 内容协商的原理

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
 /****************************************************************************************/
/**********************************内容协商部分1开始****************************************/
/****************************************************************************************/  
    //这个部分的主要工作如下
    	//1.查看我们用户发起的Request请求的请求头,获取到用户端浏览器支持接收的数据类型
    	//2.查看对于我们的当前返回值,我们现有的HttpMessageConverter转换器能够将其转换成哪些数据类型
    	//3.通过循环遍历,找到我们服务端能转换成的数据类型中,客户端浏览器支持的数据类型.
    
    //进入这个代码块
   else {
       //获取到我们用户的请求Request实例
      HttpServletRequest request = inputMessage.getServletRequest();
      List<MediaType> acceptableTypes;
      try {
          //获取我们的用户端在请求头中声明的客户端可以接收的数据类型
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
/****************************************************************************************/
/**********************************内容协商部分1完成****************************************/
/****************************************************************************************/
       
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
/****************************************************************************************/ 
/********************************内容协商部分2开始*************************************/
/****************************************************************************************/
       //这个部分我们的内容协商主要做了如下工作
       		//1.根据我们的mediaTypesToUse中的各个数据类型对于用户端浏览器而言的可接收权重,对mediaTypesToUse集合进行排序,排序后,第一个元素为最优先选择的数据类型
       
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
/****************************************************************************************/
/********************************内容协商部分2结束*************************************/
/****************************************************************************************/
       
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

