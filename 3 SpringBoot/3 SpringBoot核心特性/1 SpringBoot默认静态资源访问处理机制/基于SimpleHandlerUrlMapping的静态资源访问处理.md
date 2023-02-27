# 1 自定义拦截器实现类

# 2 将拦截器注册到我们的`SpringBoot`

## 方式一

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
 @Override
	public void addInterceptors(InterceptorRegistry registry) {
		HandlerInterceptor myInterceptor = new MyInterceptor();
		registry.addInterceptor(myInterceptor).addPathPatterns("/**").excludePathPatterns
("css/**");
 	}
    
 	private class MyInterceptor implements HandlerInterceptor{
  		@Override
  		public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
   		return false;
  		}
 	}
}
```

## 方式二

```java
@Configuration
public class WebConfig {
 	@Bean
 	public WebMvcConfigurer webMvcConfigurer() {
  		return new WebMvcConfigurer() {
   			@Override
   			public void addInterceptors(InterceptorRegistry registry) {
    			HandlerInterceptor myInterceptor = new MyInterceptor();
    			registry.addInterceptor(myInterceptor).addPathPatterns("/**").exclu
dePathPatterns("css/**");
   			}
  		};
 	}
    
 	private class MyInterceptor implements HandlerInterceptor{
  		@Override
  		public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
   			return false;
  		}
 	}
}
```
