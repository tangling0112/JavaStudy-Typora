# 什么是数据源

# 0 `SpringBoot`的默认数据源

> ​		我们前面提到了`SpringBoot`的`DataSourceAutoConfiguration`自动配置类的`PooledDataSourceConfiguration`内部类使用`@Import`注解导入了`DataSourceConfiguration.Hikari`配置类,而这个配置类实例化了一个`Hikari`数据源的`Bean`对象
>
> ​		我们注意到,我们的`spring-boot-starter-data-jdbc`这个启动器中给我们的程序导入了我们的`Hikari`数据源的相关依赖,而这其中就包含`HikariDataSource`这个类.
>
> ​		也就是说虽然我们的`PooledDataSourceConfiguration`内部类并不会被注册为`Bean`对象,但是其`@Import`注解会生效,而我们的`DataSourceConfiguration.Hikari`正好有注解为`@ConditionalOnClass(HikariDataSource.class)`,因此也就会由于我们导入了`JDBC`启动器(这个启动器中导入了我们的`Hikari`数据源相关的依赖,其中就包含着`HikariDataSource.class`这个类)而自动导入我们的`Hikari`数据源的`Bean`对象

### `PooledDataSourceConfiguration`内部类

```java
@Configuration(proxyBeanMethods = false)
@Conditional(PooledDataSourceCondition.class)

@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
      DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.OracleUcp.class,
      DataSourceConfiguration.Generic.class, DataSourceJmxConfiguration.class })
protected static class PooledDataSourceConfiguration {
```

#### 基于`@Import`导入的`DataSourceConfiguration.Hikari`配置类(注册`Hikari`数据源)

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(HikariDataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
      matchIfMissing = true)
static class Hikari {

   @Bean
   @ConfigurationProperties(prefix = "spring.datasource.hikari")
   HikariDataSource dataSource(DataSourceProperties properties) {
      HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
      if (StringUtils.hasText(properties.getName())) {
         dataSource.setPoolName(properties.getName());
      }
      return dataSource;
   }

}
```

# 1 使用常规数据源替换方法替换`SpringBoot`使用的数据源为`Druid`数据源,并进行进阶配置从而**实现整合`Druid`**

## **能够实现替换的原理**

> - 我们前面的`Hikari`类上有一个注解`@ConditionalOnMissingBean(DataSource.class)`,也就是说当我们开发者自己注册了一个实现了`DataSource`接口的实现类的`Bean`对象时,我们的`Hikari`这个`Hikari`数据源配置类不会生效,也就是说不会再向我们的容器中注入`Hikari`数据源对应的`Bean`对象
>- 这也就是我们为什么可以通过实现自己的配置类并注册`Druid`数据源的`Bean`对象来简单方便地修改我们的`SpringBoot`使用的数据源的原因

## 替换方法

- **引入`Druid`数据源的依赖**

```XML
<dependency>
	<groupId>com.alibaba.druid</groupId>
	<artifactId>druid-wrapper</artifactId>
	<version>0.2.9</version>
	<type>pom</type>
</dependency>
```

- **构建一个我们自己的配置类注册我们的`Druid`数据源**

```java
@Configuration
public class MyDruidConfig{
    //这个@ConfigurationProperties主要是为了给我们注册的这个DruidDataSource的一些属性进行基于配置文件的初始化
    @ConfigurationProperties(prefix = "spring.datasource.druid")
    @Bean
    public DataSource createDruidDataSource(){
        return new DruidDataSource();
    }
}
```

- **在`SpringBoot`配置文件中为我们的数据源做最基础的数据库链接配置**
    - 为上面的`@ConfigurationProperties`注解服务,用于给我们的`Druid`数据源的一些属性做初始化配置


```YAMl
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 58828738
    driver-class-name: com.mysql.jdbc.Driver	
