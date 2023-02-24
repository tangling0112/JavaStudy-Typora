## 前处理部分

> 在这一部分我们的程序运行了许多步骤,只不过我们前面已经在参数解析器解释小节,也就是第`3`小节进行了呈现,因此这一部分可前往第`3`小节查看

## 真正封装数据部分

![image-20230212004837955](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230212004837955.png)<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230212043743339.png" alt="image-20230212043743339" style="zoom: 80%;" />

> `InvocableHandlerMethod`类

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    MethodParameter[] parameters = getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    }

    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
        }
        try {
            //在这一步就要去对我们的传入参数进行解析封装,这个dataBinderFactory就是ServletRequestDataBindFacotry类的类实例
            //relovers就是封装了所有26个参数解析器的HandlerMethodArgumentResolverComposite类实例
            //parammeter就是我们要接收封装结果的变量
            //dataBinderFactory就是后续要用来生成ServletRequestDataBinder来辅助我们解析参数的工厂实例
            args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
        }
    }
    return args;
}
```

> `HandlerMethodArgumentResolverComposite`类

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
	//在这一步我们去查找我们的InvocableHandlerMethod实例的resolvers属性,并调用了其对应的类的类实例的getArgumentResolver方法,用于查找解析我们参数的参数解析器,并最终决定使用ServletModelAttributeMethodProcessor参数解析器来进行处理
   HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
   if (resolver == null) {
      throw new IllegalArgumentException("Unsupported parameter type [" +
            parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
   }
    //调用ServletModelAttributeMethodProcessor参数解析器来解析并封装我们的参数
    //parameter就是存储了我们形参的一些信息的封装类实例
   return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

> `ServletModelAttributeMethodProcessor`的父类`ModelAttributeMethodProcessor`

```java
public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   Assert.state(mavContainer != null, "ModelAttributeMethodProcessor requires ModelAndViewContainer");
   Assert.state(binderFactory != null, "ModelAttributeMethodProcessor requires WebDataBinderFactory");
	//从我们存储着我们的形式参数信息的parameter参数中获取我们形参的名称
   String name = ModelFactory.getNameForParameter(parameter);
   ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
   if (ann != null) {
      mavContainer.setBinding(name, ann.binding());
   }

   Object attribute = null;
   BindingResult bindingResult = null;
	//看看我们的ModelAndViewContainer中是否已经存储了我们的person参数的值
   if (mavContainer.containsAttribute(name)) {
      attribute = mavContainer.getModel().get(name);
   }
   else {
      // Create attribute instance
      try {
          //通过反射得到我们的形式参数对应的类的构造器,然后构造我们的形式参数对应的类的类实例
          //name是我们形参的名称
          //parameter是存储着我们形参的详细信息的封装类
          //binderFactory是用于生成ServletRequestDataBinder的工厂
          //这个attribute变量最终就是我们的形式参数对应的类的类实例
         attribute = createAttribute(name, parameter, binderFactory, webRequest);
      }
   }

   if (bindingResult == null) {
      //将我们构造出来的容纳着我们的形式参数对应的类的空实例的容器,我们形式参数的名称以及我们的webRequest参数传入以构造一个ServletDatabinder实例,后面我们对attribute进行解析赋值的过程都通过这个实例进行
      WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
       //target就是存储我们的实际参数的实例的属性
      if (binder.getTarget() != null) {
         if (!mavContainer.isBindingDisabled(name)) {
             //真正的处理解析步骤
            bindRequestParameters(binder, webRequest);
         }
         validateIfApplicable(binder, parameter);
         if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
            throw new BindException(binder.getBindingResult());
         }
      }
      // Value type adaptation, also covering java.util.Optional
      if (!parameter.getParameterType().isInstance(attribute)) {
         attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
      }
      bindingResult = binder.getBindingResult();
   }

   // Add resolved attribute and BindingResult at the end of the model
   Map<String, Object> bindingResultModel = bindingResult.getModel();
   mavContainer.removeAttributes(bindingResultModel);
   mavContainer.addAllAttributes(bindingResultModel);
	//由于我们的ServletRequestDataBinder的target属性存储的正是我们的attribute,而这两个变量实际上指向同一个地址,因此我们的ServletRequestDataBinder的实例的target属性在后面过程中的修改实际上也就是对我们的attribute变量的修改
   return attribute;
}
```

### 构造类实例部分

> `ServletModelAttributeMethodProcessor`类

```java
protected final Object createAttribute(String attributeName, MethodParameter parameter,
      WebDataBinderFactory binderFactory, NativeWebRequest request) throws Exception {
	//使用ServletModelAttributeMethodProcessor的getRequestValueForAttribute方法,看看我们的用户传过来的Request请求中有没有和我们的形式参数同名的请求参数,如果没有才进入构造过程.显然我们是有的
   String value = getRequestValueForAttribute(attributeName, request);
    //被跳过
   if (value != null) {
      Object attribute = createAttributeFromRequestValue(
            value, attributeName, parameter, binderFactory, request);
      if (attribute != null) {
         return attribute;
      }
   }
	//构造我们的形式参数对应的类的类实例
   return super.createAttribute(attributeName, parameter, binderFactory, request);
}
```

```java
protected String getRequestValueForAttribute(String attributeName, NativeWebRequest request) {
    Map<String, String> variables = getUriTemplateVariables(request);
    String variableValue = variables.get(attributeName);
    if (StringUtils.hasText(variableValue)) {
        return variableValue;
    }
    String parameterValue = request.getParameter(attributeName);
    if (StringUtils.hasText(parameterValue)) {
        return parameterValue;
    }
    return null;
}
```

> `ServletModelAttributeMethodProcessor`的父类`ModelAttributeMethodProcessor`

```java
protected Object createAttribute(String attributeName, MethodParameter parameter,
      WebDataBinderFactory binderFactory, NativeWebRequest webRequest) throws Exception {

   MethodParameter nestedParameter = parameter.nestedIfOptional();
    //获取到我们要封装出来的的实际参数参数的全类名
   Class<?> clazz = nestedParameter.getNestedParameterType();
	//借助反射机制获取我们待封装的实际参数对应的类的构造器
   Constructor<?> ctor = BeanUtils.getResolvableConstructor(clazz);
    //实例化出一个我们的形式参数对应的类的类实例
   Object attribute = constructAttribute(ctor, attributeName, parameter, binderFactory, webRequest);
   if (parameter != nestedParameter) {
      attribute = Optional.of(attribute);
   }
    //将实例化好的实例返回回去
   return attribute;
}
```

### 真正解析封装数据部分

> `ServletModelAttributeMethodProcessor`类

```java
protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
   ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
   Assert.state(servletRequest != null, "No ServletRequest");
   ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
   //绑定数据
    //我们的ServletRequestDataBinder实例中此时已经存储了我们的形式参数的类的空实例
   servletBinder.bind(servletRequest);
}
```

> `ServeltRequestDataBinder`类

```java
public void bind(ServletRequest request) {
    //获取我们的Request请求中携带的所有参数
   MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
    
   MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
    //被跳过
   if (multipartRequest != null) {
      bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
   }
    //被跳过
   else if (StringUtils.startsWithIgnoreCase(request.getContentType(), MediaType.MULTIPART_FORM_DATA_VALUE)) {
      HttpServletRequest httpServletRequest = WebUtils.getNativeRequest(request, HttpServletRequest.class);
      if (httpServletRequest != null && HttpMethod.POST.matches(httpServletRequest.getMethod())) {
         StandardServletPartUtils.bindParts(httpServletRequest, mpvs, isBindEmptyMultipartFiles());
      }
   }
   addBindValues(mpvs, request);
    //真正处理的部分
    //mpvs是只携带了我们的request请求的,没有携带任何其他信息
    //但是由于调用的是我们的ServletRequestDataBinder实例继承的方法,所以实际上我们的形式参数的空实例,是可以直接通过其target属性获取到
   doBind(mpvs);
}
```

> `WebDataBinder`类

```java
protected void doBind(MutablePropertyValues mpvs) {
   checkFieldDefaults(mpvs);
   checkFieldMarkers(mpvs);
   adaptEmptyArrayIndices(mpvs);
    //处理部分
   super.doBind(mpvs);
}
```

> `DataBinder`类

```java
protected void doBind(MutablePropertyValues mpvs) {
   checkAllowedFields(mpvs);
   checkRequiredFields(mpvs);
    //处理部分
   applyPropertyValues(mpvs);
}
```

```java
protected void applyPropertyValues(MutablePropertyValues mpvs) {
   try {
      // 真正处理部分
       //通过我们的ServletRequestDataBinder实例的getPropertyAccessor方法获取了一个PropertyAccessor实例,并调用了它的setPropertyValues方法
       //mpvs参数只携带了我们的request的信息,没有携带其他信息
       //我们获取到的PropertyAccessor实际上是BeanWrapperImpl类的类实例
       //在ServletRequestDataBinder实例实例化这个BeanWrapperImpl实例的时候就已经将其target属性作为其构造器参数传入这个实例了,也就是说我们的BeanWrapperImpl实例中存储着我们的形式参数的空实例
      getPropertyAccessor().setPropertyValues(mpvs, isIgnoreUnknownFields(), isIgnoreInvalidFields());
   }
}
```

![image-20230212041430963](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230212041430963.png)

> `AbstractPropertyAccessor`类

```java
public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid) throws BeansException {
    List<PropertyAccessException> propertyAccessExceptions = null;
    List<PropertyValue> propertyValues = pvs instanceof MutablePropertyValues ? ((MutablePropertyValues)pvs).getPropertyValueList() : Arrays.asList(pvs.getPropertyValues());
    //被跳过
    if (ignoreUnknown) {
        this.suppressNotWritablePropertyException = true;
    }

    try {
        //propertyValues属性就是我们上面的mpvs变量,其内存储了我们的request请求的所以请求参数以及其值
        Iterator var6 = propertyValues.iterator();
		//遍历我们获取到的Request请求的所有请求参数的值
        while(var6.hasNext()) {
            PropertyValue pv = (PropertyValue)var6.next();

            try {
                //将请求参数值尝试赋值给我们的实际参数实例的属性
                //如果该参数在我们的形式参数对应的类中没有对应的属性这个方法内部就会报错,从而当前循环结束,进入下一次循环遍历下一个请求参数
                this.setPropertyValue(pv);
            } 
}
```

```java
public void setPropertyValue(PropertyValue pv) throws BeansException {
    //transient volatile Object resolvedTokens;我们的tokens会是一个直接从JVM获取值的变量
    PropertyTokenHolder tokens = (PropertyTokenHolder)pv.resolvedTokens;
    if (tokens == null) {
        //获取我们的请求参数的名称
        String propertyName = pv.getName();

        AbstractNestablePropertyAccessor nestedPa;
        try {
            //根据我们的请求参数的名字去获取用于访问我们的实际参数实例的属性的访问器,如果获取不到就会报错,从而使得进入对下一个请求参数的遍历
            //nestedPa其实是一个BeanWrapperImpl的实例,其封装着访问实际参数指定属性的访问器
            //BeanWrapperImpl是我们的AbstractPropertyAccessor类的子类
            //也就是说我们的newtedPa是和我们通过ServletRequestDataBinder实例的getPropertyAccessor方法获取到的实例属于同一系列
            //我们的this就是指的那个我们通过ServletRequestDataBinder实例的getPropertyAccessor方法获取到的实例
            nestedPa = this.getPropertyAccessorForPropertyPath(propertyName);
        } 
		//获取用于标记真正用于访问实际参数的指定属性的访问器的PropertyTokenHolder标记器实例
        //我们的this就是指的那个我们通过ServletRequestDataBinder实例的getPropertyAccessor方法获取到的实例
        //getFinalPath是用于获取我们最终要用于应用到处理中的参数名
        //tokens是我们的标记器
        //this是我们的BeanWrapperImpl类的类实例
        tokens = this.getPropertyNameTokens(this.getFinalPath(nestedPa, propertyName));
        if (nestedPa == this) {
            pv.getOriginalPropertyValue().resolvedTokens = tokens;
        }
		//通过标记以及请求参数值对我们的实际参数实例进行真正赋值
        //nestedPa就是我们的BeanWrapperImpl实例,其存储着我们的形式参数的空实例
        nestedPa.setPropertyValue(tokens, pv);
    } else {
        this.setPropertyValue(tokens, pv);
    }

}
```

> `AbstractNeastablePropertyAccessor`类

```java
protected String getFinalPath(AbstractNestablePropertyAccessor pa, String nestedPath) {
   if (pa == this) {
      return nestedPath;
   }
   return nestedPath.substring(PropertyAccessorUtils.getLastNestedPropertySeparatorIndex(nestedPath) + 1);
}
```

```java
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
   if (tokens.keys != null) {
      processKeyedProperty(tokens, pv);
   }
   else {
      //进入这个方法
       //这个方法是BeanWrapperImpl类的实例的方法
      processLocalProperty(tokens, pv);
   }
}
```

```java
//pv就是我们前面的mpvs变量,其内部存储着request请求的所有请求参数以及其值
private void processLocalProperty(PropertyTokenHolder tokens, PropertyValue pv) {
    //通过调用我们的BeanWrapperImpl实例的getLocalPropertyHandler方法凭借我们的标记器实例获取到我们的实际参数实例的属性的PropertyHandler访问器实例
    //实际上呢就是通过请求参数的名字去获取BeanWrapperImpl存储的形式参数的空实例的指定属性的地址的封装类
   PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
    //这里有对于无法获取对于属性的解决方案,这里我们删去了,只不过我们要知道这个部分通过抛出异常来配合我们上面的方法进入下一个请求参数的处理过程
   Object oldValue = null;
   try {
       //获取到我们要用于赋值的请求参数值
      Object originalValue = pv.getValue();
      Object valueToApply = originalValue;
      if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
         if (pv.isConverted()) {
            valueToApply = pv.getConvertedValue();
         }
         else {
            if (isExtractOldValueForEditor() && ph.isReadable()) {
               try {
                   //访问我们的形式参数的空实例的对应属性,这个是在后面用来让我们的程序知道我们的形式参数的空实例的那个属性的数据类型的变量
                  oldValue = ph.getValue();
               }
            }
             //这个valueToApply最终会接收到请求参数数据类型转换为形式参数的指定属性的数据类型后的对象
             //ph.toTypeDescriptor()是用于获取我们数据类型转换的目标数据类型的封装类
             //originalValue是我们的请求参数的值
             //tokens.canonicalName是我们的请求参数的名称
            valueToApply = convertForProperty(
                  tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());
         }
         pv.getOriginalPropertyValue().conversionNecessary = (valueToApply != originalValue);
      }
       //将我们转换好后的值赋值给我们的形式参数的空实例的指定属性,在这一步由于我们的ph中存储着形式参数的指定属性的内存地址,因此这样的修改会使得指向同一个地址的变量的值均发生修改,因此我们前面的attribute变量中的指定属性的值就会同步修改
      ph.setValue(valueToApply);
   }
}
```

```java
protected Object convertForProperty(
      String propertyName, @Nullable Object oldValue, @Nullable Object newValue, TypeDescriptor td)
      throws TypeMismatchException {
	//进入这个方法
    //propertyName为请求参数名称
    //newValue为我们请求参数的值
    //td.getType()获取到我们数据类型转换的目标数据类型
   return convertIfNecessary(propertyName, oldValue, newValue, td.getType(), td);
}
```

```java
private Object convertIfNecessary(@Nullable String propertyName, @Nullable Object oldValue,
      @Nullable Object newValue, @Nullable Class<?> requiredType, @Nullable TypeDescriptor td)
      throws TypeMismatchException {

   try {
       //进入这个方法
       //这里的this就是我们的BeanWrapperImpl类的类实例
       //这里的typeConverterDelegate就是我们的BeanWrapperImpl类的类实例中存储的TypeConverterDelegate类实例
      return this.typeConverterDelegate.convertIfNecessary(propertyName, oldValue, newValue, requiredType, td);
   }
}
```

![image-20230212043238114](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230212043238114.png)

> `TypeConverterDelegate`类

```java
public <T> T convertIfNecessary(@Nullable String propertyName, @Nullable Object oldValue, @Nullable Object newValue,
      @Nullable Class<T> requiredType, @Nullable TypeDescriptor typeDescriptor) throws IllegalArgumentException {

   //
   PropertyEditor editor = this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);

   ConversionFailedException conversionAttemptEx = null;

   // 获取ConverSionService实例
    //这里的this是我们BeanWrapperImpl的typeConverterDelegate属性,是一个TypeConverterDelegate实例
    //propertyEditorRegistry是我们的BeanWrapperImpl在实例化TypeConverterDelegate实例时给TypeConverterDelegate的有参构造器传入的
    //propertyEditorRegistry实际上就是我们的BeanWrapperImpl实例,而getConversionService方法是我们的BeanWrapperImpl实例从PropertyEditorRegistrySupport类继承的
    //conversionService实际上就是我们的BeanWrapperImpl实例从PropertyEditorRegistrySupport类继承的一个属性,这个属性是GenericConversionService类或其子类的实例
   ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
   if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
      TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
       //看看我们的这个ConverSionService实例中有没有转换器能转换sourceTypeDesc所述类型到typeDescriptor所述类型,如果可以就转换
      if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
         try {
             //进行转换
             //newValue就是我们前面的请求参数的值
            return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
         }
}
```

> `GenericConversionService`类

```java
public Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType) {
   Assert.notNull(targetType, "Target type to convert to cannot be null");
   if (sourceType == null) {
      Assert.isTrue(source == null, "Source must be [null] if source type == [null]");
      return handleResult(null, targetType, convertNullSource(null, targetType));
   }
   if (source != null && !sourceType.getObjectType().isInstance(source)) {
      throw new IllegalArgumentException("Source to convert from must be an instance of [" +
            sourceType + "]; instead it was a [" + source.getClass().getName() + "]");
   }
    //获取转换器实例
    //这里调用的是GenericConversionService实例的getConverter方法
    //sourceType就是我们的请求参数的数据类型,targetType就是我们的形式参数的空实例的指定属性的数据类型
   GenericConverter converter = getConverter(sourceType, targetType);
   if (converter != null) {
       //使用转换器进行转换,并获取转换结果
       //source就是我们的newValue也就是我们的请求参数的值
       //result会获取到转换后的结果
      Object result = ConversionUtils.invokeConverter(converter, source, sourceType, targetType);
       //如果result为空证明转换失败,会使用其他方式进行转换,如果不为空证明转换成功,会返回转换后的结果
      return handleResult(sourceType, targetType, result);
   }
   return handleConverterNotFound(source, sourceType, targetType);
}
```

```java
protected GenericConverter getConverter(TypeDescriptor sourceType, TypeDescriptor targetType) {
   ConverterCacheKey key = new ConverterCacheKey(sourceType, targetType);
   GenericConverter converter = this.converterCache.get(key);
   if (converter != null) {
      return (converter != NO_MATCH ? converter : null);
   }

   converter = this.converters.find(sourceType, targetType);
   if (converter == null) {
      converter = getDefaultConverter(sourceType, targetType);
   }

   if (converter != null) {
      this.converterCache.put(key, converter);
      return converter;
   }

   this.converterCache.put(key, NO_MATCH);
   return null;
}
```

```java
private Object handleResult(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType, @Nullable Object result) {
   if (result == null) {
      assertNotPrimitiveTargetType(sourceType, targetType);
   }
   return result;
}
```

## `ServletRequestDataBinder`

```java
//this为invocableHandlerMethod实例
this.dataBinderFactory
WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name)
```

```java
ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
servletBinder.bind(servletRequest);
//ServeltRequestDataBinder
doBind(mpvs);
//WebDataBinder
//super为Databinder
super.doBind(mpvs);
//DataBinder
applyPropertyValues(mpvs);
//this为我们的ServeltRequestDataBinder实例
this.getpropertyAccessor()
```

![image-20230212050307895](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230212050307895.png)

## `BeanWrapperImpl`

```java
//this是ServletRequestDataBinder实例
//这个方法是ServletRequestDataBinder实例从DataBinder类继承的
this.getpropertyAccessor();
//this是BeanWrapperImpl实例
this.setPropertyValues(mpvs, isIgnoreUnknownFields(), isIgnoreInvalidFields());

this.setPropertyValue(pv);//这里是给我们的形式参数的实例的指定属性赋值的

nestedPa = this.getPropertyAccessorForPropertyPath(propertyName);
tokens = this.getPropertyNameTokens(this.getFinalPath(nestedPa, propertyName));
//nestedPa就是我们的BeanWrapperImpl实例
nestedPa.setPropertyValue(tokens, pv);
this.processLocalProperty(tokens, pv);
this.getLocalPropertyHandler(tokens.actualName);
this.convertForProperty(
                  tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());
         };
this.convertIfNecessary(propertyName, oldValue, newValue, td.getType(), td);
this.typeConverterDelegate.convertIfNecessary(propertyName, oldValue, newValue, requiredType, td);
//这里的propertyEditorRegistry就是我们的BeanWrapperImpl实例getConversionService方法是其从DataBinder类j
this.propertyEditorRegistry.getConversionService()
```



## `ConversionService`转换容器与`Converts`转换器
