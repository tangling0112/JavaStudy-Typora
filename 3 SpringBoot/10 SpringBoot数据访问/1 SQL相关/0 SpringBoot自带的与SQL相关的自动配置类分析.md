## `DataSourceAutoConfiguration`自动配置常规数据源

```java
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import(DataSourcePoolMetadataProvidersConfiguration.class)
public class DataSourceAutoConfiguration {
```

> ​		由于我们的`DataSourceAutoConfiguration`没有设置任何构造器,因此我们的`Spring`再将其实例化为一个`Bean`对象时,会调用`Java`给我们的这个类自动构建的无参构造器.

****

### `DataSourceProperties`属性配置(前缀为`spring.datasource`)

> `DataSourceProperties`所映射的`SpringBoot`配置文件中的项目
>
> - 太多了,可以去`DataSourceProperties`类中详细查看

```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {
```

### `EmbeddedDatabaseConfiguration`内部类

```java
@Configuration(proxyBeanMethods = false)
@Conditional(EmbeddedDatabaseCondition.class)
@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
@Import(EmbeddedDataSourceConfiguration.class)
protected static class EmbeddedDatabaseConfiguration {
```

****

### `EmbeddedDatabaseCondition`内部类

```java
static class EmbeddedDatabaseCondition extends SpringBootCondition {

   private static final String DATASOURCE_URL_PROPERTY = "spring.datasource.url";

   private final SpringBootCondition pooledCondition = new PooledDataSourceCondition();

   @Override
   public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
      ConditionMessage.Builder message = ConditionMessage.forCondition("EmbeddedDataSource");
      if (hasDataSourceUrlProperty(context)) {
         return ConditionOutcome.noMatch(message.because(DATASOURCE_URL_PROPERTY + " is set"));
      }
      if (anyMatches(context, metadata, this.pooledCondition)) {
         return ConditionOutcome.noMatch(message.foundExactly("supported pooled data source"));
      }
      EmbeddedDatabaseType type = EmbeddedDatabaseConnection.get(context.getClassLoader()).getType();
      if (type == null) {
         return ConditionOutcome.noMatch(message.didNotFind("embedded database").atAll());
      }
      return ConditionOutcome.match(message.found("embedded database").items(type));
   }

   private boolean hasDataSourceUrlProperty(ConditionContext context) {
      Environment environment = context.getEnvironment();
      if (environment.containsProperty(DATASOURCE_URL_PROPERTY)) {
         try {
            return StringUtils.hasText(environment.getProperty(DATASOURCE_URL_PROPERTY));
         }
         catch (IllegalArgumentException ex) {
            // Ignore unresolvable placeholder errors
         }
      }
      return false;
   }

}
```

****

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

#### 基于`@Import`导入的`DataSourceConfiguration.Generic`配置类

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.type")
static class Generic {

   @Bean
   DataSource dataSource(DataSourceProperties properties) {
      return properties.initializeDataSourceBuilder().build();
   }

}
```

****

### `PooledDataSourceCondition`内部类

```java
static class PooledDataSourceCondition extends AnyNestedCondition {

   PooledDataSourceCondition() {
      super(ConfigurationPhase.PARSE_CONFIGURATION);
   }

   @ConditionalOnProperty(prefix = "spring.datasource", name = "type")
   static class ExplicitType {

   }

   @Conditional(PooledDataSourceAvailableCondition.class)
   static class PooledDataSourceAvailable {

   }

}
```

### `PooledDataSourceAvailableCondition`内部类

```java
static class PooledDataSourceAvailableCondition extends SpringBootCondition {

   @Override
   public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
      ConditionMessage.Builder message = ConditionMessage.forCondition("PooledDataSource");
      if (DataSourceBuilder.findType(context.getClassLoader()) != null) {
         return ConditionOutcome.match(message.foundExactly("supported DataSource"));
      }
      return ConditionOutcome.noMatch(message.didNotFind("supported DataSource").atAll());
   }

}
```

****

## `JdbctemplateAutoConfiguration`自动配置`JdbcTemplate`

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, JdbcTemplate.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(JdbcProperties.class)
@Import({ DatabaseInitializationDependencyConfigurer.class, JdbcTemplateConfiguration.class,
      NamedParameterJdbcTemplateConfiguration.class })
public class JdbcTemplateAutoConfiguration {
```

