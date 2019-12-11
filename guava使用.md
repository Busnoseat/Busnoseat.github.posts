---
title: guava使用
date: 2019-07-29 17:16:10
tags: 缓存
---

Guava 是Google提供的一套Java工具包，而Guava Cache作为Guava的Cache部分而提供非常完善的本地缓存机制。
             <!--more--> 
Guava Cache是一个基于全内存的本地缓存实现，它提供了线程安全的实现机制。只支持增删改查，刷新规则和时效规则设定等最基本的设定。
ehcache直接在jvm虚拟机中缓存，速度快，效率高；但是缓存共享麻烦，集群分布式应用不方便。
redis是通过socket访问到缓存服务，效率比ecache低，比数据库要快很多，处理集群和分布式缓存方便，有成熟的方案。
如果如果是单个应用或者对缓存访问要求很高，并且仅仅是存取简单的操作的话用guava，
如果如果是单个应用或者对缓存访问要求很高，用ehcache。
如果是大型系统，存在缓存共享、分布式部署、缓存内容很大的，建议用redis。



以下是guava再springboot中的使用
step1： 配置文件
```
guava:
  cache:
    config:
      expireAfterAccessDuration: 24
      initialCapacity: 500
      concurrencyLevel: 5000

```

step2: 启用配置，并且生成cacheBuilder和caheManager对象
```
@ConfigurationProperties(prefix = "guava.cache.config")
public class GuavaProperties {

    private long expireAfterAccessDuration = 5;

    private int initialCapacity;

    private int concurrencyLevel;

    ......
}


@Configuration
@EnableConfigurationProperties(GuavaProperties.class)
public class GuavaConfig {

    @Autowired
    private GuavaProperties guavaProperties;

    @Bean
    public CacheBuilder<Object, Object> cacheBuilder() {
        return CacheBuilder.newBuilder()
                .initialCapacity(guavaProperties.getInitialCapacity())
                .concurrencyLevel(guavaProperties.getConcurrencyLevel())
                .expireAfterWrite(guavaProperties.getExpireAfterAccessDuration(), TimeUnit.MINUTES);
    }

    @DependsOn({"cacheBuilder"})
    @Bean
    public GuavaCacheManager guavaCacheManager(CacheBuilder<Object, Object> cacheBuilder) {
        GuavaCacheManager cacheManager = new GuavaCacheManager();
        cacheManager.setCacheBuilder(cacheBuilder);
        return cacheManager;
    }

}

```

step3: 使用
```
@Service
public class CompanyService {

    @Autowired
    private GuavaCacheManager cacheManager;

    /**
     * 最基础的是根据cacheManage获取cache，再用cache根据key获取缓存值。推荐使用这种灵活方便的使用方式
     */
    public Object getByCacheManager() {
        String cacheName = "cacheName";
        Cache cache = cacheManager.getCache(cacheName);
        return cache.get("key", () -> {
            return "1";
        });
    }

    /**
     * 使用注解的话 需要在启动类上
     * @Cacheable 用于某个方法，希望这个方法添加缓存，此方法被调用的时候，如果有缓存，此方法不执行。
     * value：缓存名称
     * key：cache根据key获取缓存的值，支持spring的el表达式。
     *    key="#id" :传入的参数
     *    key="#p0" :第一个参数
     *    key="#user.id" : User中的id值
     * condition： 指定缓存结果集
     *    condition="#user.id%2==0" ： 只有当user的id为偶数时才会进行缓存
     */
    @Cacheable(value = {"cacheName"}, key = "#user.id",condition = "#user.id%2==0")
    public String getByCacheAble() {
        return "1";
    }

    /**
     * 用于某个方法，每次被调用，此方法都执行，并把结果更新到PutCache配置的地方，一般用于缓存更新
     * 支持value、key、condition
     */
    @CachePut(value = {"cacheName"})
    public String getByCachePut() {
        return "2";
    }

    /**
     * 当标记在一个类上时表示其中所有的方法的执行都会触发缓存的清除操作,当指定了allEntries为true时，Spring Cache将忽略指定的key
     * 支持value、key、condition
     */
    @CacheEvict(value = "cacheName", allEntries = true)
    public String clear() {
        return "2";
    }
}
```