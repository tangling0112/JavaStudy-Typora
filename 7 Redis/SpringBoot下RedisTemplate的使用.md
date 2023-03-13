# 注意

> ​		不同于`Mybatis`这种第三方服务,我们的`SpringBoot`没有为其构建`AutoConfiguration`自动配置,我们要在`SpringBoot`下整合`Mybatis`必须要借助于第三方的`mybatis-spring-boot-starter`启动器,这个启动器会为我们导入`Mybatis`的第三方`mybatis-spring-boot-autoconfigure`自动配置包
>
> ​		`Redis`作为一个十分经典的数据库,我们的`SpringBoot`已经为其构建好了`AutoConfiguration`自动配置了,其相关的自动配置类已经内置在我们的`SpringBoot`自带的`spring-boot-autoconfigure`中了

# `RedisTemplate`的`SpringBoot`下自动配置

## 自动配置类的加载

```imports
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration
```

## `RedisAutoConfiguration`自动配置类

```java
@AutoConfiguration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")
	@ConditionalOnSingleCandidate(RedisConnectionFactory.class)
	public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnSingleCandidate(RedisConnectionFactory.class)
	public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
		return new StringRedisTemplate(redisConnectionFactory);
	}

}

```

### `RedisProperties`配置类(`spring.redis`前缀)

```java
@ConfigurationProperties(prefix = "spring.redis")
public class RedisProperties {
```

```java
@ConfigurationProperties(prefix = "spring.redis")
public class RedisProperties {
	//Database index used by the connection factory.
	private int database = 0;
    
    //Connection URL. Overrides host, port, and password. User is ignored. Example:
    //redis://user:password@example.com:6379
	private String url;
    
    //Redis server host.
	private String host = "localhost";

	//Login username of the redis server.
	private String username;

	//Login password of the redis server.
	private String password;

	//Redis server port
	private int port = 6379;

	//Whether to enable SSL support
	private boolean ssl;

	//Read timeout
	private Duration timeout;

	//Connection timeout
	private Duration connectTimeout;

	//Client name to be set on connections with CLIENT SETNAME
	private String clientName;

	//Type of client to use. By default, auto-detected according to the classpath
	private ClientType clientType;

	private Sentinel sentinel;

	private Cluster cluster;

	private final Jedis jedis = new Jedis();

	private final Lettuce lettuce = new Lettuce();

	//Type of Redis client to use.
	public enum ClientType {
		//Use the Lettuce redis client.
		LETTUCE,

		//Use the Jedis redis client
		JEDIS
	}
	/**
	 * Cluster properties.
	 */
	public static class Cluster {

		/**
		 * Comma-separated list of "host:port" pairs to bootstrap from. This represents an
		 * "initial" list of cluster nodes and is required to have at least one entry.
		 */
		private List<String> nodes;

		/**
		 * Maximum number of redirects to follow when executing commands across the
		 * cluster.
		 */
		private Integer maxRedirects;
	/**
	 * Redis sentinel properties.
	 */
	public static class Sentinel {

		/**
		 * Name of the Redis server.
		 */
		private String master;

		/**
		 * Comma-separated list of "host:port" pairs.
		 */
		private List<String> nodes;

		/**
		 * Login username for authenticating with sentinel(s).
		 */
		private String username;

		/**
		 * Password for authenticating with sentinel(s).
		 */
		private String password;
	/**
	 * Jedis client properties.
	 */
	public static class Jedis {

		/**
		 * Jedis pool configuration.
		 */
		private final Pool pool = new Pool();

	}
	/**
	 * Lettuce client properties.
	 */
	public static class Lettuce {

		/**
		 * Shutdown timeout.
		 */
		private Duration shutdownTimeout = Duration.ofMillis(100);

		/**
		 * Lettuce pool configuration.
		 */
		private final Pool pool = new Pool();

		private final Cluster cluster = new Cluster();

		public static class Cluster {

			private final Refresh refresh = new Refresh();

			public static class Refresh {

				/**
				 * Whether to discover and query all cluster nodes for obtaining the
				 * cluster topology. When set to false, only the initial seed nodes are
				 * used as sources for topology discovery.
				 */
				private boolean dynamicRefreshSources = true;

				/**
				 * Cluster topology refresh period.
				 */
				private Duration period;

				/**
				 * Whether adaptive topology refreshing using all available refresh
				 * triggers should be used.
				 */
				private boolean adaptive;
			}

		}

	}
    /**
	 * Pool properties.
	 */
	public static class Pool {

		/**
		 * Whether to enable the pool. Enabled automatically if "commons-pool2" is
		 * available. With Jedis, pooling is implicitly enabled in sentinel mode and this
		 * setting only applies to single node setup.
		 */
		private Boolean enabled;

		/**
		 * Maximum number of "idle" connections in the pool. Use a negative value to
		 * indicate an unlimited number of idle connections.
		 */
		private int maxIdle = 8;

		/**
		 * Target for the minimum number of idle connections to maintain in the pool. This
		 * setting only has an effect if both it and time between eviction runs are
		 * positive.
		 */
		private int minIdle = 0;

		/**
		 * Maximum number of connections that can be allocated by the pool at a given
		 * time. Use a negative value for no limit.
		 */
		private int maxActive = 8;

		/**
		 * Maximum amount of time a connection allocation should block before throwing an
		 * exception when the pool is exhausted. Use a negative value to block
		 * indefinitely.
		 */
		private Duration maxWait = Duration.ofMillis(-1);

		/**
		 * Time between runs of the idle object evictor thread. When positive, the idle
		 * object evictor thread starts, otherwise no idle object eviction is performed.
		 */
		private Duration timeBetweenEvictionRuns;
}

```

****

