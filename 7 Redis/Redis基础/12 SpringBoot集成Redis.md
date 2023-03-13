# `Jedis`,`Lettuce`,`RedisTemplate`介绍

# `SpringBoot`整合`Redis`流程

## `Jedis`版

- :one:创建一个常规的`SpringBoot`项目

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303121210577.png" alt="image-20230312121048472" style="zoom: 80%;" />

- **:two:在`pom.xml`配置文件中添加`Jedis`的依赖**

    - ```xml
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>4.3.1</version>
        </dependency>
        ```

- **:three:编写`SpringBoot`主启动类**

    - ```java
        @SpringBootApplication
        public class MyJedisApplication {
        
            public static void main(String[] args) {
                SpringApplication.run(MyJedisApplication.class, args);
            }
        
        }
        ```

- **:four:使用`Jedis`包编写连接`Redis-server`的代码**

    - ```java
        public class JedisTest {
            public static void main(String[] args) {
                Jedis jedisConnection = new Jedis("192.168.184.160",6379);
                jedisConnection.auth("58828738");
                jedisConnection.flushAll();
                System.out.println(jedisConnection.keys("*"));
                jedisConnection.set("k1","tangling");
                System.out.println(jedisConnection.get("k1"));
                jedisConnection.set("k2","zhangsan");
                System.out.println(jedisConnection.get("k2"));
            }
        }
        ```

## `Lettuce`版

- :one:**创建一个常规的`SpringBoot`项目**

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303121210577.png" alt="image-20230312121048472" style="zoom: 80%;" />

- **:two:在`pom.xml`配置文件中添加`Jedis`的依赖**

    - ```xml
        <dependency>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
            <version>6.2.0.RELEASE</version>
        </dependency>
        ```

- **:three:编写`SpringBoot`主启动类**

    - ```java
        @SpringBootApplication
        public class MyJedisApplication {
        
            public static void main(String[] args) {
                SpringApplication.run(MyJedisApplication.class, args);
            }
        
        }
        ```

- **:four:使用`Lettuce`包编写连接`Redis-server`的代码**

    - ```java
        public class JedisTest {
            public static void main(String[] args) {
                //构建我们的RedisURI
                RedisURI redisURI = RedisURI.Builder
                        .redis("192.168.184.160")
                        .withPort(6379)
                        .withAuthentication("default","58828738")
                        .build();
                //创建Redis-client客户端连接
                RedisClient redisClient = RedisClient.create(redisURI);
                StatefulRedisConnection<String, String> connection = redisClient.connect();
        
                //借助客户端连接来获取Redis命令执行器
                RedisCommands<String, String> command;
                command = connection.sync();
        
                //调用命令
                System.out.println(command.keys("*"));
                System.out.println(command.flushall());
                System.out.println(command.set("k1","v1"));
                System.out.println("*************"+command.get("k1"));
                //关闭释放获取的资源
                connection.close();
                redisClient.shutdown();
            }
        }
        ```

## `RedisTemplate`版

- :one:**创建一个常规的`SpringBoot`项目**

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303121210577.png" alt="image-20230312121048472" style="zoom: 80%;" />

- **:two:在`pom.xml`配置文件中添加`Jedis`的依赖**

    - ```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
        ```

- :three:**编写`application.properties`配置文件**

    - ```properties
        spring.redis.database=0
        spring.redis.host=192.168.184.160
        spring.redis.port=6379
        spring.redis.password=58828738
        spring.redis.username=default
        ```

    - ![image-20230312144740804](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303121447246.png)

- :four:**编写`SpringBoot`主启动类**

    - ```java
        @SpringBootApplication
        public class MyJedisApplication {
        
            public static void main(String[] args) {
                SpringApplication.run(MyJedisApplication.class, args);
            }
        
        }
        ```

- :four:**使用`RedisTenplate`包编写连接`Redis-server`的代码**

    - ```java
        @RestController
        public class JedisTest {
            
            private final String PREFIX = "tangling:";
            
            @Resource
            private StringRedisTemplate stringRedisTemplate;
            
            @RequestMapping("/hello")
            public void Tests() {
                int KeyID = ThreadLocalRandom.current().nextInt(1000);
                String serialNo = UUID.randomUUID().toString();
        
                String key = PREFIX + KeyID;
                String value = "我的订单：" + serialNo;
                System.out.println(key+ '\n' +value);
        
                stringRedisTemplate.opsForValue().set(key,value);
                String result = stringRedisTemplate.opsForValue().get(value);
                System.out.println(result);
            }
        }
        ```

![image-20230312153739169](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303121537206.png)

# 使用`RedisTemplate`连接`Redis`集群

- :one:**创建一个常规的`SpringBoot`项目**

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303121210577.png" alt="image-20230312121048472" style="zoom: 80%;" />

- **:two:在`pom.xml`配置文件中添加`Jedis`的依赖**

    - ```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
        ```

- :three:**编写`application.properties`配置文件**

    - ```properties
        spring.redis.password=58828738
        spring.redis.cluster.nodes=192.168.184.160:6379,192.168.184.161:6380,192.168.184.162:6381,192.168.184.163:6382,192.168.184.164:6383,192.168.184.165:6384
        ```

    - ![image-20230312144740804](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303121447246.png)

- :four:**编写`SpringBoot`主启动类**

    - ```java
        @SpringBootApplication
        public class MyJedisApplication {
        
            public static void main(String[] args) {
                SpringApplication.run(MyJedisApplication.class, args);
            }
        
        }
        ```

- :four:**使用`RedisTenplate`包编写连接`Redis-server`的代码**

    - ```java
        @RestController
        public class JedisTest {
            
            private final String PREFIX = "tangling:";
            
            @Resource
            private StringRedisTemplate stringRedisTemplate;
            
            @RequestMapping("/hello")
            public void Tests() {
                int KeyID = ThreadLocalRandom.current().nextInt(1000);
                String serialNo = UUID.randomUUID().toString();
        
                String key = PREFIX + KeyID;
                String value = "我的订单：" + serialNo;
                System.out.println(key+ '\n' +value);
        
                stringRedisTemplate.opsForValue().set(key,value);
                String result = stringRedisTemplate.opsForValue().get(value);
                System.out.println(result);
            }
        }
        ```

![image-20230312153739169](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303121537206.png)

# `RedisTemplate`连接`Redis`集群的相关问题

## :one:当`Redis`集群有主机发生宕机时,我们的的`SpringBoot`项目短时间内去访问我们的`Redis`集群会发生找不到主机等等一系列的问题

### 问题产生的原因

- `SpringBoot2`版本下,底层默认使用`Lettuce`的`Redis`连接池实现,而`Lettuce`默认情况下只会在第一次连接到我们的`Redis`集群时会去获取集群的拓扑信息,后续便不会再获取,即便我们的`Redis`集群中发生了主机宕机也不会去刷新.
- 这就导致我们的`SpringBoot`项目下,当我们的项目启动时就会自动取获取`Redis`集群的拓扑信息,并且在项目重启之前这一拓扑信息都不会变更,当然就会导致上面所说的问题产生

### 解决方案

- **添加`application.properties`配置项**

```properties
spring.redis.lettuce.cluster.refresh.adaptive=true

spring.redis.lettuce.cluster.refresh.period=2000
```

