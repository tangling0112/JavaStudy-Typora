## `HttpMessageConverter`接口介绍

![image-20230214040447777](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230214040447777.png)

> - `canRead`:看看我们是否能将指定的`MediaType`类型数据读出来存储到我们的给定类中
> - `canWrite`:看看我们是否能将传入的类以指定的`MediaType`写出去
> - `read`:将我们的`HttpInputMessage`的请求体内容读取并存储到指定类实例中
> - `write`:将我们的传入类实例写为指定的`MediaType`格式的数据,并写入指定的`HttpOutputMessage`实例的响应体中

## 为什么需要自定义`HttpMessageConverter`

> ​		**假设市面上没有为`YAML`数据存储格式提供转换支持的第三方包**		
>
> ​		目前我们知道无论是`XML`还是`JSON`等等格式,我们都可以使用第三方包来为我们的`SpringBoot`提供相关的`HttpMessageConverter`支持.
>
> ​		但是突然有一天,我们的上级希望我们此后我们的服务端支持转换并发送`YAML`格式的响应体内容.此时我们没有第三方包可以使用.为了实现这一目的我们就必须去自定义属于我们自己的`HttpmessageConverter`.

## 如何自定义`HttpMessageConverter`

### `SpringBoot`下我们的`HttpMessageConverter`注册的原理

### 自定义`HttpMessageConverter`

#### 方式一

```java
@Configuration
public class WebConfig implements WebMvcConfigurer{
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        HttpMessageConverter<Person> converter = new MyFirstConverter();
        converters.add(converter);
    }
    public static class MyFirstConverter implements HttpMessageConverter<Person>{

        @Override
        public boolean canRead(Class<?> clazz, MediaType mediaType) {
            return false;
        }

        @Override
        public boolean canWrite(Class<?> clazz, MediaType mediaType) {
            clazz.isAssignableFrom(Person.class);
        }

        @Override
        public List<MediaType> getSupportedMediaTypes() {
            return MediaType.parseMediaTypes("application/tangling");
        }

        @Override
        public Person read(Class<? extends Person> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
            return null;
        }

        @Override
        public void write(Person person, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
            OutputStream outputStream = outputMessage.getBody();
            outputStream.write((person.toString()).getBytes(StandardCharsets.UTF_8));
        }
    }
}
```

#### 方式二

```java
@Configuration
public class WebConfig{
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {
            @Override
            public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
                HttpMessageConverter<Person> converter = new MyFirstConverter();
                converters.add(converter);
            }
        };
    }
    public static class MyFirstConverter implements HttpMessageConverter<Person>{

        @Override
        public boolean canRead(Class<?> clazz, MediaType mediaType) {
            return false;
        }

        @Override
        public boolean canWrite(Class<?> clazz, MediaType mediaType) {
            clazz.isAssignableFrom(Person.class);
        }

        @Override
        public List<MediaType> getSupportedMediaTypes() {
            return MediaType.parseMediaTypes("application/tangling");
        }

        @Override
        public Person read(Class<? extends Person> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
            return null;
        }

        @Override
        public void write(Person person, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
            OutputStream outputStream = outputMessage.getBody();
            outputStream.write((person.toString()).getBytes(StandardCharsets.UTF_8));
        }
    }
}
```