```

> `@ConfigurationProperties`的作用我们在`Spring常用注解`章节详细地介绍过.其作用主要是能够为我们这里构造的`DruidDataSource`的各个属性根据我们的配置文件进行自动初始化.

## 使用注册`Servlet`原生组件的方式配置`Druid`数据源的监控页

## `Druid`数据源监控页进阶配置

# 2 使用`Druid Starter`启动器**整合我们的`Druid`数据源**的方法与原理

> **这种方式与前面的直接注册`DruidDataSource`实例为`Bean`对象的方式相比的好处**
>
> - 

## 方法

- **引入`Druid`数据源的`Starter`**

```XML
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.17</version>
</dependency>
```

- **在`SpringBoot`配置文件中为我们的数据源做最基础的数据库链接配置**

    - 为上面的`@ConfigurationProperties`注解服务,用于给我们的`Druid`数据源的一些属性做初始化配置

    ```java
    spring:
      datasource:
        url: jdbc:mysql://localhost:3306/test
        username: root
        password: 58828738
        driver-class-name: com.mysql.jdbc.Driver	
    ```

- **直接使用`SpringBoot`的配置文件来进阶配置我们的`Druid`**

    - 示例:使用配置文件开启我们的`Druid`数据源的监控页面
        - ![image-20230225165250342](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251652383.png)
        - 开启后,我们访问`/druid`映射就可以访问到我们的`druid`数据源的监控页面
            - ![image-20230225165310757](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251653809.png)

## 原理

> ​		前面我们讲`SpringBoot`的自动配置原理的时候,我们知道我们的`SpringBoot`会读取我们的项目的类路径下以及我们的项目导入的第三方`jar`包下的`META-INF/spring.factories`文件,以获取部分用于实现自动配置的配置类.
>
> ​		正好我们在引入了我们的`Druid`数据源的`Starter`后,就会自动引入一个其对应的第三方包,而这个第三方包正好就有这个`META-INF/spring.factories`文件.
>
> ​		而这个文件中正好就给出了我们的`DruidDataSourceAutoConfigure`自动配置类,因此我们的`SpringBoot`会在启动的时候借助自动配置机制去加载我们的这个`DruidDataSourceAutoConfigure`自动配置类,而这也就使得我们实现了使用`Starter`来配置我们的`Druid`的目的

![image-20230225011407020](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302250114073.png)

![image-20230225011436278](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302250114318.png)

****

### `DruidDataSourceAutoConfigure`自动配置类

```java
@Configuration
@ConditionalOnClass(DruidDataSource.class)
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
@EnableConfigurationProperties({DruidStatProperties.class, DataSourceProperties.class})
@Import({DruidSpringAopConfiguration.class,
    DruidStatViewServletConfiguration.class,
    DruidWebStatFilterConfiguration.class,
    DruidFilterConfiguration.class})
public class DruidDataSourceAutoConfigure {

    private static final Logger LOGGER = LoggerFactory.getLogger(DruidDataSourceAutoConfigure.class);

