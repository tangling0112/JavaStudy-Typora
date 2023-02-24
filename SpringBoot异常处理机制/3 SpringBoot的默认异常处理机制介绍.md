# `SpringBoot`的默认异常处理机制

> ​		我们前面提到当我们的`DispatcherServlet`的`doDispatch`方法在运行遇到异常时会触发我们的`SpringMVC`的异常处理机制,并且我们有四大默认异常处理器来处理我们的异常
>
> ​		但是,很显然的是有些时候我们遇到的异常会无法被这四大异常处理器中的任何一个处理(`DefualtErrorAttribute`可以处理,但是它只会返回`null`),也就是说四大异常处理器都会返回`null`,这就会导致我们的`doDispatch`方法调用的`processDispatchResult`方法所调用的`processException`方法会将我们的异常抛出,而我们的`processDispatchResult`方法并不会进行异常处理,而只会将我们的异常向我们的`doDispatch`方法抛出,而这就导致了我们的`processDispatchResult`方法中调用的视图渲染方法`render`方法不会被调用.也就导致我们无法渲染出交付给我们的用户的视图.

> ​		`SpringBoot`对于这个情况做出了很好的处理,当遇到这种情况时,我们的`SpringBoot`会将我们的`Request`请求进行自动地请求转发,这个请求转发的默认路径为`/error`(当然我们可以借助`SpringBoot`的配置文件来修改它).而我们的`SpringBoot`的`ErrorMvcAutoConfiguration`自动配置类正好注册了一个`BasicErrorController`控制类,这个控制类就可以对我们的`/error`映射的请求进行处理.
>
> ​		我们的`BasicErrorController`又调用了我们的`DefualtErrorViewResolver`的方法来处理我们的用户请求.
>
> ​		我们的`DefualtErrorViewResolver`会先去寻找我们的`SpringBoot`项目的类路径下的静态资源存储路径以及`templates`目录,看看是否具备`/error`目录,`/error`目录下是否有与我们的错误状态码匹配的静态错误页面,如果有就返回一个携带有这个页面的`ModelAndView`实例交给我们的视图解析器去解析,如果这一步没有找到合适的错误页面,就会返回一个`viewName`属性为`error`的`ModelAndView`实例,这个实例在进入视图解析的时候就会被我们的`BeanNameViewResolver`解析,而我们的这个视图解析器就会去我们的`BeanFactory`中找到我们的`ID=error`的实现了`View`接口的`Bean`对象来作为我们的视图解析结果.
>
> ​		而我们的`ErrorMvcAutoConfiguration`刚好就注册了一个`ID=error`的`StaticView`实例的`Bean`对象,也就是我们的`WhitelabelErrorView`

## 源码分析

### `BasicErrorController`

```java
@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
   HttpStatus status = getStatus(request);
   Map<String, Object> model = Collections
         .unmodifiableMap(getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML)));
   response.setStatus(status.value());
    //这里就会调用我们的BasicErrorController实例从其父类的父类AbstractErrorController类继承的resolveErrorView来进行我们的错误处理
   ModelAndView modelAndView = resolveErrorView(request, response, status, model);
    //这里我们发现当我们的后续处理找到了合适的静态文件的时候会返回一个包含了HtmlResourceView实例的ModelAndView对象,而如果没有找到就会返回一个ModelAndView("error", model)实例.
    //而后者会被我们的BeanNameViewResolver所解析.这个也就是我们的WhitelabelErrorView实现的原理
   return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
}

@RequestMapping
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
   HttpStatus status = getStatus(request);
   if (status == HttpStatus.NO_CONTENT) {
      return new ResponseEntity<>(status);
   }
   Map<String, Object> body = getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.ALL));
   return new ResponseEntity<>(body, status);
}
```

### `AbstractErrorController`

```java
protected ModelAndView resolveErrorView(HttpServletRequest request, HttpServletResponse response, HttpStatus status,
      Map<String, Object> model) {
    //这里的this就是指我们的BasicErrorController实例,而errorViewResolvers就是这个实例在被我们的ErrorMvcAutoConfiguration注册为Bean对象时初始化设置的存储错误视图解析器的集合属性
   for (ErrorViewResolver resolver : this.errorViewResolvers) {
       //调用我们的第一个错误视图解析器默认情况下就是我们的DefualtErrorViewResolver视图解析器的resolveErrorView来处理我们的错误
      ModelAndView modelAndView = resolver.resolveErrorView(request, status, model);
      if (modelAndView != null) {
         return modelAndView;
      }
   }
   return null;
}
```

### `DefaultErrorViewResolver`

```java
public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
    //这里调用的是我们的ErrorMvcAutoConfiguration注册的DefaultErrorViewResolver这个Bean对象的resolve方法,这个方法会按照我们的Request请求的错误状态码去寻找合适的静态页面,如404.html等等
   ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
   if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
       //这里又再次调用了我们的resolve方法,这里二次调用的目的在于当我们无法精确地找到404.html这样的文件时,我们就会去寻找4xx.html这个文件
      modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
   }
   return modelAndView;
}
```

```java
private ModelAndView resolve(String viewName, Map<String, Object> model) {
    //将我们的查找路径拼接为/error404或/error/4xx
   String errorViewName = "error/" + viewName;
   TemplateAvailabilityProvider provider = this.templateAvailabilityProviders.getProvider(errorViewName,
         this.applicationContext);
   if (provider != null) {
      return new ModelAndView(errorViewName, model);
   }
    //调用我们的DefaultErrorViewResolver实例的resolveResource去真正地获取我们的资源
   return resolveResource(errorViewName, model);
}
```

```java
private ModelAndView resolveResource(String viewName, Map<String, Object> model) {
    //这里的getStaticLocations获取到的就是我们将会要去查找的路径
   for (String location : this.resources.getStaticLocations()) {
      try {
          //这里就会获取到我们的当前目录实例,去查看该静态资源存储目录下是否有我们的/error/404.html文件
         Resource resource = this.applicationContext.getResource(location);
         resource = resource.createRelative(viewName + ".html");
         if (resource.exists()) {
             //如果找到了我们想要的资源就将其做出一些处理后封装为ModelAndView实例返回,然后经由我们的视图解析器处理后呈现给我们的用户
            return new ModelAndView(new HtmlResourceView(resource), model);
         }
      }
      catch (Exception ex) {
      }
   }
   return null;
}
```

```java
private static class HtmlResourceView implements View {

   private Resource resource;

   HtmlResourceView(Resource resource) {
      this.resource = resource;
   }

   @Override
   public String getContentType() {
      return MediaType.TEXT_HTML_VALUE;
   }

   @Override
   public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
         throws Exception {
      response.setContentType(getContentType());
      FileCopyUtils.copy(this.resource.getInputStream(), response.getOutputStream());
   }

}
```
