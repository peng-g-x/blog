## 自动装配机制

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- Redis依赖commons-pool 这个依赖使用连接池是必须的 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

spring-boot-starter-data-redis包含了spring-data-redis和lettuce-core两个包，默认使用的lettuce客户端。想要使用jedis客户端就需要排除lettuce这个依赖，再引入jedis的相关依赖就可以了

![img](https://img-blog.csdnimg.cn/e80a9a8b3b0c4c0eb5ce5697ccd80acf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiA57yVODLlubTnmoTmuIXpo44=,size_20,color_FFFFFF,t_70,g_se,x_16)

springBoot预设的自动化配置类都位于spring-boot-autoconfigure这个包中，只要搭建了springBoot的项目，这个包就会被引入进来

![](https://cdn.nlark.com/yuque/0/2023/png/12836966/1693234041699-9151a59f-9eb1-48f5-9d60-50c3979b75eb.png#averageHue=%23f7f7da&clientId=u5220c6d8-7bee-4&from=paste&id=uf60505e5&originHeight=1506&originWidth=1014&originalType=url&ratio=1.5625&rotation=0&showTitle=false&status=done&style=none&taskId=u49984a56-2a20-4432-af33-bcfeacc5074&title=)

而这个包下就有一个RedisAutoConfiguration这个类，顾名思义就是Redis的自动化配置。在这个类中，会引入LettuceConnectionConfiguration 和 JedisConnectionConfiguration 两个配置类，分别对应lettuce和jedis两个客户端

![](https://cdn.nlark.com/yuque/0/2023/png/12836966/1693234067622-cbda85ea-f587-40fd-88e8-5ca949370c9b.png#averageHue=%23f8f7f5&clientId=u5220c6d8-7bee-4&from=paste&id=u3408073a&originHeight=644&originWidth=1056&originalType=url&ratio=1.5625&rotation=0&showTitle=false&status=done&style=none&taskId=u0af870de-582a-42ef-8d47-361a643f484&title=)

![](https://cdn.nlark.com/yuque/0/2023/png/12836966/1693234077737-58d374dd-b383-4724-8c6e-b8b856e578e2.png#averageHue=%23f7f5f4&clientId=u5220c6d8-7bee-4&from=paste&id=u6637cde4&originHeight=710&originWidth=1056&originalType=url&ratio=1.5625&rotation=0&showTitle=false&status=done&style=none&taskId=u06c02c52-b5e8-4390-861c-720e6bffaad&title=)

![](https://cdn.nlark.com/yuque/0/2023/png/12836966/1693234079477-dbb3034b-355f-4ca2-9fc1-31a6187731c6.png#averageHue=%23f5f3f2&clientId=u5220c6d8-7bee-4&from=paste&id=ufa398c5f&originHeight=480&originWidth=1102&originalType=url&ratio=1.5625&rotation=0&showTitle=false&status=done&style=none&taskId=ue224afb7-4367-486c-aeb4-db4ad153c9e&title=)

## 配置优先级

### LettureConnectionConfiguration

```java
// 第一次参数来自于 application.properties 中以 spring.redis 为前缀的属性
// 第二个参数来自于我们注入Spring容器的RedisSentinelConfiguration实例
// 第三个参数同理，来自于我们注入Spring容器的RedisClusterConfiguration实例
LettuceConnectionConfiguration(RedisProperties properties,
    ObjectProvider<RedisSentinelConfiguration> sentinelConfigurationProvider,
    ObjectProvider<RedisClusterConfiguration> clusterConfigurationProvider) {
    super(properties, sentinelConfigurationProvider, clusterConfigurationProvider);
}
```

```java
// 如果Spring容器中还没有 RedisConnectionFactory 的实例，则向容器中注入LettuceConnectionFactory
@Bean
@ConditionalOnMissingBean(RedisConnectionFactory.class)
LettuceConnectionFactory redisConnectionFactory(
    ObjectProvider<LettuceClientConfigurationBuilderCustomizer> builderCustomizers,
    ClientResources clientResources) {
    // 获取 Lettuce 客户端配置
    LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(builderCustomizers, clientResources, getProperties().getLettuce().getPool());
    return createLettuceConnectionFactory(clientConfig);
}

// 从加载优先级来看，代码设计的逻辑是 哨兵模式 > 集群模式 > 单机模式
private LettuceConnectionFactory createLettuceConnectionFactory(LettuceClientConfiguration clientConfiguration) {
    if (getSentinelConfig() != null) {
        // 创建哨兵模式对应的连接工厂
        return new LettuceConnectionFactory(getSentinelConfig(), clientConfiguration);
    }
    if (getClusterConfiguration() != null) {
        // 创建集群模式对应的连接工厂
        return new LettuceConnectionFactory(getClusterConfiguration(), clientConfiguration);
    }
    // 创建单机模式对应的连接工厂
    return new LettuceConnectionFactory(getStandaloneConfig(), clientConfiguration);
}


protected final RedisSentinelConfiguration getSentinelConfig() {
    // 如果这个不为空，则说明Spring容器中有RedisSentinelConfiguration类型的Bean
    // 同时说明，从优先级来看，Java代码注入的RedisSentinelConfiguration类型的Bean > application.properties 中以 spring.redis.sentinel 为前缀的配置
    if (this.sentinelConfiguration != null) {
    	return this.sentinelConfiguration;
    }
    RedisProperties.Sentinel sentinelProperties = this.properties.getSentinel();
    if (sentinelProperties != null) {
        RedisSentinelConfiguration config = new RedisSentinelConfiguration();
        // 哨兵服务器可以监控多组 master-slave，这里指定连接其中某组 master-slave 的名字
        // 例如，sentinel.conf 中的配置 sentinel monitor mymaster 172.22.0.3 6379 2
        // mymaster就是我们需要的值
        config.master(sentinelProperties.getMaster());
        // 哨兵服务器的 ip:port 解析成 RedisNode
        config.setSentinels(createSentinels(sentinelProperties));
        config.setUsername(this.properties.getUsername());
        // 如果 redis-server 配置了 requirepass 属性，则客户端需要提供密码
        if (this.properties.getPassword() != null) {
          config.setPassword(RedisPassword.of(this.properties.getPassword()));
        }
        // 如果 redis-sentinel 配置了 requirepass 属性，则客户端需要提供密码
        if (sentinelProperties.getPassword() != null) {
          config.setSentinelPassword(RedisPassword.of(sentinelProperties.getPassword()));
        }
        config.setDatabase(this.properties.getDatabase());
        return config;
    }
    return null;
}


protected final RedisClusterConfiguration getClusterConfiguration() {
    // 如果这个不为空，则说明Spring容器中有RedisClusterConfiguration类型的Bean
    // 同时说明，从优先级来看，Java代码注入的RedisClusterConfiguration类型的Bean > application.properties 中以 spring.redis.cluster 前缀的配置
    if (this.clusterConfiguration != null) {
    	return this.clusterConfiguration;
    }
    if (this.properties.getCluster() == null) {
    	return null;
    }
    RedisProperties.Cluster clusterProperties = this.properties.getCluster();
    // Redis 集群节点配置，形式为 ip:port，多个节点之间用逗号分隔
    RedisClusterConfiguration config = new RedisClusterConfiguration(clusterProperties.getNodes());
    if (clusterProperties.getMaxRedirects() != null) {
    	config.setMaxRedirects(clusterProperties.getMaxRedirects());
    }
    config.setUsername(this.properties.getUsername());
    // Redis 集群节点设置了密码，则客户端需要提供密码
    if (this.properties.getPassword() != null) {
    	config.setPassword(RedisPassword.of(this.properties.getPassword()));
    }
    return config;
}

```

在application.properties同时包含

1. spring.redis.host和spring.redis.port
2. spring.redis.sentinel.master和spring.redis.sentinel.nodes
3. spring.redis.cluster.nodes和spring.redis.cluster.maxRedirects

这三组配置同时存在时，最终会采用第二种哨兵模式，而忽略第一种单机模式以及第三种集群模式对应的配置

## 单机配置

```yaml
spring:
  redis:
    host: 127.0.0.1       #Redis 地址
    port: 6379            #Redis 端口号
    database: 0           #Redis 索引（0~15，默认为0）
    timeout: 1000         #Redis 连接的超时时间
    password: 123456      #Redis 密码，如果没有就默认不配置此参数
    lettuce:              #使用 lettuce 连接池
      pool:
        max-active: 20    #连接池最大连接数（使用负值表示没有限制）
        max-wait: -1      #连接池最大阻塞等待时间（使用负值表示没有限制）
        min-idle: 0       #连接池中的最大空闲连接
        max-idle: 10      #连接池中的最小空闲连接
```

## 哨兵配置

```yaml
spring:
  redis:
    sentinel:                 #哨兵配置
      master: my-master       #Redis服务器名称
      nodes:                  #Redis哨兵地址
        - 192.168.2.11:6379
        - 192.168.2.12:6379
        - 192.168.2.13:6379
    database: 0               #Redis 索引（0~15，默认为0）
    timeout: 1000             #Redis 连接的超时时间
    password: 123456          #Redis 密码，如果没有就默认不配置此参数  
    lettuce:                  #使用 lettuce 连接池
      pool:
        max-active: 20        #连接池最大连接数（使用负值表示没有限制）
        max-wait: -1          #连接池最大阻塞等待时间（使用负值表示没有限制）
        min-idle: 0           #连接池中的最大空闲连接
        max-idle: 10          #连接池中的最小空闲连接
```

## 集群配置

```yaml
spring:
  redis:
    database: 0               #Redis 索引（0~15，默认为0）
    timeout: 1000             #Redis 连接的超时时间
    password: 123456          #Redis 密码，如果没有就默认不配置此参数  
    cluster:                  #Redis 集群配置
      max-redirects: 5        #Redis 命令执行时最多转发次数    
      nodes:                  #Redis 集群地址  
        - 192.168.2.11:6379
        - 192.168.2.12:6379
        - 192.168.2.13:6379
        - 192.168.2.11:6380
        - 192.168.2.12:6380
        - 192.168.2.13:6380
    lettuce:                  #使用 lettuce 连接池
      pool:
        max-active: 20        #连接池最大连接数（使用负值表示没有限制）
        max-wait: -1          #连接池最大阻塞等待时间（使用负值表示没有限制）
        min-idle: 0           #连接池中的最大空闲连接
        max-idle: 10          #连接池中的最小空闲连接
```

## 序列化

什么是redis的序列化呢？就是把对象存入到redis中到底以什么方式存储的，可以是二进制数据，可以是xml也可以是json。比如说我们经常会将POJO 对象存储到 Redis 中，一般情况下会使用 JSON 方式序列化成字符串，存储到 Redis 中

1. GenericToStringSerializer：可以将任何对象泛化为字符串并序列化
2. Jackson2JsonRedisSerializer：跟JacksonJsonRedisSerializer实际上是一样的
3. JacksonJsonRedisSerializer：序列化object对象为json字符串
4. JdkSerializationRedisSerializer：序列化java对象
5. StringRedisSerializer：简单的字符串序列化

如果存储的是String类型，默认使用的是StringRedisSerializer 这种序列化方式。如果存储的是对象，默认使用的是 JdkSerializationRedisSerializer，也就是Jdk的序列化方式（通过ObjectOutputStream和ObjectInputStream实现，缺点是无法直观看到存储的对象内容）

**源码分析**

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")   
    //我们可以自己定义一个RedisTemplate来替换默认的
    // 默认的template没有过多的设置，redis对象都是需要序列化的
    //两个泛型都是object类型，我们后面使用需要强制转换<String,Object>
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
         throws UnknownHostException {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    @ConditionalOnMissingBean	
    //由于String类型是Redis中最常使用的类型，所以单独提出来了一个bean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
         throws UnknownHostException {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

StringRedisTemplate和RedisTemplate的区别：

1. 两者的数据是不共通的，StringRedisTemplate只能管理StringRedisTemplate里面的数据，RedisTemplate只能管理RedisTemplate中的数据
2. RedisTemplate使用的是JdkSerializationRedisSerializer，存入数据会将数据先序列化成字节数组然后再存入数据库，StringRedisTemplate使用的是StringRedisSerializer
3. 当redis数据库里面本来存的是字符串数据或者存取的数据就是字符串类型数据的时候，就使用StringRedisTemplate即可；当数据是复杂的对象类型，取出的时候又不想做任何的数据转换，直接从redis里面取出一个对象，那么使用RedisTemplate是更好的选择
4. redisTemplate中存取数据都是字节数组，当redis中存入的数据是可读形式而非字节数组时，使用redisTemplate取值的时候会无法获取导出数据，获得的值为null，可以使用StringRedisTemplate

注意：在RedisTemplate默认是使用的Java字符串序列化，该序列化存入Redis并不能够提供可读性，比较流行的做法是替换成Jackson序列化，以JSON方式进行存储。

**方式一**

```java
@Configuration
public class RedisConfig {

    @Bean
    @SuppressWarnings("all")
    public RedisTemplate<String,Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        //一般使用<String,Object>
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);

        //JSON序列化配置
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);

        //string的序列化
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        //key采用string的序列化方式
        template.setKeySerializer(stringRedisSerializer);
        //hash的key采用string的序列化方式
        template.setHashKeySerializer(stringRedisSerializer);
        //value采用string的序列化方式
        template.setValueSerializer(jackson2JsonRedisSerializer);
        //hash的value采用string的序列化方式
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();

        return template;
    }
}
```

**方式二**

```java
@Configuration
public class RedisConfig {
    
    //自定义缓存key生成策略
//    @Bean
//    public KeyGenerator keyGenerator() {
//        return new KeyGenerator(){
//            @Override
//            public Object generate(Object target, java.lang.reflect.Method method, Object... params) {
//                StringBuffer sb = new StringBuffer();
//                sb.append(target.getClass().getName());
//                sb.append(method.getName());
//                for(Object obj:params){
//                    sb.append(obj.toString());
//                }
//                return sb.toString();
//            }
//        };
//    }
    
    @Bean
    public RedisTemplate<String,Object> template(RedisConnectionFactory factory) {
        RedisTemplate<String,Object> redisTemplate=new RedisTemplate<>();
        redisTemplate.setConnectionFactory(factory);

        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer genericJackson2JsonRedisSerializer = new GenericJackson2JsonRedisSerializer();

        redisTemplate.setKeySerializer(stringRedisSerializer);
        redisTemplate.setValueSerializer(genericJackson2JsonRedisSerializer);

        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        redisTemplate.setHashValueSerializer(genericJackson2JsonRedisSerializer);
        return redisTemplate;
    }
}
```

```java
//实体类序列化，需要实现Serializable接口
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {
    private String name;
    private int age;
}
```

```java
@SpringBootTest
class SpringbootRedisApplicationTests {

    @Autowired
    private RedisTemplate<String,Object> template;

    @Test
    void contextLoads() {
        redisTemplate.opsForValue().set("name","das");
        System.out.println(redisTemplate.opsForValue().get("name"));
    }

    @Test
    void test01() throws JsonProcessingException {
        User user = new User("da", 3);
        redisTemplate.opsForValue().set("user",user);
        System.out.println(redisTemplate.opsForValue().get("user"));
    }
}
```

## Lecctue连接池配置

```java
@Configuration
@Slf4j
public class RedisConfig {

    @Autowired
    RedisProperties redisProperties;

    /**
     * GenericObjectPoolConfig 连接池配置
     */
    @Bean
    public GenericObjectPoolConfig genericObjectPoolConfig() {
		RedisProperties.Lettuce lettuce = redisProperties.getLettuce();
        RedisProperties.Pool pool = lettuce.getPool();
        
        GenericObjectPoolConfig genericObjectPoolConfig = new GenericObjectPoolConfig();
        genericObjectPoolConfig.setMaxIdle(pool.getMaxIdle());
        genericObjectPoolConfig.setMinIdle(pool.getMinIdle());
        genericObjectPoolConfig.setMaxTotal(pool.getMaxActive());
        genericObjectPoolConfig.setMaxWaitMillis(pool.getMaxWait().toMillis());
        return genericObjectPoolConfig;
    }

    /**
     * 工厂创建了连接 但是一个单实例
     **/
    @Bean
    public LettuceConnectionFactory redisConnectionFactory(GenericObjectPoolConfig genericObjectPoolConfig) {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration(redisProperties.getHost(),redisProperties.getPort());
        redisStandaloneConfiguration.setPassword(redisProperties.getPassword());
        LettuceClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
                .commandTimeout(redisProperties.getTimeout())
                .poolConfig(genericObjectPoolConfig)
                .build();
        LettuceConnectionFactory lettuceConnectionFactory = new LettuceConnectionFactory(redisStandaloneConfiguration,clientConfig);
        return lettuceConnectionFactory;
    }
}

    /**
     * 集群配置
    **/
    @Bean
    public LettuceConnectionFactory redisConnectionFactory(GenericObjectPoolConfig genericObjectPoolConfig) {
        // 集群
        RedisClusterConfiguration redisClusterConfiguration = new RedisClusterConfiguration(redisProperties.getCluster().getNodes());
        redisClusterConfiguration.setMaxRedirects(redisProperties.getCluster().getMaxRedirects());
        redisClusterConfiguration.setPassword(redisProperties.getPassword());
        // 配置池
        LettuceClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
                .commandTimeout(redisProperties.getTimeout())
                .poolConfig(genericObjectPoolConfig)
                .build();

        LettuceConnectionFactory lettuceConnectionFactory = new LettuceConnectionFactory(redisClusterConfiguration,clientConfig);
        log.debug("redis 集群启动");
        return lettuceConnectionFactory;
    }

    /**
     * 哨兵配置
    **/
    @Bean
    public LettuceConnectionFactory redisConnectionFactory(GenericObjectPoolConfig genericObjectPoolConfig) {
        RedisSentinelConfiguration redisSentinelConfiguration = new RedisSentinelConfiguration(redisProperties.getSentinel().getMaster(),
                new HashSet<>(redisProperties.getSentinel().getNodes()));
        redisSentinelConfiguration.setPassword(redisProperties.getPassword());
        // 配置池
        LettuceClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
                .commandTimeout(redisProperties.getTimeout())
                .poolConfig(genericObjectPoolConfig)
                .build();
        LettuceConnectionFactory lettuceConnectionFactory = new LettuceConnectionFactory(redisSentinelConfiguration,clientConfig);
        log.debug("redis 哨兵启动");
        return lettuceConnectionFactory;
    }
}
```

