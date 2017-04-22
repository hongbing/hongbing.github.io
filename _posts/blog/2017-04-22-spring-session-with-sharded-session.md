---
layout: post
title: spring-session支持分片redis
description: 项目中的spring-session使用redis做后端存储，月活在1000万左右的时候，redis内存容量达到25G左右，随着月活的增长，应该思考一下redis扩容的问题？
categories: posts blog
---

前一篇博客解析了spring session源码，分析了spring session的内部结构。本篇将扩展spring session使用分片redis。在使用spring-session一段时间后，发现spring-session后端redis实例的内存容量不断增长，从刚开始的8G（阿里云ECS），一直扩容到现在的64G。阿里云ECS单机最大容量就到64G，集群版可支持256G。经过对redis rdb的分析，使用spring-session，每个用户占用约2K的内存。由于session key里面保存数据的差异，以及使用不同的序列化方式，每个用户占用内存大小对于不同的业务来说是不一样的。

<!-- more -->
如果看过前一篇博客，应该知道spring-session保存到redis中的数据都会过期，过期时间取决于maxInactiveInSeconds的大小，我们业务上使用的时间是1个月。因此，我们redis的内存容量可以简单的计算： 月活 * 2K

上面说过，阿里云ecs单实例最大64G，当我们月活到3000万的时候就撑不住了。3000万，很远吗？

由于业务发展很快，做运营活动时，会有大量的新增用户进来，新增用户会带来大量的redis的写操作，导致读性能下降，影响老用户登录。

当前的两大问题就是，redis的容量？，redis的读写性能提升？
 
阿里云的redis集群可以在一定时间内解决容量和性能问题，但是容量依旧不可线性扩展，且集群不支持读写分离，据说已经在roadmap里了。还有就是不知道阿里云的redis集群的实现逻辑，心理没谱，对于session如此重要的功能，需要有对其实现，包括存储都有一定的掌控度，不然影响面太广了。

不采用阿里云集群，采用的是分片redis集群的方式，用户session按照key哈希到不同的redis实例中，redis既可以线性扩展，每个redis实例承担的读写qps也相应降低。

spring-session支持redis cluster模式，但是不支持分片redis，需要自己实现。

要使用spring-session只需要一个注解@EnableRedisHttpSession，然后注入一个RedisConnectionFactory的bean即可。使用了@EnableRedisHttpSession注解，就绑定了redis单实例或者使用redis cluster模式。要想使用分片redis集群，需要自定义注解。

```
@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.TYPE})
@Documented
@Import(ShardedRedisHttpSessionConfiguration.class)
@Configuration
public @interface EnableShardedRedisHttpSession {

    /**
     * 最大不活跃时间间隔
     * @return
     */
    int maxInactiveIntervalInSeconds() default 1800;

    /**
     * spring session key的默认前缀
     *
     * @return
     */
    String keyPrefix() default "";
}
```

注解EnableShardedRedisHttpSession，表明我们要使用分片Redis，而不是spring-session默认的redis使用方式。

ShardedRedisHttpSessionConfiguration类指定分片redis session的配置。在spring session中，session创建和管理都是在SessionRepository的实现类中完成的，因此要使用分片 redis，也需要实现自定义的SessionRepository来完成session的创建和管理。

```
public class ShardedRedisOperationsSessionRepository implements SessionRepository<ShardedRedisOperationsSessionRepository.ShardedRedisSession>, MessageListener {

    private static final Logger log = LoggerFactory.getLogger(ShardedRedisOperationsSessionRepository.class);

    @Autowired
    private ShardedRedisClient shardedRedisClient;

    private String keyPrefix = SessionConst.DEFAULT_SPRING_SESSION_REDIS_PREFIX;
    private Integer defaultMaxInactiveInterval;
	... ...
}
```

自定义的ShardedRedisOperationsSessionRepository实现了SessionRepository和MessageListener接口，ShardedRedisOperationsSessionRepository类在ShardedRedisHttpSessionConfiguration里面完成初始化。在ShardedRedisOperationsSessionRepository里面注入了一个ShardedRedisClient类，该类即为我们的分片redis，对redis的读写都在这个类中完成。ShardedRedisClient可以使用不同的hash算法来分片，也可以实现redis的读写分离。

另外，由于spring session中对redis的操作都封装在RedisTemplate里面，RedisTemplate对redis的读写key和value都进行了序列化，因此还需要实现一个对redis的key和value进行序列化的类。

```
public class RedisSerializerOperations {

    private boolean enableDefaultSerializer = true;
    private RedisSerializer<?> defaultSerializer;
    private RedisSerializer keySerializer = new StringRedisSerializer();
    private RedisSerializer valueSerializer = null;
    private RedisSerializer hashKeySerializer = new StringRedisSerializer();
    private RedisSerializer hashValueSerializer = null;
    private RedisSerializer<String> stringSerializer = new StringRedisSerializer();

    public void afterSerializerPropertiesSet() {
        if (defaultSerializer == null) {
            defaultSerializer = new JdkSerializationRedisSerializer(this.getClass().getClassLoader());
        }

        if (enableDefaultSerializer) {
            if (keySerializer == null) {
                keySerializer = defaultSerializer;
            }
            if (valueSerializer == null) {
                valueSerializer = defaultSerializer;
            }
            if (hashKeySerializer == null) {
                hashKeySerializer = defaultSerializer;
            }
            if (hashValueSerializer == null) {
                hashValueSerializer = defaultSerializer;
            }
        }
    }
}
```
RedisSerializerOperations与RedisTemplate中的序列化实现方式一致，采用JdkSerializationRedisSerializer作为默认的序列化方式，用户可以根据自己需要，对key和value配置不同的序列化方式。

有上面所说的几个类实现的功能（注解类，配置类，实现SessionRepository的session管理类，分片redis client，redis序列化类）就可以扩展spring session，实现自定义的redis实现方式。


