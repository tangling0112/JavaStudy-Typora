# 主程序启动

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
		return run(new Class<?>[] { primarySource }, args);
	}
```

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
```

## `SpringApplication`的有参构造器调用过程

```java
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    //很明显，我们传入的resourceLoader为Null
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    //将我们的@SpringBootApplication注解标识的类的Class实例存储在SpringApplication实例的primarySources属性中
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //获取当前我们的SpringBoot应用的类型Reactive响应式，Servlet式，NONE三种。这里我们的应用为Servlet
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //初始化赋值bootstrapRegistryInitializers，initializers，listeners，mainApplicationClass三大属性
    this.bootstrapRegistryInitializers = new ArrayList<>(
        getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

## `SpringApplication`的`run`方法调用过程

```java
public ConfigurableApplicationContext run(String... args) {
    //这一步是用来统计我们的SpringBoot启动用时的，后面我们会看到一个Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
    long startTime = System.nanoTime();
    //
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;
    configureHeadlessProperty();
    //
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        //在这一步呢SpringBoot做了很多事情，但是总结来说呢就其首先获取了我们SpringBootApplication的类型，我们这里是Servlet类型，然后通过SpringBoot的一个自己的类的Class实例的getResource方法找到了SpringBoot自己的类路径下的/META-INF/spring.factories这个配置文件然后读取这个配置文件知道了SpringBoot的Servlet类型的应用应该使用的ConfigurableApplicationContext实现类，也就是org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext.Factory。这一步实际上就实例化了这个AnnotationConfigServletWebServerApplicationContext类
        context = createApplicationContext();
        context.setApplicationStartup(this.applicationStartup);
        //在这一步我们就进行了对我们的SpringBoot的所有配置类获取加载
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
        }
        listeners.started(context, timeTakenToStartup);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }
    try {
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
        listeners.ready(context, timeTakenToReady);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

### `SpringApplication`的`preareContext`方法调用过程

```java
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments, Banner printedBanner) {
   context.setEnvironment(environment);
   postProcessApplicationContext(context);
   applyInitializers(context);
   listeners.contextPrepared(context);
   bootstrapContext.close(context);
   if (this.logStartupInfo) {
      logStartupInfo(context.getParent() == null);
      logStartupProfileInfo(context);
   }
   // Add boot specific singleton beans
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
   beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
   if (printedBanner != null) {
      beanFactory.registerSingleton("springBootBanner", printedBanner);
   }
   if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
      ((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);
      if (beanFactory instanceof DefaultListableBeanFactory) {
         ((DefaultListableBeanFactory) beanFactory)
               .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
      }
   }
   if (this.lazyInitialization) {
      context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
   }
   context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
   // Load the sources
    //在这里我们首先获取到了我们的源配置类，在我们这个项目中为SpringBootApplication.class
   Set<Object> sources = getAllSources();
   Assert.notEmpty(sources, "Sources must not be empty");
    //在这一步我们让我们的ApplicationContext去解析我们的源配置类中的所有的Bean相关的注解，并将这些注解指示的Bean对象注册到我们的这个ApplicationContext
   load(context, sources.toArray(new Object[0]));
   listeners.contextLoaded(context);
}
```

## 源配置类上的注解分析

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
//这个注解用于将我们的源类标记为配置类
@SpringBootConfiguration
//这个注解就是我们真正协助实现自动配置的注解
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
//这一个注解使用到了我们的Spring的Import注解装配Bean对象的装配实现了`ImportSelector`接口的实现类的这种Bean对象装配模式
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

> **`@import`的三种使用模式**
>
> - 装配普通`Java`类
>     - 在这种模式下我们的`Import`会导入我们指定的这个`Java`类为`Bean`对象
> - 装配实现了`ImportSelector`接口的实现类
>     - 在这种模式下,我们的`Import`会导入这个实现类的`selectImports()`方法所返回的字符串数组中所指示的所有`Java`类
> - 装配实现了``的实现类
>

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    //在这一步就去读取了我们当前项目的目录以及项目引用了的所有Jar包的目录下的特定文件，以获取我们要导入为Bean对象的类
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    //这一步就是我们要对特定文件进行读取的步骤
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    configurations = removeDuplicates(configurations);
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    configurations = getConfigurationClassFilter().filter(configurations);
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
   	//在这一步,我们就会去通过ClassLoader的getResources的特性获取到了我们上面所说的那些特定文件的第一部分,并且将其中所指示的所有的要导入为Bean对象的类的全类名添加到了我们的configurations属性中
    List<String> configurations = new ArrayList<>(
            SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader()));
    //在这一步,我们就会去通过ClassLoader的getResources的特性获取到了我们上面所说的那些特定文件,并且借助实现了iterable接口的ImportCandidates实例的forEach方法将所有的要导入为Bean对象的类的全类名添加到了我们的configurations属性中
    ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()).forEach(configurations::add);
    Assert.notEmpty(configurations,
            "No auto configuration classes found in META-INF/spring.factories nor in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

### 第一部分特殊文件

```java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
}
```

```java
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
    Map<String, List<String>> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    result = new HashMap<>();
    try {
        //在这一步就实际加载了我们上面所说的那些特定文件,FACTORIES_RESOURCE_LOCATION=META-INF/spring.factories
        Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                String[] factoryImplementationNames =
                        StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
                for (String factoryImplementationName : factoryImplementationNames) {
                    result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
                            .add(factoryImplementationName.trim());
                }
            }
        }

        // Replace all lists with unmodifiable lists containing unique elements
        result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
                .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
        cache.put(classLoader, result);
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
    return result;
}
```

> 下面只是我们的`SpringBoot`对应的`jar`包下的`spring.factories`文件，由于我们使用的是`ClassLoader.getResources()`方法,因此实际上我们所有的`jar`包下的`/META-INF`目录下只要存在`spring.factories`文件就都会被读取

```properties
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.autoconfigure.integration.IntegrationPropertiesEnvironmentPostProcessor

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition

# Failure analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.autoconfigure.data.redis.RedisUrlSyntaxFailureAnalyzer,\
org.springframework.boot.autoconfigure.diagnostics.analyzer.NoSuchBeanDefinitionFailureAnalyzer,\
org.springframework.boot.autoconfigure.flyway.FlywayMigrationScriptMissingFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.DataSourceBeanCreationFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.HikariDriverConfigurationFailureAnalyzer,\
org.springframework.boot.autoconfigure.jooq.NoDslContextBeanFailureAnalyzer,\
org.springframework.boot.autoconfigure.r2dbc.ConnectionFactoryBeanCreationFailureAnalyzer,\
org.springframework.boot.autoconfigure.r2dbc.MissingR2dbcPoolDependencyFailureAnalyzer,\
org.springframework.boot.autoconfigure.r2dbc.MultipleConnectionPoolConfigurationsFailureAnalyzer,\
org.springframework.boot.autoconfigure.r2dbc.NoConnectionFactoryBeanFailureAnalyzer,\
org.springframework.boot.autoconfigure.session.NonUniqueSessionRepositoryFailureAnalyzer

# Template availability providers
org.springframework.boot.autoconfigure.template.TemplateAvailabilityProvider=\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.mustache.MustacheTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.web.servlet.JspTemplateAvailabilityProvider

# DataSource initializer detectors
org.springframework.boot.sql.init.dependency.DatabaseInitializerDetector=\
org.springframework.boot.autoconfigure.flyway.FlywayMigrationInitializerDatabaseInitializerDetector

# Depends on database initialization detectors
org.springframework.boot.sql.init.dependency.DependsOnDatabaseInitializationDetector=\
org.springframework.boot.autoconfigure.batch.JobRepositoryDependsOnDatabaseInitializationDetector,\
org.springframework.boot.autoconfigure.quartz.SchedulerDependsOnDatabaseInitializationDetector,\
org.springframework.boot.autoconfigure.session.JdbcIndexedSessionRepositoryDependsOnDatabaseInitializationDetector

```

 

### 第二部分特殊文件

```java
public static ImportCandidates load(Class<?> annotation, ClassLoader classLoader) {
    Assert.notNull(annotation, "'annotation' must not be null");
    ClassLoader classLoaderToUse = decideClassloader(classLoader);
    //这里的LOCATION是我们的ImportCandidates类的一个静态常量,LOCATION = "META-INF/spring/%s.imports",前面我们知道我们的annotation是AutoConfiguration.class,而它getName后的结果为org.apache.springframework.boot.autoconfigure.AutoConfiguration所以我们的String.format的结果其中之一为就是我们的SpringBoot的autoconfigure包下的一个特定文件
    String location = String.format(LOCATION, annotation.getName());
    Enumeration<URL> urls = findUrlsInClasspath(classLoaderToUse, location);
    List<String> importCandidates = new ArrayList<>();
    while (urls.hasMoreElements()) {
        URL url = urls.nextElement();
        //将特定文件中指示的所有的全类名加载到我们的importCandidates中
        importCandidates.addAll(readCandidateConfigurations(url));
    }
    return new ImportCandidates(importCandidates);
}
```

### `org.springframework.boot.autoconfigure.AutoConfiguration.imports`文件,也就是我们读取的文件

```properties
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration
org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.cassandra.CassandraRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseDataAutoConfiguration
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveDataAutoConfiguration
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRestClientAutoConfiguration
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.ldap.LdapRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveDataAutoConfiguration
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.neo4j.Neo4jDataAutoConfiguration
org.springframework.boot.autoconfigure.data.neo4j.Neo4jReactiveDataAutoConfiguration
org.springframework.boot.autoconfigure.data.neo4j.Neo4jReactiveRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.neo4j.Neo4jRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.r2dbc.R2dbcDataAutoConfiguration
org.springframework.boot.autoconfigure.data.r2dbc.R2dbcRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration
org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchRestClientAutoConfiguration
org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration
org.springframework.boot.autoconfigure.freemarker.FreeMarkerAutoConfiguration
org.springframework.boot.autoconfigure.graphql.GraphQlAutoConfiguration
org.springframework.boot.autoconfigure.graphql.data.GraphQlReactiveQueryByExampleAutoConfiguration
org.springframework.boot.autoconfigure.graphql.data.GraphQlReactiveQuerydslAutoConfiguration
org.springframework.boot.autoconfigure.graphql.data.GraphQlQueryByExampleAutoConfiguration
org.springframework.boot.autoconfigure.graphql.data.GraphQlQuerydslAutoConfiguration
org.springframework.boot.autoconfigure.graphql.reactive.GraphQlWebFluxAutoConfiguration
org.springframework.boot.autoconfigure.graphql.rsocket.GraphQlRSocketAutoConfiguration
org.springframework.boot.autoconfigure.graphql.rsocket.RSocketGraphQlClientAutoConfiguration
org.springframework.boot.autoconfigure.graphql.security.GraphQlWebFluxSecurityAutoConfiguration
org.springframework.boot.autoconfigure.graphql.security.GraphQlWebMvcSecurityAutoConfiguration
org.springframework.boot.autoconfigure.graphql.servlet.GraphQlWebMvcAutoConfiguration
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAutoConfiguration
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration
org.springframework.boot.autoconfigure.h2.H2ConsoleAutoConfiguration
org.springframework.boot.autoconfigure.hateoas.HypermediaAutoConfiguration
org.springframework.boot.autoconfigure.hazelcast.HazelcastAutoConfiguration
org.springframework.boot.autoconfigure.hazelcast.HazelcastJpaDependencyAutoConfiguration
org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration
org.springframework.boot.autoconfigure.http.codec.CodecsAutoConfiguration
org.springframework.boot.autoconfigure.influx.InfluxDbAutoConfiguration
org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration
org.springframework.boot.autoconfigure.integration.IntegrationAutoConfiguration
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration
org.springframework.boot.autoconfigure.jms.artemis.ArtemisAutoConfiguration
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration
org.springframework.boot.autoconfigure.jooq.JooqAutoConfiguration
org.springframework.boot.autoconfigure.jsonb.JsonbAutoConfiguration
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration
org.springframework.boot.autoconfigure.availability.ApplicationAvailabilityAutoConfiguration
org.springframework.boot.autoconfigure.ldap.embedded.EmbeddedLdapAutoConfiguration
org.springframework.boot.autoconfigure.ldap.LdapAutoConfiguration
org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration
org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration
org.springframework.boot.autoconfigure.mongo.MongoReactiveAutoConfiguration
org.springframework.boot.autoconfigure.mustache.MustacheAutoConfiguration
org.springframework.boot.autoconfigure.neo4j.Neo4jAutoConfiguration
org.springframework.boot.autoconfigure.netty.NettyAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
org.springframework.boot.autoconfigure.quartz.QuartzAutoConfiguration
org.springframework.boot.autoconfigure.r2dbc.R2dbcAutoConfiguration
org.springframework.boot.autoconfigure.r2dbc.R2dbcTransactionManagerAutoConfiguration
org.springframework.boot.autoconfigure.rsocket.RSocketMessagingAutoConfiguration
org.springframework.boot.autoconfigure.rsocket.RSocketRequesterAutoConfiguration
org.springframework.boot.autoconfigure.rsocket.RSocketServerAutoConfiguration
org.springframework.boot.autoconfigure.rsocket.RSocketStrategiesAutoConfiguration
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration
org.springframework.boot.autoconfigure.security.reactive.ReactiveSecurityAutoConfiguration
org.springframework.boot.autoconfigure.security.reactive.ReactiveUserDetailsServiceAutoConfiguration
org.springframework.boot.autoconfigure.security.rsocket.RSocketSecurityAutoConfiguration
org.springframework.boot.autoconfigure.security.saml2.Saml2RelyingPartyAutoConfiguration
org.springframework.boot.autoconfigure.sendgrid.SendGridAutoConfiguration
org.springframework.boot.autoconfigure.session.SessionAutoConfiguration
org.springframework.boot.autoconfigure.security.oauth2.client.servlet.OAuth2ClientAutoConfiguration
org.springframework.boot.autoconfigure.security.oauth2.client.reactive.ReactiveOAuth2ClientAutoConfiguration
org.springframework.boot.autoconfigure.security.oauth2.resource.servlet.OAuth2ResourceServerAutoConfiguration
org.springframework.boot.autoconfigure.security.oauth2.resource.reactive.ReactiveOAuth2ResourceServerAutoConfiguration
org.springframework.boot.autoconfigure.solr.SolrAutoConfiguration
org.springframework.boot.autoconfigure.sql.init.SqlInitializationAutoConfiguration
org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration
org.springframework.boot.autoconfigure.task.TaskSchedulingAutoConfiguration
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration
org.springframework.boot.autoconfigure.transaction.jta.JtaAutoConfiguration
org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration
org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.HttpHandlerAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.ReactiveMultipartAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.ReactiveWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.WebFluxAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.WebSessionIdResolverAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.error.ErrorWebFluxAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.function.client.ClientHttpConnectorAutoConfiguration
org.springframework.boot.autoconfigure.web.reactive.function.client.WebClientAutoConfiguration
#DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration
#WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.websocket.reactive.WebSocketReactiveAutoConfiguration
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketMessagingAutoConfiguration
org.springframework.boot.autoconfigure.webservices.WebServicesAutoConfiguration
org.springframework.boot.autoconfigure.webservices.client.WebServiceTemplateAutoConfiguration

```

> 现在的问题是,我们的配置文件中的类太多了,难道我们`ApplicationContext`要把他们全部都加载吗?当然是不需要的,`Spring`管理`Bean`对象中有着`@Conditional`注解以及其大量的派生注解,借助于`ConditionalOnMissingBean`,`ConditionalOnClass`,`ConditionalOnWebApplication`等一系列注解,能够让我们的`Spring`在只有上面这些注解指示的条件满足的情况下,才会真正地将我们的类注册为到我们的`ApplicationContext`中

```java
@AutoConfiguration(after = { DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
    ValidationAutoConfiguration.class })
//只有当我们使用的是Servlet构造Web应用时才加载
@ConditionalOnWebApplication(type = Type.SERVLET)
//只有当我们项目中存在Servlet类，DispatcherServlet类，WebConfigurer类的情况下才加载
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
//只有当我们没有注册WebMvcConfigurationSupport类以及其子类为Bean对象时才加载
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebMvcAutoConfiguration {
```

