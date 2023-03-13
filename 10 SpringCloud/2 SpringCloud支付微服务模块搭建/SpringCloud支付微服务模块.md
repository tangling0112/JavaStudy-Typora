# 构建父项目

```xml
<dependencyManagement>
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.6.13</version>
    <type>pom</type>
    <scope>import</scope>
  </dependency>

  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>2021.0.6</version>
    <type>pom</type>
    <scope>import</scope>
  </dependency>

  <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2.1.0.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
  </dependency>

  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
  </dependency>

  <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.16</version>
  </dependency>

  <dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.4</version>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>2.6.4</version>
    <type>pom</type>
    <scope>import</scope>
  </dependency>

</dependencies>
</dependencyManagement>
```



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
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.1.10</version>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
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

## 配置子模块`YAML`文件

```yaml
server:
  port: 8080

spring:
  application:
    name: PaymentBlock
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: org.gjt.mm.mysql.Driver
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 58828738
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.tangling.springcloud.entities
```

## 构建主启动类

```java
@SpringBootApplication
public class MainSpringApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainSpringApplication.class,args);
    }
}
```

## 构建业务类

### 构建`MySQL`数据表

```mysql
CREATE TABLE `payment`(
	`id` BIGINT(20) NOT NULL AUTO_INCREMENT PRIMARY KEY COMMENT 'ID',
	`serial` VARCHAR(200) DEFAULT ''
)ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

### 构建`Java Entities`实体类

#### `Payment`实体类

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Payment implements Serializable {
    private Long id;
    private String serial;
}
```

#### `CommonResult`实体类

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class CommonResult<T> {
    private Integer statusCode;
    private String message;
    private T data;

    public CommonResult(Integer statusCode, String message) {
        this(statusCode,message,null);
    }
}
```

### 构建`Dao`数据表操作接口以及`Mybatis Mapper`映射文件

#### `PaymentDao`操作接口

```java
@Mapper
public interface PaymentDao {
    public int create(Payment payment);
    public Payment getPaymentById(@Param("id") Long id);
}
```

#### `PaymentMapper.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.tangling.springcloud.dao.PaymentDao">

    <insert id="create" parameterType="com.tangling.springcloud.entities.Payment" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO payment(serial) Value(#{serial});
    </insert>

    <select id="getPaymentById" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT * FROM payment WHERE id=#{id};
    </select>
    
    <resultMap id="BaseResultMap" type="com.tangling.springcloud.entities.Payment">
        <id column="id" property="id" jdbcType="BIGINT"/>
        <id column="serial" property="serial" jdbcType="VARCHAR"/>
    </resultMap>

</mapper>
```

### 构建`PaymentService`服务接口及其`PaymentServiceImpl`实现类

#### `PaymentService`接口

```java
public interface PaymentService {
    public int create(Payment payment);
    public Payment getPaymentById(@Param("id") Long id);
}
```

#### `PaymentServiceImpl`实现类

```java
@Service
public class PaymentServiceImpl implements PaymentService{

    @Resource
    private PaymentDao paymentDao;

    @Override
    public int create(Payment payment) {
        return paymentDao.create(payment);
    }

    @Override
    public Payment getPaymentById(Long id) {
        return paymentDao.getPaymentById(id);
    }
}
```

### 构建`PaymentController`控制器

#### `PaymentController`控制器

```java
@RestController
@Slf4j
@RequestMapping("/payment")
public class PaymentController {

    @Resource
    private PaymentService paymentService;

    @PostMapping("/create")
    public CommonResult<Integer> create(@RequestBody Payment payment){
        int result = paymentService.create(payment);
        if(result>0){
            log.info("*******插入成功");
            return new CommonResult<Integer>(200,"插入成功",result);
        }
        else{
            log.info("*******插入失败");
            return new CommonResult<Integer>(444,"插入失败",result);
        }
    }

    @GetMapping("/get/{id}")
    public CommonResult<Payment> getPaymentById(@PathVariable("id") Long id){
        Payment payment = paymentService.getPaymentById(id);
        if(payment!=null){
            log.info("查询结果为："+payment);
            return new CommonResult<Payment>(200,"插入成功",payment);
        }
        else{
            log.info("未找到您的支付订单");
            return new CommonResult<Payment>(444,"未找到您的支付订单,ID:"+id,null);
        }
    }

}
```

## 开启服务