    @Bean(initMethod = "init")
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        LOGGER.info("Init DruidDataSource");
        return new DruidDataSourceWrapper();
    }
}
```

****

#### `DruidStatProperties`属性配置(前缀为`spring.datasource.druid`)

```java
@ConfigurationProperties("spring.datasource.druid")
public class DruidStatProperties {
```

##### 监控页基本配置相关配置属性

> - `StatViewServlet.enabled`
> - `StatViewServlet.urlPattern`
> - `StatViewServlet.allow`
> - `StatViewServlet.deny`
> - `StatViewServlet.loginUsername`
> - `StatViewServlet.loginPassword`
> - `StatViewServlet.resetEnable`

![image-20230225013640939](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302250136990.png)

##### `WebStatFilter`相关配置属性

> - `WebStatFilter.enabled`
> - `WebStatFilter.urlPattern`
> - `WebStatFilter.exclusions`
> - `WebStatFilter.sessionStatMaxCount`
> - `WebStatFilter.sessionStatEnable`
> - `WebStatFilter.principalSessionName`
> - `WebStatFilter.principalCookieName`
> - `WebStatFilter.profileEnable`

![image-20230225013722447](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302250137496.png)

#### `DataSourceProperties`属性配置(前缀为`spring.datasource`)

```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {
```

> - `generateUniqueName`
> - `name`
> - `type`
> - `driverClassName`
> - `url`
> - `username`
> - `password`
> - `jndiName`
> - `embeddedDatabaseConnection`
> - `uniqueName`
> - `Xa.dataSourceClassName`

****

#### `DruidDataSourceWrapper`类

```java
@ConfigurationProperties("spring.datasource.druid")
class DruidDataSourceWrapper extends DruidDataSource implements InitializingBean {
    @Autowired
    private DataSourceProperties basicProperties;

    @Override
    public void afterPropertiesSet() throws Exception {
        //if not found prefix 'spring.datasource.druid' jdbc properties ,'spring.datasource' prefix jdbc properties will be used.
        if (super.getUsername() == null) {
            super.setUsername(basicProperties.determineUsername());
        }
        if (super.getPassword() == null) {
            super.setPassword(basicProperties.determinePassword());
        }
        if (super.getUrl() == null) {
            super.setUrl(basicProperties.determineUrl());
        }
        if (super.getDriverClassName() == null) {
            super.setDriverClassName(basicProperties.getDriverClassName());
        }
    }

    @Autowired(required = false)
    public void autoAddFilters(List<Filter> filters){
        super.filters.addAll(filters);
    }

    /**
     * Ignore the 'maxEvictableIdleTimeMillis < minEvictableIdleTimeMillis' validate,
     * it will be validated again in {@link DruidDataSource#init()}.
     *
     * for fix issue #3084, #2763
     *
     * @since 1.1.14
     */
    @Override
    public void setMaxEvictableIdleTimeMillis(long maxEvictableIdleTimeMillis) {
        try {
            super.setMaxEvictableIdleTimeMillis(maxEvictableIdleTimeMillis);
        } catch (IllegalArgumentException ignore) {
            super.maxEvictableIdleTimeMillis = maxEvictableIdleTimeMillis;
        }
    }
}
```

#### `DruidSpringAopConfiguration`类

```java
@ConditionalOnProperty("spring.datasource.druid.aop-patterns")
public class DruidSpringAopConfiguration {

    @Bean
    public Advice advice() {
        return new DruidStatInterceptor();
    }

    @Bean
    public Advisor advisor(DruidStatProperties properties) {
        return new RegexpMethodPointcutAdvisor(properties.getAopPatterns(), advice());
    }

    @Bean
    public DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
        advisorAutoProxyCreator.setProxyTargetClass(true);
        return advisorAutoProxyCreator;
    }
}
```

#### `DruidStatViewServletConfiguration`类

```java
@ConditionalOnWebApplication
@ConditionalOnProperty(name = "spring.datasource.druid.stat-view-servlet.enabled", havingValue = "true")
public class DruidStatViewServletConfiguration {
    private static final String DEFAULT_ALLOW_IP = "127.0.0.1";

    @Bean
    public ServletRegistrationBean statViewServletRegistrationBean(DruidStatProperties properties) {
        DruidStatProperties.StatViewServlet config = properties.getStatViewServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean();
        registrationBean.setServlet(new StatViewServlet());
        registrationBean.addUrlMappings(config.getUrlPattern() != null ? config.getUrlPattern() : "/druid/*");
        if (config.getAllow() != null) {
            registrationBean.addInitParameter("allow", config.getAllow());
        } else {
            registrationBean.addInitParameter("allow", DEFAULT_ALLOW_IP);
        }
        if (config.getDeny() != null) {
            registrationBean.addInitParameter("deny", config.getDeny());
        }
        if (config.getLoginUsername() != null) {
            registrationBean.addInitParameter("loginUsername", config.getLoginUsername());
        }
        if (config.getLoginPassword() != null) {
            registrationBean.addInitParameter("loginPassword", config.getLoginPassword());
        }
        if (config.getResetEnable() != null) {
            registrationBean.addInitParameter("resetEnable", config.getResetEnable());
        }
        return registrationBean;
    }
}
```

#### `DruidWebStatFilterConfiguration`类

```java
@ConditionalOnWebApplication
@ConditionalOnProperty(name = "spring.datasource.druid.web-stat-filter.enabled", havingValue = "true")
public class DruidWebStatFilterConfiguration {
    @Bean
    public FilterRegistrationBean webStatFilterRegistrationBean(DruidStatProperties properties) {
        DruidStatProperties.WebStatFilter config = properties.getWebStatFilter();
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        WebStatFilter filter = new WebStatFilter();
        registrationBean.setFilter(filter);
        registrationBean.addUrlPatterns(config.getUrlPattern() != null ? config.getUrlPattern() : "/*");
        registrationBean.addInitParameter("exclusions", config.getExclusions() != null ? config.getExclusions() : "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        if (config.getSessionStatEnable() != null) {
            registrationBean.addInitParameter("sessionStatEnable", config.getSessionStatEnable());
        }
        if (config.getSessionStatMaxCount() != null) {
            registrationBean.addInitParameter("sessionStatMaxCount", config.getSessionStatMaxCount());
        }
        if (config.getPrincipalSessionName() != null) {
            registrationBean.addInitParameter("principalSessionName", config.getPrincipalSessionName());
        }
        if (config.getPrincipalCookieName() != null) {
            registrationBean.addInitParameter("principalCookieName", config.getPrincipalCookieName());
        }
        if (config.getProfileEnable() != null) {
            registrationBean.addInitParameter("profileEnable", config.getProfileEnable());
        }
        return registrationBean;
    }
}
```

#### `DruidFilterConfiguration`类

```java
public class DruidFilterConfiguration {

