Spring Cache 的官方文档在 [Cache Abstraction](https://docs.spring.io/spring-framework/docs/5.2.22.RELEASE/spring-framework-reference/integration.html#cache)，有两个核心的接口 `Cache` 和 `CacheManager`，在 Spring Boot 的项目中只需要在启动类或者某一个配置类上加上 `EnableCaching` 注解即可开启缓存功能。

## EnableCaching

我们都知道 `@Enable*` 能起作用全靠它上面的 `@Import` 注解，`EnableCaching` 也不例外，因此进入 `CachingConfigurationSelector` 看看它做了些什么。

Spring Cache 是支持 JSR-107 的，即 JCache 注解功能，但是我们暂时不关心它，所有与它有关的都可以放心的跳过，因此 static 代码块中能容就不用管了，它们都与 JCache 有关。

`CachingConfigurationSelector` 类中核心的方法是 `selectImports`，它的代码如下

```java
// CachingConfigurationSelector.java
public String[] selectImports(AdviceMode adviceMode) {
    // adviceMode 参数即是 EnableCaching#mode 属性的值，默认为 AdviceMode.PROXY
    switch (adviceMode) {
        case PROXY:
            return getProxyImports();
        case ASPECTJ:
            return getAspectJImports();
        default:
            return null;
    }
}
```

在 PROXY 模式下 `getProxyImports` 方法向容器中注入了两个 bean，`AutoProxyRegistrar` 和 `ProxyCachingConfiguration`。

`AutoProxyRegistrar` 会向容器中注入 `InfrastructureAdvisorAutoProxyCreator`，如果有必要它在 bean 初始化后应用 BeanFactoryCacheOperationSourceAdvisor 给 bean 创建代理。

```java
// AbstractAutoProxyCreator.java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // 省略了...

    // Create proxy if we have advice.
    // specificInterceptors 中就包含 BeanFactoryCacheOperationSourceAdvisor，
    // 在查找 advisor 的过程中会调用 AnnotationCacheOperationSource#getCacheOperations 方法解析 Cacheable、CahceEvict 等注解，
    // 如果存在这些注解则可以给 bean 创建代理
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

另一个类 `ProxyCachingConfiguration` 看起来更重要一些，它向容器中导入了 3 个 bean

* BeanFactoryCacheOperationSourceAdvisor：
* AnnotationCacheOperationSource：在 bean 创建的过程中使用 `SpringCacheAnnotationParser`将 `Cacheable`、`CacheEvict` 等注解解析为 `CacheOperation`
* CacheInterceptor：缓存拦截器，负责执行被 `Cacheable`、`CacheEvict` 等注解标记的方法

因此如果没有在项目的启动类或者配置类上添加 `EnableCaching` 注解缓存拦截器就不会生效。

## CacheAutoConfiguration

`CacheAutoConfiguration` 要生效需要有几个前提

1. 在 classpath 中存在 `CacheManager` 类
2. 在容器中存在类型为 `CacheAspectSupport` 的 bean，也就是 CacheInterceptor
3. 容器中没有类型为 `CacheManager` 的 Bean 且没有名字为 cacheResolver 的 bean

它向容器中引入了两个 Bean

* CacheManagerCustomizers：默认是没有 CacheManagerCustomizer 的，放心的略过
* CacheManagerValidator：验证容器中是否有类型为 CacheManager 的 bean，放心的略过

值得重点关注的是由 CacheAutoConfiguration 上的 Import 注解导入的 `CacheConfigurationImportSelector` 类。

Spring 默认支持的缓存类型定义在 `CacheType` 枚举中，每一种类型都对应一个配置类，定义在 `CacheConfigurations` 类中，由 CacheConfigurationImportSelector 负责导入，默认生效的是 `SimpleCacheConfiguration`，它使用 `ConcurrentHashMap` 作为缓存的存储。当我们向 classpath 加入 spring-boot-starter-data-redis 后生效的就是 RedisCacheConfiguration 配置类了。为什么呢？

直接从源码分析不生效的情况有以下两种

* classpath 中不存在相应的类：EhCacheCacheConfiguration，HazelcastCacheConfiguration，InfinispanCacheConfiguration，JCacheCacheConfiguration，CouchbaseCacheConfiguration，CaffeineCacheConfiguration
* 容器中不存在相应的 bean：GenericCacheConfiguration

现在还剩下三个 RedisCacheConfiguration、SimpleCacheConfiguration 和 NoOpCacheConfiguration，他们都有 @ConditionalOnMissingBean(CacheManager.class) 注解，只要有一个生效了其他几个都不会生效。Spring 是怎么给他们排定优先级的呢？

```java
static class CacheConfigurationImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // 重点在这里：CacheType 中枚举的定义顺序即是 CacheConfiguration 的加载顺序，
        // REDIS 排在前面，所有先加载了 redis 的缓存配置类，往容器中注入了 CacheManager，其他几个配置类都被忽略了
        CacheType[] types = CacheType.values();
        String[] imports = new String[types.length];
        for (int i = 0; i < types.length; i++) {
            imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
        }
        return imports;
    }
}
```

我们可以来验证以下，EHCACHE 排在 REDIS 的前面，我们往 classpath 加入相应的 jar 包使 EhCacheCacheConfiguration 的条件都满足，然后看看谁生效了。

打开 trace 日志级别，搜索关键字 CacheConfiguration 的日志如下

```
EhCacheCacheConfiguration:
    Did not match:
        - ResourceCondition (EhCache) did not find resource 'classpath:/ehcache.xml' (EhCacheCacheConfiguration.ConfigAvailableCondition)
    Matched:
        - @ConditionalOnClass found required classes 'net.sf.ehcache.Cache', 'org.springframework.cache.ehcache.EhCacheCacheManager' (OnClassCondition)
        - Cache org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration automatic cache type (CacheCondition)
```

好像失败了，因为没找到 `classpath:/ehcache.xml` 文件，疏忽了，还得加上再看看。

```
RedisCacheConfiguration:
    Did not match:
        - @ConditionalOnMissingBean (types: org.springframework.cache.CacheManager; SearchStrategy: all) found beans of type 'org.springframework.cache.CacheManager' cacheManager (OnBeanCondition)
    Matched:
        - @ConditionalOnClass found required class 'org.springframework.data.redis.connection.RedisConnectionFactory' (OnClassCondition)
        - Cache org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration automatic cache type (CacheCondition)
```

这次没问题了，RedisCacheConfiguration 没能匹配上，因为容器中已经有一个 CacheManager 了，断点调试也证实了这一点。

如果即存在 EHCACHE 又存在 REDIS，而 EHCACHE 又先加载生效，有没有什么办法能改变这一点呢？有的，可以通过设置 `spring.cache.type=redis` 来指定要使用得缓存存储类型。

```
EhCacheCacheConfiguration:
    Did not match:
        - Cache org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration unknown cache type (CacheCondition)
    Matched:
        - @ConditionalOnClass found required classes 'net.sf.ehcache.Cache', 'org.springframework.cache.ehcache.EhCacheCacheManager' (OnClassCondition)
```

这时候 EhCacheCacheConfiguration 就会因为缓存类型不匹配而被忽略掉。

## RedisCacheConfiguration

## 参考资料

1. [SpringBoot 中各种CacheConfiguration 加载顺序填坑探索](https://www.jianshu.com/p/8a953cd0b265)
2. [深度理解springboot集成cache缓存之源码解析](https://blog.csdn.net/Kevinnsm/article/details/112638500)
