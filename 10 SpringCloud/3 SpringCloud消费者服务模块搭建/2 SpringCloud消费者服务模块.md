# 构建子项目

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
    </dependency>
</dependencies>
```

# 构建`application.yaml`配置文件

```yaml
server:
  port: 80
```

# 构建主启动类

```java
@SpringBootApplication
public class MainSpringApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainSpringApplication.class,args);
    }
}
```

# 构建业务类

## 构建`Payment`与`CommonResult`实体类

> **直接从支付模块复制即可**

## 注册`RestTemplate`

```java
@Configuration
public class ApplicationConfiguration {
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

## 构建控制器

### `ConsumerController`控制器

```java
@RestController
@RequestMapping("/consumer/payment")
public class ConsumerController {

    private static final String PAYMENT_URL = "http://localhost:8080";
    @Resource
    private RestTemplate restTemplate;

    @PostMapping("/create")
    public CommonResult<Integer> create(@RequestBody Payment payment){
        return restTemplate.postForObject(PAYMENT_URL+"/payment/create",payment,CommonResult.class);
    }
    @GetMapping("/get/{id}")
    public CommonResult<Payment> getPaymentById(@PathVariable Long id){
        return restTemplate.getForObject(PAYMENT_URL+"/payment/get/"+id,CommonResult.class);
    }

}
```

# 注意

> ​		对于上面的例子`RestTemplate`实际上是把我们的`payment`转换为了`JSON`格式发送到我们的支付微服务模块的`/payment/create`的(如果是`Java`基础数据类型或者包装类那么就会附带在请求`URL`上作为请求参数传递)
>
> ​		因此我们如果要使用这种微服务开发模式,那么像我们的支付微服务模块中的`/payment/create`这种会要接收来自其他微服务模块的调用的部分,务必使用好`@RequestBody`注解,以便于解析来自其他模块的`JSON`格式数据