    @Bean
    @ConfigurationProperties(FILTER_STAT_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_STAT_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public StatFilter statFilter() {
        return new StatFilter();
    }

    @Bean
    @ConfigurationProperties(FILTER_CONFIG_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_CONFIG_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public ConfigFilter configFilter() {
        return new ConfigFilter();
    }

    @Bean
    @ConfigurationProperties(FILTER_ENCODING_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_ENCODING_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public EncodingConvertFilter encodingConvertFilter() {
        return new EncodingConvertFilter();
    }

    @Bean
    @ConfigurationProperties(FILTER_SLF4J_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_SLF4J_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public Slf4jLogFilter slf4jLogFilter() {
        return new Slf4jLogFilter();
    }

    @Bean
    @ConfigurationProperties(FILTER_LOG4J_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_LOG4J_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public Log4jFilter log4jFilter() {
        return new Log4jFilter();
    }

    @Bean
    @ConfigurationProperties(FILTER_LOG4J2_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_LOG4J2_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public Log4j2Filter log4j2Filter() {
        return new Log4j2Filter();
    }

    @Bean
    @ConfigurationProperties(FILTER_COMMONS_LOG_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_COMMONS_LOG_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public CommonsLogFilter commonsLogFilter() {
        return new CommonsLogFilter();
    }

    @Bean
    @ConfigurationProperties(FILTER_WALL_CONFIG_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_WALL_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public WallConfig wallConfig() {
        return new WallConfig();
    }

    @Bean
    @ConfigurationProperties(FILTER_WALL_PREFIX)
    @ConditionalOnProperty(prefix = FILTER_WALL_PREFIX, name = "enabled")
    @ConditionalOnMissingBean
    public WallFilter wallFilter(WallConfig wallConfig) {
        WallFilter filter = new WallFilter();
        filter.setConfig(wallConfig);
        return filter;
    }


    private static final String FILTER_STAT_PREFIX = "spring.datasource.druid.filter.stat";
    private static final String FILTER_CONFIG_PREFIX = "spring.datasource.druid.filter.config";
    private static final String FILTER_ENCODING_PREFIX = "spring.datasource.druid.filter.encoding";
    private static final String FILTER_SLF4J_PREFIX = "spring.datasource.druid.filter.slf4j";
    private static final String FILTER_LOG4J_PREFIX = "spring.datasource.druid.filter.log4j";
    private static final String FILTER_LOG4J2_PREFIX = "spring.datasource.druid.filter.log4j2";
    private static final String FILTER_COMMONS_LOG_PREFIX = "spring.datasource.druid.filter.commons-log";
    private static final String FILTER_WALL_PREFIX = "spring.datasource.druid.filter.wall";
    private static final String FILTER_WALL_CONFIG_PREFIX = FILTER_WALL_PREFIX + ".config";


}
```