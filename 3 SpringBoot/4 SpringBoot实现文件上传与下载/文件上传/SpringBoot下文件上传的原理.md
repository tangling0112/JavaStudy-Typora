# `SpringBoot`的文件上传的自动配置

## `MultiPartAutoConfiguration`自动配置类

```java
@AutoConfiguration
@ConditionalOnClass({ Servlet.class, StandardServletMultipartResolver.class, MultipartConfigElement.class })
@ConditionalOnProperty(prefix = "spring.servlet.multipart", name = "enabled", matchIfMissing = true)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(MultipartProperties.class)
public class MultipartAutoConfiguration {

	private final MultipartProperties multipartProperties;

	public MultipartAutoConfiguration(MultipartProperties multipartProperties) {
		this.multipartProperties = multipartProperties;
	}

	@Bean
	@ConditionalOnMissingBean({ MultipartConfigElement.class, CommonsMultipartResolver.class })
	public MultipartConfigElement multipartConfigElement() {
		return this.multipartProperties.createMultipartConfig();
	}

	@Bean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
	@ConditionalOnMissingBean(MultipartResolver.class)
	public StandardServletMultipartResolver multipartResolver() {
		StandardServletMultipartResolver multipartResolver = new StandardServletMultipartResolver();
		multipartResolver.setResolveLazily(this.multipartProperties.isResolveLazily());
		return multipartResolver;
	}

}
```

### `MultipartProperties`配置类(`spring.servlet.multipart`前缀)

```java
@ConfigurationProperties(prefix = "spring.servlet.multipart", ignoreUnknownFields = false)
public class MultipartProperties {
	//是否开启Multipart文件上传功能
	private boolean enabled = true;

	//默认的上传文件的存储位置
	private String location;
	
    //最大可上传的单个文件的大小,如10MB
    	//这里的默认值为1MB
	private DataSize maxFileSize = DataSize.ofMegabytes(1);

    //最大可上传的所有文件的大小的总和,如100MB
    	//这里的默认值为10MB
	private DataSize maxRequestSize = DataSize.ofMegabytes(10);

	//Threshold after which files are written to disk.
	private DataSize fileSizeThreshold = DataSize.ofBytes(0);

	//Whether to resolve the multipart request lazily at the time of file or parameter
	//* access.
	private boolean resolveLazily = false;

	/**
	 * Create a new {@link MultipartConfigElement} using the properties.
	 * @return a new {@link MultipartConfigElement} configured using there properties
	 */
	public MultipartConfigElement createMultipartConfig() {
		MultipartConfigFactory factory = new MultipartConfigFactory();
		PropertyMapper map = PropertyMapper.get().alwaysApplyingWhenNonNull();
		map.from(this.fileSizeThreshold).to(factory::setFileSizeThreshold);
		map.from(this.location).whenHasText().to(factory::setLocation);
		map.from(this.maxRequestSize).to(factory::setMaxRequestSize);
		map.from(this.maxFileSize).to(factory::setMaxFileSize);
		return factory.createMultipartConfig();
	}

}
```

# `SpringBoot`下`DispatcherServelt`处理文件上传的流程

## `DispatcherServlet`类

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
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
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
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
    if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
        if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
        }
        else if (hasMultipartException(request)) {
        }
        else {
            try {
                return this.multipartResolver.resolveMultipart(request);
            }
        }
    }
    // If not returned before: return original request.
    return request;
}
```

### `StandardMultipartHttpServletRequest`类

```java
public MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException {
    return new StandardMultipartHttpServletRequest(request, this.resolveLazily);
}
```

```java
public StandardMultipartHttpServletRequest(HttpServletRequest request, boolean lazyParsing)
        throws MultipartException {

    super(request);
    if (!lazyParsing) {
        parseRequest(request);
    }
}
```

```java
private void parseRequest(HttpServletRequest request) {
    try {
        Collection<Part> parts = request.getParts();
        this.multipartParameterNames = new LinkedHashSet<>(parts.size());
        MultiValueMap<String, MultipartFile> files = new LinkedMultiValueMap<>(parts.size());
        for (Part part : parts) {
            String headerValue = part.getHeader(HttpHeaders.CONTENT_DISPOSITION);
            ContentDisposition disposition = ContentDisposition.parse(headerValue);
            String filename = disposition.getFilename();
            if (filename != null) {
                if (filename.startsWith("=?") && filename.endsWith("?=")) {
                    filename = MimeDelegate.decode(filename);
                }
                files.add(part.getName(), new StandardMultipartFile(part, filename));
            }
            else {
                this.multipartParameterNames.add(part.getName());
            }
        }
        setMultipartFiles(files);
    }
}
```

#### `ServletRequestWrapper`类

```java
public ServletRequestWrapper(ServletRequest request) {
    if (request == null) {
        throw new IllegalArgumentException(lStrings.getString("wrapper.nullRequest"));
    } else {
        this.request = request;
    }
}
```

## `RequestPartMethodArgumentResolver`参数解析器

> d

```java
public class RequestPartMethodArgumentResolver extends AbstractMessageConverterMethodArgumentResolver {
	@Override
	@Nullable
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest request, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
		Assert.state(servletRequest != null, "No HttpServletRequest");

		RequestPart requestPart = parameter.getParameterAnnotation(RequestPart.class);
		boolean isRequired = ((requestPart == null || requestPart.required()) && !parameter.isOptional());

		String name = getPartName(parameter, requestPart);
		parameter = parameter.nestedIfOptional();
		Object arg = null;

		Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
		if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
			arg = mpArg;
		}
		else {
			try {
				HttpInputMessage inputMessage = new RequestPartServletServerHttpRequest(servletRequest, name);
				arg = readWithMessageConverters(inputMessage, parameter, parameter.getNestedGenericParameterType());
				if (binderFactory != null) {
					WebDataBinder binder = binderFactory.createBinder(request, arg, name);
					if (arg != null) {
						validateIfApplicable(binder, parameter);
						if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
							throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
						}
					}
					if (mavContainer != null) {
						mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
					}
				}
			}
			catch (MissingServletRequestPartException | MultipartException ex) {
				if (isRequired) {
					throw ex;
				}
			}
		}

		if (arg == null && isRequired) {
			if (!MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
				throw new MultipartException("Current request is not a multipart request");
			}
			else {
				throw new MissingServletRequestPartException(name);
			}
		}
		return adaptArgumentIfNecessary(arg, parameter);
	}

	private String getPartName(MethodParameter methodParam, @Nullable RequestPart requestPart) {
		String partName = (requestPart != null ? requestPart.name() : "");
		if (partName.isEmpty()) {
			partName = methodParam.getParameterName();
			if (partName == null) {
				throw new IllegalArgumentException("Request part name for argument type [" +
						methodParam.getNestedParameterType().getName() +
						"] not specified, and parameter name information not found in class file either.");
			}
		}
		return partName;
	}
}
