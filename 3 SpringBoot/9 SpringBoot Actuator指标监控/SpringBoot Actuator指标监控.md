# 1 什么是`SpringBoot Actuator`指标监控

# 2 使用`Actuator Starter`启动器配置我们的`SpringBoot Actuator`的方法与原理

## 方法

- **为我们的`SpringBoot`项目引入我们的`Actuator Starter`启动器**

```XML
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
  <version>2.6.3</version>
</dependency>
```

![image-20230225161547603](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251615652.png)

- 启动我们的`SpringBoot`主应用程序
- 访问`localhost:8080/actuator`即可查询到我们能够使用到的指标监控页面
    - ![image-20230225163121580](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251631618.png)

## 原理

> - 我们的`spring-boot-starter-actuator`直接或者间接地为我们导入了众多的`jar`包,这其中就有一个`spring-boot-actuator-autoconfigure`的`jar`包,这个`jar`包下的`/META-INF`目录下刚好就有`spring.factories`以及`/spring/xxx.imports`文件.
>
> - 也就是说当我们导入了`SpringBoot Actuator`的`Starter`,借助于我们的`SpringBoot`的基于`spring.factories`以及`xxx.imports`这两种文件的自动配置机制,我们的`SpringBoot`就会自动地使用我们上面谈到的`spring-boot-actuator-autoconfigure` `jar`包中的这些文件来实现我们的`SpringBoot Actuator`的自动配置.
>
> **注意**
>
> - 我们会发现有两个`xxx.imports`文件,我们应该清楚,对于我们的`SpringBoot`的自动装配机制而言,由于其读取文件使用的命名规则,只会有第二个文件被读取,而第一个是不会被我们的`SpringBoot`的自动配置机制读取的,也就是说在我们使用常规的`SpringBoot`的情况下只有第二个`imports`文件中的自动配置类才会被加载到我们的`BeanFactory`中.
>     - 当然我们只是说我们的`SpringBoot`的基于两种文件的自动配置机制不会对第一个文件生效,但我不排除可能在`SpringBoot Actuator`的某些地方使用注解加载了这个配置文件中所述的配置类
>     - ![image-20230225164116424](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251641469.png)

# 3 `SpringBoot Actuator EndPoint`

## 3.1 什么是`SpringBoot Actuator EndPoint`

## 3.2 常见`EndPoint`汇总

### **重要的**

|     `ID `      | 说明                                                         |
| :------------: | ------------------------------------------------------------ |
|   `health `    | 显示应用程序的健康信息。                                     |
|    `info `     | 显示任意的应用程序信息。                                     |
|  `mappings `   | 显示所有 `@RequestMapping` 路径的整理列表。                  |
|  `shutdown `   | 让应用程序优雅地关闭。只在使用`jar`打包时有效。默认情况下是禁 用的。 |
|    `beans `    | 显示你的应用程序中所有`Spring Bean`的完整列表。              |
|     `env `     | 暴露`Spring`的 `ConfigurableEnvironment `中的属性。          |
| `configprops ` | 显示所有 `@ConfigurationProperties` 的整理列表。             |
|   `sessions`   | 允许从`Spring Session`支持的会话存储中检索和删除用户会话。 需要一个使用`Spring Session`的基于`Servlet`的`Web`应用程序。 |
|   `logfile `   | 返回日志文件的内容（如果 `logging.file.name` 或 `logging.file.path` 属性已被设置）。 支持使用`HTTP Range` 头来检索日志文件的部分内容。 |
|   `caches `    | 显示可用的缓存。                                             |
|   `startup `   | 显示由 `ApplicationStartup `收集的启动步骤数据。要求 `SpringApplication `被配置为 `BufferingApplicationStartup`。 |
| `conditions `  | 显示对配置和自动配置类进行评估的条件，以及它们符合或不符合的 原因。 |

### 非高使用频率的

