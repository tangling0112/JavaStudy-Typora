# 1 使用`Mybatis Starter`整合我们的`Mybaits`方法与原理

## 1.1 整合方法

- **引入`Mybatis`的`Starter`**

```XML
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>3.0.1</version>
  <type>pom</type>
</dependency>
```

![image-20230225143839964](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251438398.png)

> **注意**
>
> - 这里我们可以发现,我们的`Mybatis`的`Starter`为我们导入了我们的`JDBC`的`Starter`,而我们的`JDBC`的`Starter`会为我们引入`Hikari`数据源相关的依赖,并且我们的`SpringBoot`的自动装配机制在检测到这个`Hikari`数据源相关依赖时就会自动为我们注册这个`Hikari`数据源.
> - 而这个数据源的正确配置必须要我们在我们`SpringBoot`的配置文件中给出`spring.datasource`相关的配置属性(包括数据库`URL`,`Admin`,`Password`等等),因此我们必须进行这些配置才能保证项目的正确启动
> - 并且如果我们想修改数据源为我们的`Druid`数据源,也是可行的

- **对我们`SpringBoot`的配置文件进行最基础的配置**

    - 配置我们的全局配置文件与映射文件
    - 配置我们的数据库连接

    ```Yaml
    spring:
    	datasource:
    		url: jdbc:mysql://localhost:3306/test
    		username: root
            password: 58828738
            driver-class-name: com.mysql.jdbc.Driver	
    ```

****

## 1.2 原理

> ​		前面我们介绍`Druid`数据源的基于`Starter`的自动配置,这里我们的`Mybatis`的基于`Starter`的自动配置也是同理，当然我们还要注意到我们的`Druid`数据源的自动配置，只使用了`/META-INF/spring.factories`这一个配置类注册机制.而实际上我们知道,`SpringBoot`的自动装配还有另外一个装配机制,也就是基于`/META-INF/spring/xxx.imports`文件的配置类注册机制.而我们的`Mybatis`的配置刚好就使用到了这两个机制
>
> ​		在我们的`Mybaits`的`Starter`为我们导入的第三方`jar`包的中有一个`jar`包存在`/META-INF/spring.factories`文件,这个文件就告诉我们的`SpringBoot`我们要去为我们的`Mybatis`加载哪些用于自动配置的类为配置类

![image-20230225113008619](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251130661.png)

![image-20230225112900063](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251129097.png)

![image-20230225113056465](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251130497.png)

****

## 1.3 `MybatisAutoConfiguration`配置类

```java
@org.springframework.context.annotation.Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, MybatisLanguageDriverAutoConfiguration.class })
public class MybatisAutoConfiguration implements InitializingBean {
```

> 

```java
public MybatisAutoConfiguration(MybatisProperties properties, ObjectProvider<Interceptor[]> interceptorsProvider,
    ObjectProvider<TypeHandler[]> typeHandlersProvider, ObjectProvider<LanguageDriver[]> languageDriversProvider,
    ResourceLoader resourceLoader, ObjectProvider<DatabaseIdProvider> databaseIdProvider,
    ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider,
    ObjectProvider<List<SqlSessionFactoryBeanCustomizer>> sqlSessionFactoryBeanCustomizers) {
  this.properties = properties;
  this.interceptors = interceptorsProvider.getIfAvailable();
  this.typeHandlers = typeHandlersProvider.getIfAvailable();
  this.languageDrivers = languageDriversProvider.getIfAvailable();
  this.resourceLoader = resourceLoader;
  this.databaseIdProvider = databaseIdProvider.getIfAvailable();
  this.configurationCustomizers = configurationCustomizersProvider.getIfAvailable();
  this.sqlSessionFactoryBeanCustomizers = sqlSessionFactoryBeanCustomizers.getIfAvailable();
}
```

****

### 1.3.1 `MybatisProperties`属性配置类(前缀为`mybatis`)

> **常用的配置属性介绍**
>
> - **`String configLocation`**
>     - 用于指定我们当前项目的`Mybatis`全局配置文件的位置.
>     - 通过设置这个属性可以让我们的`Mybatis`在自动加载全局配置文件时去我们指定的位置加载
> - **`String[] mapperLocations`**
>     - 用于指定我们当前项目的`Mybatis`映射文件的位置,我们可以指定多个位置,`Mybatis`会自动把我们指定的所有位置下的`Mybatis`映射文件都加载进来.
> - **`CoreConfiguration configuration`**
> - **`Properties configurationProperties`**
>
> ![image-20230225120704117](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302251207189.png)

```java
@ConfigurationProperties(prefix = MybatisProperties.MYBATIS_PREFIX)
public class MybatisProperties {

  public static final String MYBATIS_PREFIX = "mybatis";
```

****

### 1.3.2 `SqlSessionFactory`的注册

