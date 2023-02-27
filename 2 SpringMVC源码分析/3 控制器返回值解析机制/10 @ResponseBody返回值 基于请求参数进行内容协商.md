> ​		前面的内容协商的原理中我们提到了,我们的程序会去读取我们的用户端发送过来的`Request`请求的`Accept`请求头去获取我们的用户端能够处理的数据类型.
>
> ​		但是实际上,上面的这种处理方式只不过是我们的`SpringBoot`的默认内容协商方式.实际上我们还可以借助我们的请求参数来进行内容协商,甚至于我们可以通过自定义我们自己的`contentNegotiationManager`内容协商管理器来定义一些我们自己的内容协商规则.

## 基于`Accept`请求头的内容协商策略与基于请求参数的内容协商策略并存时的内容协商的工作效果

![image-20230214032507665](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230214032507665.png)

> ​		首先我们应该要知道,我们的内容协商管理器在使用`ContentNegotiationStrategy`获取我们的用户端能够接收的数据格式时,采用的是我们前面参数解析,返回值解析一样的原理,就是直接遍历我们的当前`ContentNegotiationManager`的`strategies`属性中存储的所有`ContentNegotiationStrategy`接口的实现类实例.
>
> ​		当我们的基于请求`URL`参数的内容协商策略被开启后,其在`strategies`属性中的存储是在我们的基于请求头的内容协商策略之前的.也就是说,会先看看基于请求`URL`参数的内容协商策略能不能处理,如果能处理就会用它处理,而我们的基于请求头的内容协商策略就不会生效.如果不能处理,就会通过遍历来让我们的基于请求头的内容协商策略来进行处理.
>
> ​		因此我们在自定义我们自己的内容协商策略的时候也要注意我们使用`Strategies`方法将它们添加到我们的`ContentNegotiationManager`中时在集合中的存储顺序

## `contentNegotiationManager`内容协商管理器的引入

### `AbstractMessageConverterMethodProcessor`类

```java
/****************************************************************************************/
/**********************************内容协商部分1开始****************************************/
/****************************************************************************************/ 
   else {
      HttpServletRequest request = inputMessage.getServletRequest();
      List<MediaType> acceptableTypes;
      try {
          //获取我们的用户端在请求头中声明的客户端可以接收的数据类型
          //就是在这一步中我们使用到了我们的contentNegotiationManager内容协商管理器
         acceptableTypes = getAcceptableMediaTypes(request);
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
/****************************************************************************************/
/**********************************内容协商部分1完成****************************************/
/****************************************************************************************/
```

> 就是在这一步使用到了我们的`@RequestResponseBodyMethodProcessor`实例的`contentNegotiationManager`属性,也就是其存储的内容协商管理器

```java
private List<MediaType> getAcceptableMediaTypes(HttpServletRequest request)
      throws HttpMediaTypeNotAcceptableException {

   return this.contentNegotiationManager.resolveMediaTypes(new ServletWebRequest(request));
}
```

```java
private final ContentNegotiationManager contentNegotiationManager
```

### `ContentNegotiationManager`类

```java
public List<MediaType> resolveMediaTypes(NativeWebRequest request) throws HttpMediaTypeNotAcceptableException {
   for (ContentNegotiationStrategy strategy : this.strategies) {
      List<MediaType> mediaTypes = strategy.resolveMediaTypes(request);
      if (mediaTypes.equals(MEDIA_TYPE_ALL_LIST)) {
         continue;
      }
      return mediaTypes;
   }
   return MEDIA_TYPE_ALL_LIST;
}
```

```java
private final List<ContentNegotiationStrategy> strategies = new ArrayList<>();
```

### `EnableWebMvcConfiguration`类

```java
@Bean
@Override
@SuppressWarnings("deprecation")
public ContentNegotiationManager mvcContentNegotiationManager() {
    //super就是我们的WebMvcAutoConfigurationSupport
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

### `WebMvcAutoConfigurationSupport`类

```java
@Bean
public ContentNegotiationManager mvcContentNegotiationManager() {
    if (this.contentNegotiationManager == null) {
        ContentNegotiationConfigurer configurer = new ContentNegotiationConfigurer(this.servletContext);
        configurer.mediaTypes(getDefaultMediaTypes());
        configureContentNegotiation(configurer);
        this.contentNegotiationManager = configurer.buildContentNegotiationManager();
    }
    return this.contentNegotiationManager;
}
```

## 如何开启我们基于请求参数的内容协商策略

> - 首先我们需要在我们的`SpringBoot`的`properties`或`yaml`配置文件中开启我们基于请求参数的内容协商策略
>     - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230214032458065.png" alt="image-20230214032458065" style="zoom:150%;" />
> - 然后我们只需要在发送请求时给出一个`format`请求参数,并用它来呈现我们想要的数据类型,如`format=json,xml`我们的服务器端就能够实现基于我们的请求参数进行内容协商
>     - ![image-20230214032507665](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230214032507665.png)
>
> **注意**:开启基于请求参数的内容协商策略并不会关闭我们的基于请求头的内容协商策略,两者会共同来做决策
>
> - 

## 为什么通过配置文件就可以开启基于请求参数的内容协商策略

### `WebMvcAutoConfigurationAdapter`类

```java
@Override
public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
    //这里的mvcProperties就是指的我们的配置文件的实体类,这里我们获取到了Contentnegotiation内容协商相关的配置
   WebMvcProperties.Contentnegotiation contentnegotiation = this.mvcProperties.getContentnegotiation();
    //根据我们的配置文件去配置我们的ContentnegotiationManager内容协商管理器
    //也就是这一步使得我们能够实现基于配置文件开启基于请求参数的内容协商策略
   configurer.favorPathExtension(contentnegotiation.isFavorPathExtension());
   configurer.favorParameter(contentnegotiation.isFavorParameter());
   if (contentnegotiation.getParameterName() != null) {
      configurer.parameterName(contentnegotiation.getParameterName());
   }
   Map<String, MediaType> mediaTypes = this.mvcProperties.getContentnegotiation().getMediaTypes();
   mediaTypes.forEach(configurer::mediaType);
}
```