|        `ID `        | 说明                                                         |
| :-----------------: | ------------------------------------------------------------ |
|   `auditevents `    | 公开当前应用程序的审计事件信息。 需要一个 `AuditEventRepository bean`。 |
|      `flyway `      | 显示任何已经应用的`Flyway`数据库迁移。 需要一个或多个 `Flyway bean`。 |
|  `httpexchanges `   | 显示 `HTTP exchange` 信息（默认情况下，最后 $100$ 个 `HTTP request/response exchange`）。 需要一个 `HttpExchangeRepository bean`。 |
| `integrationgraph ` | 显示`Spring`集成图。 需要依赖 `spring-integration-core`。    |
|     `loggers `      | 显示和修改应用程序中`logger`的配置。                         |
|    `liquibase `     | 显示任何已经应用的`Liquibase`数据库迁移。 需要一个或多个 `Liquibase Bean`。 |
|     `metrics `      | 显示当前应用程序的 `“metrics”` 信息。                        |
|      `quartz `      | 显示有关`Quartz Scheduler Job`的信息。                       |
|  `scheduledtasks `  | 显示你的应用程序中的计划任务。                               |
|    `threaddump `    | `Performs a thread dump`.                                    |
|     `heapdump `     | 返回一个堆`dump`文件。 在`HotSpot JVM`上，返回一个 `HPROF `格式的文件。 在`OpenJ9 JVM`上，返回一个 `PHD `格式的文件。 |
|    `prometheus `    | 以可被 `Prometheus `服务器抓取的格式展示度量（`metric`）。 依赖于 `micrometer-registry-prometheus`。 |

## 3.3 使用`SpringBoot`配置文件自定义化设置我们的`EndPoint`

### 3.3.1 `EndPoint`的开启与关闭

> ​		我们的`SpringBoot Actuator`的每一个`EndPoint`都支持独立的开启与关闭.在开启状态下,只要我们的`EndPoint`是被设置为暴露的,那么我们能够通过发送对应请求访问到对应的`EndPoint`(**如果设为不暴露那么是无法访问的**),在关闭状态下我们则无论如何都无法访问.

> **注意**
>
> - 我们的`SpringBoot Actuator`的**`management.endpoints.enabled-by-default  `**属性是能够设置我们的`SpringBoot Actuator`在启动时是直接**开启所有**的`EndPoint`还是**都不开启**
>     - **默认情况下这个属性是不生效的**.(默认情况下除`shutdown`外的`EndPoint`都会被开启)只有我们开发者配置它才会生效
>     - **`true`**
>         - 如果配置为`true`那么我们的`SpringBoot Actuator`启动时,**所有的`EndPoint`**都会随之开启
>     - **`false`**
>         - 如果配置为`false`那么我们的`SpringBoot Actuator`启动时,**所有的`EndPoint`**都**不会**随之开启

- ```Properties
    #EndPoint的开启
    management.endpoint.<要设置的EndPoint的id>.enabled=true
    #EndPoint的关闭
    management.endpoint.<要设置的EndPoint的id>.enabled=false
    ```

- ```yaml
    #EndPoint的开启
    management:
    	endpoint:
    		<要设置的EndPoint的id>:
    			enabled: true
    #EndPoint的关闭
    management:
    	endpoint:
    		<要设置的EndPoint的id>:
    			enabled: false
    ```

### 3.3.2 `EndPoint`的暴露与封闭(**默认情况下只有`health`这个`EndPoint`是暴露的**)

> - 前面我们提到了我们的`EndPoint`只是开启而不暴露的话,我们也是无法访问到这个`EndPoint`的.而默认情况下`SpringBoot Actuator`只有`health`这个`EndPoint`是被暴露的,因此即便我们上面开启了一个或者所有的其他的`EndPoint`我们也照样只能访问`health`这个`EndPoint`
>
> - 因此我们需要同时将我们的`EndPoint`暴露出来,才能够使我们能够访问指定的`EndPoint`
>
> **当然,我们只暴露节点,而不开启节点同样也无法访问**