> ​		我们知道我们的`Mybatis`在使用时是借助于`SqlSession`进行使用的,而我们的`SqlSession`又是由我们的`SqlSessionFactory`获取到的.因此我们在使用`Mybaits`之前都要实例化我们的`SqlSessionFacotry`.
>
> ​		我们这里就借助于`@Bean`注解为我们的`Spring BeanFactory`中注册进了一个`SqlSessionFactory`实例的`Bean`对象,这也就意味着如果我们在我们的`SpringBoot`项目中如果需要使用`SqlSessionFactory`实例时,直接借助于我们的`Spring IOC`的自动装配机制即可很方便地取得,而不再需要去自己实例化.
>
> ​		并且我们的`SpringBoot`下的`Mybaits`的自动配置也用到了这个`Bean`对象

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
  SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
  factory.setDataSource(dataSource);
  if (properties.getConfiguration() == null || properties.getConfiguration().getVfsImpl() == null) {
    factory.setVfs(SpringBootVFS.class);
  }
  if (StringUtils.hasText(this.properties.getConfigLocation())) {
    factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
  }
  applyConfiguration(factory);
  if (this.properties.getConfigurationProperties() != null) {
    factory.setConfigurationProperties(this.properties.getConfigurationProperties());
  }
  if (!ObjectUtils.isEmpty(this.interceptors)) {
    factory.setPlugins(this.interceptors);
  }
  if (this.databaseIdProvider != null) {
    factory.setDatabaseIdProvider(this.databaseIdProvider);
  }
  if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
    factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
  }
  if (this.properties.getTypeAliasesSuperType() != null) {
    factory.setTypeAliasesSuperType(this.properties.getTypeAliasesSuperType());
  }
  if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
    factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
  }
  if (!ObjectUtils.isEmpty(this.typeHandlers)) {
    factory.setTypeHandlers(this.typeHandlers);
  }
  Resource[] mapperLocations = this.properties.resolveMapperLocations();
  if (!ObjectUtils.isEmpty(mapperLocations)) {
    factory.setMapperLocations(mapperLocations);
  }
  Set<String> factoryPropertyNames = Stream
      .of(new BeanWrapperImpl(SqlSessionFactoryBean.class).getPropertyDescriptors()).map(PropertyDescriptor::getName)
      .collect(Collectors.toSet());
  Class<? extends LanguageDriver> defaultLanguageDriver = this.properties.getDefaultScriptingLanguageDriver();
  if (factoryPropertyNames.contains("scriptingLanguageDrivers") && !ObjectUtils.isEmpty(this.languageDrivers)) {
    // Need to mybatis-spring 2.0.2+
    factory.setScriptingLanguageDrivers(this.languageDrivers);
    if (defaultLanguageDriver == null && this.languageDrivers.length == 1) {
      defaultLanguageDriver = this.languageDrivers[0].getClass();
    }
  }
  if (factoryPropertyNames.contains("defaultScriptingLanguageDriver")) {
    // Need to mybatis-spring 2.0.2+
    factory.setDefaultScriptingLanguageDriver(defaultLanguageDriver);
  }
  applySqlSessionFactoryBeanCustomizers(factory);
  return factory.getObject();
}
```

****

```java
private void applyConfiguration(SqlSessionFactoryBean factory) {
  MybatisProperties.CoreConfiguration coreConfiguration = this.properties.getConfiguration();
  Configuration configuration = null;
  if (coreConfiguration != null || !StringUtils.hasText(this.properties.getConfigLocation())) {
    configuration = new Configuration();
  }
  if (configuration != null && coreConfiguration != null) {
    coreConfiguration.applyTo(configuration);
  }
  if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
    for (ConfigurationCustomizer customizer : this.configurationCustomizers) {
      customizer.customize(configuration);
    }
  }
  factory.setConfiguration(configuration);
}

private void applySqlSessionFactoryBeanCustomizers(SqlSessionFactoryBean factory) {
  if (!CollectionUtils.isEmpty(this.sqlSessionFactoryBeanCustomizers)) {
    for (SqlSessionFactoryBeanCustomizer customizer : this.sqlSessionFactoryBeanCustomizers) {
      customizer.customize(factory);
    }
  }
}
```

****

### 1.3.3 `SqlSessionTemplate`的注册

```java
public class SqlSessionTemplate implements SqlSession, DisposableBean {
    private final SqlSessionFactory sqlSessionFactory;
    private final ExecutorType executorType;
    private final SqlSession sqlSessionProxy;
    private final PersistenceExceptionTranslator exceptionTranslator;
```

> ​		

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
  ExecutorType executorType = this.properties.getExecutorType();
  if (executorType != null) {
    return new SqlSessionTemplate(sqlSessionFactory, executorType);
  } else {
    return new SqlSessionTemplate(sqlSessionFactory);
  }
}
```

****

### 1.3.4 `MapperScannerRegistrarNotFoundConfiguration`内部类

```java
@org.springframework.context.annotation.Configuration(proxyBeanMethods = false)
@Import(AutoConfiguredMapperScannerRegistrar.class)
@ConditionalOnMissingBean({ MapperFactoryBean.class, MapperScannerConfigurer.class })
public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {
```

### 1.3.5 `AutoConfiguredMapperScannerRegistrar`内部类

```java
public static class AutoConfiguredMapperScannerRegistrar
    implements BeanFactoryAware, EnvironmentAware, ImportBeanDefinitionRegistrar {
```

****

## 1.4 `MybatisLanguageDriverAutoConfiguration`配置类

# 2 `SpringBoot`下`Mybatis`的使用

> ​		实际上,我们只要通过`SpringBoot`的配配置文件配置好我们的`Mybatis`的全局配置文件与映射文件的路径之,设置好我们的数据源相关的配置之后我们就可以开启我们的`Mybatis`的使用了
>
> ​		并且此时`Mybatis`的使用和我们常规情况下的使用方式是一致的.

# 3 `SpringBoot`基于注解配置`Mybatis`

> ​		