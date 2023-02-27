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