#### 基于`exclude`与`include`的暴露与封闭

> - 我们可以使用`exclude`与`include`属性来指定我们想要暴露或者封闭的`EndPoint`.
>
> - 并且,值得注意的是**如果我们使用`include`指定暴露了某个节点,又使用`exclude`封闭了这个节点,那么这种情况下我们的这个节点最后的状态会是封闭的,也就是说我们的`exclude`的优先级高于`include`**

##### **`web`形式(十分重要)**

- ```properties
    #通过web暴露所有节点
    management.endpoints.web.exposure.include='*'
    #通过web暴露指定的某个节点
    management.endpoints.web.exposure.include='指定节点的ID'
    #通过web暴露指定的某些节点
    management.endpoints.web.exposure.include='节点1 ID,节点2 ID,节点3 ID'
    
    #通过web封闭所有节点
    management.endpoints.web.exposure.include='*'
    #通过web封闭指定的某个节点
    management.endpoints.web.exposure.include='指定节点的ID'
    #通过web封闭指定的某些节点
    management.endpoints.web.exposure.include='节点1 ID,节点2 ID,节点3 ID'
    ```

- ```yaml
    #通过web暴露所有节点
    management:
    	endpoints:
    		web:
    			exposure:
    				include: '*'
    #通过web暴露指定的某个节点
    management:
    	endpoints:
    		web:
    			exposure:
    				include: '指定节点的ID'
    #通过web暴露指定的某些节点
    management:
    	endpoints:
    		web:
    			exposure:
    				include: '节点1 ID,节点2 ID,节点3 ID'
    
    #通过web封闭所有节点
    management:
    	endpoints:
    		web:
    			exposure:
    				exclude: '*'
    #通过web封闭指定的某个节点
    management:
    	endpoints:
    		web:
    			exposure:
    				exclude: '指定节点的ID'
    #通过web封闭指定的某些节点
    management:
    	endpoints:
    		web:
    			exposure:
    				exclude: '节点1 ID,节点2 ID,节点3 ID'
    ```

##### `jmx`形式(暂时不重要)

- ```properties
    #通过jmx暴露所有节点
    management.endpoints.jmx.exposure.include='*'
    #通过jmx暴露指定的某个节点
    management.endpoints.jmx.exposure.include='指定节点的ID'
    #通过jmx暴露指定的某些节点
    management.endpoints.jmx.exposure.include='节点1 ID,节点2 ID,节点3 ID'
    
    #通过jmx封闭所有节点
    management.endpoints.jmx.exposure.exclude='*'
    #通过jmx封闭指定的某个节点
    management.endpoints.jmx.exposure.exclude='指定节点的ID'
    #通过jmx封闭指定的某些节点
    management.endpoints.jmx.exposure.exclude='节点1 ID,节点2 ID,节点3 ID'
    ```

- ```yaml
    #通过jmx暴露所有节点
    management:
    	endpoints:
    		jmx:
    			exposure:
    				include: '*'
    #通过jmx暴露指定的某个节点
    management:
    	endpoints:
    		jmx:
    			exposure:
    				include: '指定节点的ID'
    #通过jmx暴露指定的某些节点
    management:
    	endpoints:
    		jmx:
    			exposure:
    				include: '节点1 ID,节点2 ID,节点3 ID'
    
    #通过jmx封闭所有节点
    management:
    	endpoints:
    		jmx:
    			exposure:
    				exclude: '*'
    #通过jmx封闭指定的某个节点
    management:
    	endpoints:
    		jmx:
    			exposure:
    				exclude: '指定节点的ID'
    #通过jmx封闭指定的某些节点
    management:
    	endpoints:
    		jmx:
    			exposure:
    				exclude: '节点1 ID,节点2 ID,节点3 ID'
    ```

## 3.4 构建我们自己的`EndPoint`

# 4 `SpringBoot Admin Server`可视化`Actuator`