> ​		由于我们的`JdbctemplateAutoConfiguration`没有设置任何构造器,因此我们的`Spring`再将其实例化为一个`Bean`对象时,会调用`Java`给我们的这个类自动构建的无参构造器.

****

### `JdbcProperties`属性配置(前缀为`spring.jdbc`)

```java
@ConfigurationProperties(prefix = "spring.jdbc")
public class JdbcProperties {
```

> ​		`JdbcProperties`所映射的`SpringBoot`配置文件中的项目
>
> - `spring.jdbc.template`
> - `spring.jdbc.template.fetchSize`
> - `spring.jdbc.template.maxRows`
> - `spring.jdbc.template.queryTimeout`

```java
private final Template template = new Template();

public Template getTemplate() {
   return this.template;
}

/**
 * {@code JdbcTemplate} settings.
 */
public static class Template {

   /**
    * Number of rows that should be fetched from the database when more rows are
    * needed. Use -1 to use the JDBC driver's default configuration.
    */
   private int fetchSize = -1;

   /**
    * Maximum number of rows. Use -1 to use the JDBC driver's default configuration.
    */
   private int maxRows = -1;

   /**
    * Query timeout. Default is to use the JDBC driver's default configuration. If a
    * duration suffix is not specified, seconds will be used.
    */
   @DurationUnit(ChronoUnit.SECONDS)
   private Duration queryTimeout;
```

****

### 基于`@Import`的`JdbcTemplateConfiguration`引入

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(JdbcOperations.class)
class JdbcTemplateConfiguration {

   @Bean
   @Primary
   JdbcTemplate jdbcTemplate(DataSource dataSource, JdbcProperties properties) {
      JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
      JdbcProperties.Template template = properties.getTemplate();
      jdbcTemplate.setFetchSize(template.getFetchSize());
      jdbcTemplate.setMaxRows(template.getMaxRows());
      if (template.getQueryTimeout() != null) {
         jdbcTemplate.setQueryTimeout((int) template.getQueryTimeout().getSeconds());
      }
      return jdbcTemplate;
   }

}
```

****

## `DataSourceTransactionManagerAutoConfiguration`自动配置事务管理

```java
@AutoConfiguration(before = TransactionAutoConfiguration.class)
@ConditionalOnClass({ JdbcTemplate.class, TransactionManager.class })
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceTransactionManagerAutoConfiguration {
```

### `DataSourceProperties`属性配置(前缀为`spring.datasource`)

> `DataSourceProperties`所映射的`SpringBoot`配置文件中的项目
>
> - 太多了,可以去`DataSourceProperties`类中详细查看

```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {
```

****

### `JdbcTransactionManagerConfiguration`内部类

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnSingleCandidate(DataSource.class)
static class JdbcTransactionManagerConfiguration {
```

> ​		由于我们的`JdbcTransactionManagerConfiguration`没有设置任何构造器,因此我们的`Spring`再将其实例化为一个`Bean`对象时,会调用`Java`给我们的这个类自动构建的无参构造器.

#### 将`DataSourceTransactionManager`注册为`Bean`对象

```java
@Bean
@ConditionalOnMissingBean(TransactionManager.class)
DataSourceTransactionManager transactionManager(Environment environment, DataSource dataSource,
      ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
   DataSourceTransactionManager transactionManager = createTransactionManager(environment, dataSource);
   transactionManagerCustomizers.ifAvailable((customizers) -> customizers.customize(transactionManager));
   return transactionManager;
}
```

```java
private DataSourceTransactionManager createTransactionManager(Environment environment, DataSource dataSource) {
   return environment.getProperty("spring.dao.exceptiontranslation.enabled", Boolean.class, Boolean.TRUE)
         ? new JdbcTransactionManager(dataSource) : new DataSourceTransactionManager(dataSource);
}
```

****

## `JndiDataSourceAutoConfiguration`自动配置`Jndi`数据源

## `XADataSourceAutoConfiguration`自动配置分布式数据源