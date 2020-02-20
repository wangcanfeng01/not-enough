# redis简介
redis是一个key-value[存储系统](https://baike.baidu.com/item/%E5%AD%98%E5%82%A8%E7%B3%BB%E7%BB%9F)。和Memcached类似，它支持存储的value类型相对更多，包括string(字符串)、list([链表](https://baike.baidu.com/item/%E9%93%BE%E8%A1%A8))、set(集合)、zset(sorted set --有序集合)和hash（哈希类型）
# 一、redis入门级配置
## 1.1单机版配置文件
``` java
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.time.Duration;

/**
 * @author wangcanfeng
 * @description redis配置信息
 * @Date Created in 11:07-2019/4/1
 */
@ConfigurationProperties(prefix = "spring.redis")
public class RedisStandaloneProperties {

    /**
     * redis的数据库索引
     */
    private int database = 0;

    /**
     * host地址，可以输入host的ip地址
     */
    private String host;

    /**
     * redis服务的端口号
     */
    private int port;

    /**
     * 连接超时时间
     */
    private Duration timeout;

    /**
     * lettuce的配置信息
     */
    private final Lettuce lettuce = new Lettuce();

    /**
     * lettuce的配置信息.
     */
    public static class Lettuce {

        /**
         * 停止服务的超时时间.
         */
        private Duration shutdownTimeout = Duration.ofMillis(100);

        /**
         * lettuce连接池配置信息.
         */
        private Pool pool;

        public Duration getShutdownTimeout() {
            return this.shutdownTimeout;
        }

        public void setShutdownTimeout(Duration shutdownTimeout) {
            this.shutdownTimeout = shutdownTimeout;
        }

        public Pool getPool() {
            return this.pool;
        }

        public void setPool(Pool pool) {
            this.pool = pool;
        }

    }

    /**
     * 连接池的配置信息.
     */
    public static class Pool {

        /**
         * 最大的空闲连接数
         */
        private int maxIdle;

        /**
         * 最少需要保持的空闲连接数
         */
        private int minIdle;

        /**
         * 最大活跃连接数
         */
        private int maxActive;

        /**
         * 最大等待时间，也就是阻塞时间
         */
        private Duration maxWait;

        public int getMaxIdle() {
            return this.maxIdle;
        }

        public void setMaxIdle(int maxIdle) {
            this.maxIdle = maxIdle;
        }

        public int getMinIdle() {
            return this.minIdle;
        }

        public void setMinIdle(int minIdle) {
            this.minIdle = minIdle;
        }

        public int getMaxActive() {
            return this.maxActive;
        }

        public void setMaxActive(int maxActive) {
            this.maxActive = maxActive;
        }

        public Duration getMaxWait() {
            return this.maxWait;
        }

        public void setMaxWait(Duration maxWait) {
            this.maxWait = maxWait;
        }

    }


    public int getDatabase() {
        return database;
    }

    public void setDatabase(int database) {
        this.database = database;
    }

    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }


    public int getPort() {
        return port;
    }

    public void setPort(int port) {
        this.port = port;
    }

    public Duration getTimeout() {
        return timeout;
    }

    public void setTimeout(Duration timeout) {
        this.timeout = timeout;
    }

    public Lettuce getLettuce() {
        return lettuce;
    }
}

```
## 1.2 单机版配置类
不知道为什么，官方将很多默认配置类都做成了非公有的类，在外部根本无法直接使用，那么我们只能自己动手做一个类来满足需求了。
``` java
import io.lettuce.core.ClientOptions;
import io.lettuce.core.resource.ClientResources;
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceClientConfiguration;
import org.springframework.data.redis.connection.lettuce.LettucePoolingClientConfiguration;

import java.time.Duration;
import java.util.Optional;

/**
 * @author wangcanfeng
 * @description 单机版具有连接池的配置信息
 * @Date Created in 14:50-2019/4/1
 */
public class StandaloneLettucePoolingClientConfiguration implements LettucePoolingClientConfiguration {

    /**
     * 客户端配置对象
     */
    private final LettuceClientConfiguration clientConfiguration;
    /**
     * 连接池配置对象
     */
    private final GenericObjectPoolConfig poolConfig;
    /**
     * 单机版配置对象
     */
    private final RedisStandaloneConfiguration standaloneConfiguration;

    /**
     * 功能描述: 单机版配置类的构造函数
     * @param poolConfig 连接池信息
     * @param host 主机地址
     * @param port 主机端口
     * @param timeout 连接超时时间
     * @return:
     * @since: v1.0
     * @Author:
     * @Date:
     */
    public StandaloneLettucePoolingClientConfiguration(GenericObjectPoolConfig poolConfig,
                                                       String host, Integer port,Duration timeout) {
        this.poolConfig = poolConfig;
        this.clientConfiguration = LettuceClientConfiguration.builder().commandTimeout(timeout).build();
        this.standaloneConfiguration = new RedisStandaloneConfiguration(host, port);
    }

    public RedisStandaloneConfiguration getStandaloneConfiguration() {
        return standaloneConfiguration;
    }

  
    @Override
    public GenericObjectPoolConfig getPoolConfig() {
        return poolConfig;
    }

   
    @Override
    public boolean isUseSsl() {
        return clientConfiguration.isUseSsl();
    }

   
    @Override
    public boolean isVerifyPeer() {
        return clientConfiguration.isVerifyPeer();
    }

   
    @Override
    public boolean isStartTls() {
        return clientConfiguration.isStartTls();
    }

    
    @Override
    public Optional<ClientResources> getClientResources() {
        return Optional.empty();
    }

    
    @Override
    public Optional<ClientOptions> getClientOptions() {
        return Optional.empty();
    }
    
    @Override
    public Duration getCommandTimeout() {
        return clientConfiguration.getCommandTimeout();
    }

    @Override
    public Duration getShutdownTimeout() {
        return clientConfiguration.getShutdownTimeout();
    }
}

```
上面的代码中我们可以看到配置类是实现了LettucePoolingClientConfiguration的，这里先不展开，后面通过阅读另一部分源码就可以轻松的理解了。
``` java
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.CachingConfigurerSupport;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.interceptor.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import org.springframework.util.ObjectUtils;


/**
 * @author wangcanfeng
 * @description redis配置类
 * @Date Created in 19:38-2019/3/28
 */
@EnableCaching
@Configuration
@EnableConfigurationProperties(RedisStandaloneProperties.class)
public class RedisConfig extends CachingConfigurerSupport {

    @Autowired
    private RedisStandaloneProperties properties;

    /**
     * 功能描述: 具备连接池的lettuce连接工厂
     */
    @Bean
    public LettuceConnectionFactory lettuceConnectionFactory() {
        //连接池配置
        RedisStandaloneProperties.Pool pool = properties.getLettuce().getPool();
        GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();
        poolConfig.setMaxIdle(pool.getMaxIdle());
        poolConfig.setMinIdle(pool.getMinIdle());
        poolConfig.setMaxWaitMillis(pool.getMaxWait().toMillis());
        poolConfig.setMaxTotal(pool.getMaxActive());
        // 单机配置初始化
        StandaloneLettucePoolingClientConfiguration standalone =
                new StandaloneLettucePoolingClientConfiguration(
                        poolConfig, properties.getHost(), properties.getPort(), properties.getTimeout());
        LettuceConnectionFactory factory = new LettuceConnectionFactory(standalone.getStandaloneConfiguration(), standalone);
        //一定要执行这个方法，才能使得factory生效
        factory.afterPropertiesSet();
        return factory;
    }

    /**
     * 功能描述: redis模型
     */
    @Bean
    public <K, V> RedisTemplate<K, V> redisTemplate() {
        RedisTemplate<K, V> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(lettuceConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        redisTemplate.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }

    /**
     * 功能描述: string类型的template
     * 键和值都是String,主要功能就是方便String类型的存储
     *
     * @param
     * @return:org.springframework.data.redis.core.StringRedisTemplate
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/4/1 16:45
     */
    @Bean
    public StringRedisTemplate stringRedisTemplate() {
        return new StringRedisTemplate(lettuceConnectionFactory());
    }

    /**
     * 功能描述: 缓存管理器
     */
    @Bean
    @Override
    public CacheManager cacheManager() {
        //springboot2.X以上版本引入的缓存管理器和以前版本的缓存管理器初始化方式不一样，
        // 新版本的缓存管理器通过连接工程进行初始化
        return RedisCacheManager.create(lettuceConnectionFactory());
    }

    /**
     * 功能描述: 缓存键的生成器
     */
    @Bean
    @Override
    public KeyGenerator keyGenerator() {
        return (target, method, params) -> {
            StringBuilder sb = new StringBuilder();
            // 目标类的类名
            sb.append(target.getClass().getName());
            // 目标方法名
            sb.append('#').append(method.getName());
            if (ObjectUtils.isEmpty(params)) {
                sb.append("()");
            } else {
                sb.append("(");
                for (Object param : params) {
                    if (param != null) {
                        sb.append(param).append(",");
                    } else {
                        // 参数为空的时候拼接上null
                        sb.append("null").append(",");
                    }
                }
                //移除最后一个逗号
                sb.deleteCharAt(sb.length() - 1);
                sb.append(")");
            }
            return sb.toString();
        };
    }

    /**
     * 功能描述: 缓存解析器，可以通过定时的方式，对缓存中的内容进行处理
     */
    @Override
    public CacheResolver cacheResolver() {
        return new SimpleCacheResolver(cacheManager());
    }

    /**
     * 功能描述: 缓存异常处理器
     */
    @Override
    public CacheErrorHandler errorHandler() {
        return new SimpleCacheErrorHandler();
    }
}
```
在上面的方法中，我们连接工厂bean创建的最后调用了如下方法
``` java
factory.afterPropertiesSet();
```
这个方法中的源码如下，方法内部调用的createConnectionProvider方法，会去判断配置对象是否实现于LettucePoolingClientConfiguration，然后生成对应的provider。
``` java
public void afterPropertiesSet() {
		this.client = createClient();
        //在这个方法里面，会根据配置对象是否实现于LettucePoolingClientConfiguration来生成对应的provider
		this.connectionProvider = createConnectionProvider(client, LettuceConnection.CODEC);
		this.reactiveConnectionProvider = createConnectionProvider(client, LettuceReactiveRedisConnection.CODEC);

		if (isClusterAware()) {
			this.clusterCommandExecutor = new ClusterCommandExecutor(
					new LettuceClusterTopologyProvider((RedisClusterClient) client),
					new LettuceClusterConnection.LettuceClusterNodeResourceProvider(this.connectionProvider),
					EXCEPTION_TRANSLATION);
		}
}
```

``` java
private LettuceConnectionProvider createConnectionProvider(AbstractRedisClient client, RedisCodec<?, ?> codec) {

		LettuceConnectionProvider connectionProvider = doConnectionProvider(client, codec);
       //这里判断客户端配置是否实现于LettucePoolingClientConfiguration，如果是，则创建带有连接池的provider。
		if (this.clientConfiguration instanceof LettucePoolingClientConfiguration) {
			return new LettucePoolingConnectionProvider(connectionProvider,
					(LettucePoolingClientConfiguration) this.clientConfiguration);
		}

		return connectionProvider;
	}
```
## 1.3 值的存取
``` java
//利用restTemplate进行存值
redisTemplate.opsForValue().set("test","test");
//取值
Object test=  redisTemplate.opsForValue().get("test");
```

原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)