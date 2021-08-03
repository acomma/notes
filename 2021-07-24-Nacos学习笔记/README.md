## 准备 Nacos 服务器

按照 [Nacos 快速开始](https://nacos.io/zh-cn/docs/quick-start.html) 中说明的准备 Nacos 服务器。当前下载的版本是 `nacos-server-2.0.2.zip`，将其解压到比如 `E:` 盘根目录，然后在 `E:\nacos\bin` 目录下执行 `startup.cmd -m standalone` 命令启动服务器

`standalone` 参数表示单机模式运行 Nacos，此时在控制台上可以看到如下输出

```
"nacos is starting with standalone"

         ,--.
       ,--.'|
   ,--,:  : |                                           Nacos 2.0.2
,`--.'`|  ' :                       ,---.               Running in stand alone mode, All function modules
|   :  :  | |                      '   ,'\   .--.--.    Port: 8848
:   |   \ | :  ,--.--.     ,---.  /   /   | /  /    '   Pid: 13712
|   : '  '; | /       \   /     \.   ; ,. :|  :  /`./   Console: http://192.168.224.1:8848/nacos/index.html
'   ' ;.    ;.--.  .-. | /    / ''   | |: :|  :  ;_
|   | | \   | \__\/: . ..    ' / '   | .; : \  \    `.      https://nacos.io
'   : |  ; .' ," .--.; |'   ; :__|   :    |  `----.   \
|   | '`--'  /  /  ,.  |'   | '.'|\   \  /  /  /`--'  /
'   : |     ;  :   .'   \   :    : `----'  '--'.     /
;   |.'     |  ,     .-./\   \  /            `--'---'
'---'        `--`---'     `----'
```

这里有几个信息需要注意下

* `Nacos 2.0.2`：当前 Nacos 服务器的版本是 2.0.2
* `Running in stand alone mode, All function modules`：当前 Nacos 正运行在单机模式下
* `Port: 8848`：Nacos 默认的端口是 8848
* `Console: http://192.168.224.1:8848/nacos/index.html`：控制台的地址

在浏览器中打开地址 `http://192.168.224.1:8848/nacos/index.html`，在[控制台手册](https://nacos.io/zh-cn/docs/console-guide.html)中给出的默认用户名/密码为：`nacos/nacos`。

如果要关闭 Nacos 服务器，只需要在打开一个新的控制台，进入 `E:\nacos\bin` 目录执行 `shutdown.cmd` 命令即可。

## 准备 Spring Boot 项目

按照 [Nacos Spring Boot 快速开始](https://nacos.io/zh-cn/docs/quick-start-spring-boot.html) 中启动配置管理部分操作，`nacos-config-spring-boot-starter` 使用的版本为 ~~`0.2.10`~~`0.2.7`（因为 `0.2.10` 使用的 Nacos 客户端改为了 gRPC，目前感觉有点复杂，为了减少 gRPC 的干扰，从而选择 `0.2.7`）。

`nacos-config-spring-boot-starter` 为项目引入了下面几个依赖

* ~~`com.alibaba.nacos:nacos-client:2.0.2`~~`com.alibaba.nacos:nacos-client:1.2.0`：Nacos 的 [Java SDK](https://nacos.io/zh-cn/docs/sdk.html)，提供了与 Nacos 交互的原始 API
* ~~`com.alibaba.nacos:nacos-spring-context:1.1.1`~~`com.alibaba.nacos:nacos-spring-context:0.3.6`：[Nacos 与 Spring 集成支持](https://nacos.io/zh-cn/docs/nacos-spring.html)，可以在 Spring 项目中使用 Nacos
* ~~`com.alibaba.boot:nacos-config-spring-boot-autoconfigure:0.2.10`~~`com.alibaba.boot:nacos-config-spring-boot-autoconfigure:0.2.7`：Nacos 自动配置
* ~~`com.alibaba.boot:nacos-spring-boot-base:0.2.10`~~`com.alibaba.boot:nacos-spring-boot-base:0.2.7`：提供 `NacosFailureAnalyzer`

在示例项目中使用了两个 Nacos 的注解

* `@NacosPropertySource` 来自 `nacos-spring-context`，作用类似 `@PropertySource`，只是前者是从 Nacos 中加载配置信息
* `@NacosValue` 来自 `nacos-client`，作用类似 `@Value`，只是前者增加了自动更新的功能，即当配置变更后 `@NacosValue` 注解的属性的值也会变更

那这两个注解是如何发挥作用的呢？请看下面的分析。

## 神秘的 spring.factories 文件

因为是 Spring Boot 项目，所以先来看看 `nacos-config-spring-boot-autoconfigure` 中 `spring.factories` 文件有些什么内容。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.alibaba.boot.nacos.config.autoconfigure.NacosConfigAutoConfiguration
org.springframework.boot.env.EnvironmentPostProcessor=\
  com.alibaba.boot.nacos.config.autoconfigure.NacosConfigEnvironmentProcessor
org.springframework.context.ApplicationListener=\
  com.alibaba.boot.nacos.config.logging.NacosLoggingListener
```

* `NacosConfigAutoConfiguration`：向 Spring 容器中加入与 Nacos 有关的 Bean
* `NacosConfigEnvironmentProcessor`：在刷新应用程序上下文之前从 Nacos 服务器加载配置
* `NacosLoggingListener`：重新加载 Nacos 的日志配置文件

看起来只有前两个类是我们比较关心的，因此后面的分析将忽略 `NacosLoggingListener` 类，下面先来看看 `NacosConfigEnvironmentProcessor` 干了些什么事情。

## 从 Nacos 服务器加载配置

Spring Boot 应用在准备好当前应用的环境后会广播 `ApplicationEnvironmentPreparedEvent` 事件，`ConfigFileApplicationListener` 将处理该事件，处理方式是从 `spring.factories` 文件中加载所有的 `EnvironmentPostProcessor`，并调用它们的 `postProcessEnvironment` 方法，`NacosConfigEnvironmentProcessor` 正好在其中。

```java
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    // 向 SpringApplication 添加一个初始化器，作用仍然是从 Nacos 服务器加载配置
    application.addInitializers(new NacosConfigApplicationContextInitializer(this));
    // 创建一个 NacosConfigProperties 对象，并将 environment 中以 nacos.config 开头的配置设置到该对象的属性上
    nacosConfigProperties = NacosConfigPropertiesUtils.buildNacosConfigProperties(environment);
    // 如果 nacosConfigProperties 不为空并且 nacosConfigProperties.bootstrap.logEnable 的值为 true 则从 Nacos 服务器加载配置
    if (enable()) {
        System.out.println("[Nacos Config Boot] : The preload log configuration is enabled");
        loadConfig(environment);
    }
}
```

在示例应用只在 `application.properties` 文件中配置了 Nacos 服务器的地址信息 `nacos.config.server-addr=127.0.0.1:8847`，因此 `NacosConfigProperties` 对象中大多数的值，包括 `bootstrap.logEnable`，都是默认值，因此在这里并不会调用 `loadConfig` 方法从 Nacos 服务器加载配置信息。所以不对 `loadConfig` 方法进行展开。

有一点需要注意的是 `NacosConfigProperties` 类的 `serverAddr` 属性默认值也是 `127.0.0.1:8847`，也就是说当在本地以单机模式启动 Nacos 服务器时可以不配置 `nacos.config.server-addr` 的值。

Spring Boot 应用在准备好 `ApplicationContext` 后会调用所有的初始化器，其中就有前面加入的 `NacosConfigApplicationContextInitializer`。

```java
public void initialize(ConfigurableApplicationContext context) {
    singleton.setApplicationContext(context);
    environment = context.getEnvironment();
    // 再次创建一个 NacosConfigProperties 对象，经过 EnvironmentProcessor 处理后可能从 Naocos 服务器获得了新的 nacos.config 开头的配置
    nacosConfigProperties = NacosConfigPropertiesUtils.buildNacosConfigProperties(environment);
    // 新建一个 Nacos 配置加载器
    final NacosConfigLoader configLoader = new NacosConfigLoader(nacosConfigProperties, environment, builder);
    // 在当前示例项目中 enable() 返回值为 false，因此不会进到 else 分支
    if (!enable()) {
        logger.info("[Nacos Config Boot] : The preload configuration is not enabled");
    }
    else {
        // If it opens the log level loading directly will cache
        // DeferNacosPropertySource release
        if (processor.enable()) {
            processor.publishDeferService(context);
            configLoader.addListenerIfAutoRefreshed(processor.getDeferPropertySources());
        }
        else {
            configLoader.loadConfig();
            configLoader.addListenerIfAutoRefreshed();
        }
    }

    final ConfigurableListableBeanFactory factory = context.getBeanFactory();
    // 如果 BeanFactory 不包含名字为 globalNacosProperties 的单例，则向 BeanFactory 中注册一个
    if (!factory.containsSingleton(NacosBeanUtils.GLOBAL_NACOS_PROPERTIES_BEAN_NAME)) {
        factory.registerSingleton(NacosBeanUtils.GLOBAL_NACOS_PROPERTIES_BEAN_NAME,configLoader.buildGlobalNacosProperties());
    }
}
```

`enable()` 方法稍微有一点复杂，需要单独拿出来说一下。

```java
private boolean enable() {
    return processor.enable() || nacosConfigProperties.getBootstrap().isEnable();
}
```

`processer` 就是前面添加初始化器时传入的 `NacosConfigEnvironmentProcessor`，`nacosConfigProperties` 就是在 `initialize()` 方法中创建的，因此这里先判断 `nacos.config.bootstrap.logEnable` 配置的值，如果为 `true` 则走 else 分支，否则判断 `nacos.config.bootstrap.enable` 配置的值，如果为 `true` 则走 else 分支。

总结一下，在当前这个示例项目中，未从 Nacos 服务器加载配置。因此下面来看看 `NacosConfigAutoConfiguration` 干了些什么事情。

## Nacos 自动配置

打开 `NacosConfigAutoConfiguration` 的代码

```java
@ConditionalOnProperty(name = NacosConfigConstants.ENABLED, matchIfMissing = true)
@ConditionalOnMissingBean(name = CONFIG_GLOBAL_NACOS_PROPERTIES_BEAN_NAME)
@EnableConfigurationProperties(value = NacosConfigProperties.class)
@ConditionalOnClass(name = "org.springframework.boot.context.properties.bind.Binder")
@Import(value = { NacosConfigBootBeanDefinitionRegistrar.class })
@EnableNacosConfig
public class NacosConfigAutoConfiguration {

}
```

我们看到了熟悉的 `@Import` 注解和 `@Enable...` 注解，下面后一个一个的来分析它，在这之前需要简单的说明下 Nacos 自动配置生效的条件

1. `nacos.config.enabled` 配置为 `true`，如果没配置，则默认为 `true`。
2. 容器中没有名字为 `globalNacosProperties$config` 的 Bean。
3. `Binder` 类存在。

我们先来看看 `@Import` 引入的类 `NacosConfigBootBeanDefinitionRegistrar`

```java
@Configuration
public class NacosConfigBootBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar, BeanFactoryAware {
	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		DefaultListableBeanFactory defaultListableBeanFactory = (DefaultListableBeanFactory) beanFactory;
		BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.rootBeanDefinition(NacosBootConfigurationPropertiesBinder.class);
		defaultListableBeanFactory.registerBeanDefinition(NacosBootConfigurationPropertiesBinder.BEAN_NAME, beanDefinitionBuilder.getBeanDefinition());
	}

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

	}
}
```

它向容器中注册了一个名字为 `nacosConfigurationPropertiesBinder`，类型为 `NacosBootConfigurationPropertiesBinder` 的 Bean，作用是处理 bean 上的 `@NacosConfigurationProperties` 注解，具体处理过程是怎样的，示例项目没有涉及，留待下次学习。

接下来看看 `@EnableNacosConfig` 注解的内容

```java
@Import(NacosConfigBeanDefinitionRegistrar.class)
public @interface EnableNacosConfig {
    // 一堆占位符定义

    // 构建一个全局配置
	NacosProperties globalProperties() default @NacosProperties(username = USERNAME_PLACEHOLDER, password = PASSWORD_PLACEHOLDER, endpoint = ENDPOINT_PLACEHOLDER, namespace = NAMESPACE_PLACEHOLDER, accessKey = ACCESS_KEY_PLACEHOLDER, secretKey = SECRET_KEY_PLACEHOLDER, serverAddr = SERVER_ADDR_PLACEHOLDER, contextPath = CONTEXT_PATH_PLACEHOLDER, clusterName = CLUSTER_NAME_PLACEHOLDER, encode = ENCODE_PLACEHOLDER, configLongPollTimeout = CONFIG_LONG_POLL_TIMEOUT_PLACEHOLDER, configRetryTime = CONFIG_RETRY_TIME_PLACEHOLDER, maxRetry = MAX_RETRY_PLACEHOLDER);
}
```

我们任然要关注 `@Import` 注解引入的类 `NacosConfigBeanDefinitionRegistrar`，它最主要的一个方法是 `registerBeanDefinitions`

```java
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    // 这里获取的 attributes 就是 EnableNacosConfig#globalProperties 中配置的值，目前是一堆占位符
    AnnotationAttributes attributes = fromMap(metadata.getAnnotationAttributes(EnableNacosConfig.class.getName()));
    // 注册 Nacos 全局属性 bean，名字为 globalNacosProperties$config，类型为 Properties，同时把有值的占位符替换掉
    registerGlobalNacosProperties(attributes, registry, environment, CONFIG_GLOBAL_NACOS_PROPERTIES_BEAN_NAME);
    // 注册 Nacos 通用 bean
    // 1、名字为 nacosApplicationContextHolder，类型为 ApplicationContextHolder 的 bean
    // 2、名字为 annotationNacosInjectedBeanPostProcessor， 类型为 AnnotationNacosInjectedBeanPostProcessor 的 bean
    registerNacosCommonBeans(registry);
    // 注册 Nacos 配置 bean
    // 1、名字为 propertySourcesPlaceholderConfigurer，类型为 PropertySourcesPlaceholderConfigurer 的 bean
    // 2、名字为 nacosConfigurationPropertiesBindingPostProcessor，类型为 NacosConfigurationPropertiesBindingPostProcessor 的 bean
    // 3、名字为 nacosConfigListenerMethodProcessor，类型为 NacosConfigListenerMethodProcessor 的 bean
    // 4、名字为 nacosPropertySourcePostProcessor，类型为 NacosPropertySourcePostProcessor 的 bean
    // 5、名字为 annotationNacosPropertySourceBuilder，类型为 AnnotationNacosPropertySourceBuilder 的 bean
    // 6、名字为 nacosConfigListenerExecutor，类型为 ThreadPoolExecutor 的 bean
    // 7、名字为 nacosValueAnnotationBeanPostProcessor，类型为 NacosValueAnnotationBeanPostProcessor 的 bean
    // 8、名字为 configServiceBeanBuilder，类型为 ConfigServiceBeanBuilder 的 bean
    // 9、名字为 loggingNacosConfigMetadataEventListener，类型为 LoggingNacosConfigMetadataEventListener 的 bean
    registerNacosConfigBeans(registry, environment, beanFactory);
    // 立即调用 NacosPropertySourcePostProcessor 以优先处理 @NacosPropertySource 注解
    invokeNacosPropertySourcePostProcessor(beanFactory);
}
```

在进行下一步之前，先来看看注册的这些 bean 的作用都是些什么？

1. `ApplicationContextHolder`：容器的持有者，方便在其他非 Spring 管理的对象中获取由 Spring 管理的对象
2. `AnnotationNacosInjectedBeanPostProcessor`：处理 `@NacosInjected` 注解
3. `PropertySourcesPlaceholderConfigurer`：处理 ${...} 占位符
4. `NacosConfigurationPropertiesBindingPostProcessor`：处理 bean 上的 `@NacosConfigurationProperties` 注解
5. `NacosConfigListenerMethodProcessor`：处理 `@NacosConfigListener` 注解
6. `NacosPropertySourcePostProcessor`：处理 `@NacosPropertySource`、`@NacosPropertySources`、`NacosPropertySourceXmlBeanDefinition`
7. `AnnotationNacosPropertySourceBuilder`：好像还是与处理 `@NacosPropertySources` 和 `@NacosPropertySource` 注解有关
8. `ThreadPoolExecutor`：创建了一个线程池
9. `NacosValueAnnotationBeanPostProcessor`：处理 `@NacosValue` 注解，并监听 `NacosConfigReceivedEvent` 事件
10. `ConfigServiceBeanBuilder`：`ConfigService` 创建器
11. `LoggingNacosConfigMetadataEventListener`：监听 `NacosConfigMetadataEvent` 事件，打印一下日志

现在来看看 `NacosPropertySourcePostProcessor` 对 `@NacosPropertySource` 注解的处理逻辑。

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    // 获取到的是前面注册的类型为 AnnotationNacosPropertySourceBuilder 的 bean
    String[] abstractNacosPropertySourceBuilderBeanNames = BeanUtils.getBeanNames(beanFactory, AbstractNacosPropertySourceBuilder.class);

    this.nacosPropertySourceBuilders = new ArrayList<AbstractNacosPropertySourceBuilder>(abstractNacosPropertySourceBuilderBeanNames.length);

    for (String beanName : abstractNacosPropertySourceBuilderBeanNames) {
        this.nacosPropertySourceBuilders.add(beanFactory.getBean(beanName, AbstractNacosPropertySourceBuilder.class));
    }

    NacosPropertySourcePostProcessor.beanFactory = beanFactory;
    // 获取的是前面注册的类型为 ConfigServiceBeanBuilder 的 bean
    this.configServiceBeanBuilder = getConfigServiceBeanBuilder(beanFactory);
    // 获取当前容器中所有的 bean 的名字，并依次处理
    String[] beanNames = beanFactory.getBeanDefinitionNames();

    for (String beanName : beanNames) {
        processPropertySource(beanName, beanFactory);
    }
}
```

这部分逻辑还是很简单的，重头戏都在循环中的 `processPropertySource` 方法中。

```java
private void processPropertySource(String beanName, ConfigurableListableBeanFactory beanFactory) {
    // 跳过已经处理的 bean
    if (processedBeanNames.contains(beanName)) {
        return;
    }

    BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);

    // 根据 bean 上的 @NacosPropertySources 和 @NacosPropertySource 注解创建一个或多个 NacosPropertySource 实例，
    // 这里会创建一个 ConfigService 的实例 NacosConfigService，并用装饰器模式包装为 EventPublishingConfigService，使 ConfigService 具有发布事件的能力
    // 同时调用 ConfigService#getConfig 方法从 Nacos 服务器加载配置信息
    List<NacosPropertySource> nacosPropertySources = buildNacosPropertySources(beanName, beanDefinition);

    // 依次将 NacosPropertySource 加入 environment 中
    for (NacosPropertySource nacosPropertySource : nacosPropertySources) {
        // 根据在 @NacosPropertySource 配置的 first、before、after 决定加入的具体位置，默认加在最后
        addNacosPropertySource(nacosPropertySource);
        Properties properties = configServiceBeanBuilder.resolveProperties(nacosPropertySource.getAttributesMetadata());
        // 如果 autoRefreshed 配置为 true，则创建一个 Listener 来监听配置的变化，如果配置变化了就会用新配置替换旧配置
        addListenerIfAutoRefreshed(nacosPropertySource, properties, environment);
    }

    processedBeanNames.add(beanName);
}
```

经过上面的过程我们就从 Nacos 服务器加载了配置信息，并对它的变化进行了监听。

## `@NacosValue` 的处理过程

在 bean 初始化阶段会调用 `NacosValueAnnotationBeanPostProcessor` 的 `postProcessPropertyValues` 方法完成对 `@NacosValue` 注解的属性的赋值。

接着调用 `NacosValueAnnotationBeanPostProcessor` 的 `postProcessBeforeInitialization` 方法将 `@NacosValue` 注解的字段包装成 `NacosValueTarget` 对象加入 `placeholderNacosValueTargetMap` 对象中，当发生 `NacosConfigReceivedEvent` 事件时依次对所有字段重新赋值。

## `ConfigService` 的创建过程

在 `AnnotationNacosPropertySourceBuilder` 实例化后会调用它的 `afterPropertiesSet` 方法创建一个 `NacosConfigLoader` 对象。在创建 `NacosPropertySource` 时会调用 `NacosConfigLoader` 的 `load` 方法加载配置，在这里完成 `ConfigService` 的创建。

```java
public String load(String dataId, String groupId, Properties nacosProperties) throws RuntimeException {
    try {
        // nacosServiceFactory 是 CacheableEventPublishingNacosServiceFactory 对象的一个单例且不为 null，
        configService = nacosServiceFactory != null
                ? nacosServiceFactory.createConfigService(nacosProperties)
                : NacosFactory.createConfigService(nacosProperties);
    }
    catch (NacosException e) {
        throw new RuntimeException("ConfigService can't be created with dataId :" + dataId + " , groupId : " + groupId + " , properties : " + nacosProperties, e);
    }
    // 加载配置文件的内容
    return NacosUtils.getContent(configService, dataId, groupId);
}
```

先来看看 `CacheableEventPublishingNacosServiceFactory` 对象的 `createConfigService` 方法

```java
public ConfigService createConfigService(Properties properties) throws NacosException {
    Properties copy = new Properties();
    copy.putAll(properties);
    // 1、获取 ConfigCreateWorker，在 CacheableEventPublishingNacosServiceFactory 的构造器中初始化
    // 2、调用 ConfigCreateWorker 的 run 方法创建 ConfigSerivce 对象
    return (ConfigService) createWorkerManager.get(ServiceType.CONFIG).run(copy, null);
}
```

再来看看 `ConfigCreateWorker` 的 `run` 方法

```java
public ConfigService run(Properties properties, ConfigService service) throws NacosException {
    // 先从缓存中获取 ConfigService，如果已经存在就不在创建
    String cacheKey = identify(properties);
    ConfigService configService = configServicesCache.get(cacheKey);

    if (configService == null) {
        if (service == null) {
            // 通过反射的方式创建 NacosConfigService 对象
            service = NacosFactory.createConfigService(properties);
        }
        // 使用装饰器模式赋予 ConfigService 事件发布的能力
        configService = new EventPublishingConfigService(service, properties, getSingleton().context, getSingleton().nacosConfigListenerExecutor);
        configServicesCache.put(cacheKey, configService);
    }
    return configService;
}
```

重点来看看 `NacosConfigService` 的构建过程

```java
public NacosConfigService(Properties properties) throws NacosException {
    String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
    if (StringUtils.isBlank(encodeTmp)) {
        encode = Constants.ENCODE;
    } else {
        encode = encodeTmp.trim();
    }
    initNamespace(properties);
    // 创建了一个 HttpAgent 对象，使用了装饰器模式
    // 1、ServerHttpAgent 有两个作用，
    //     1）管理 Nacos 服务器列表，完成向服务器发起的 http 请求
    //     2）定时向服务器发起登录操作，保持 token 的更新
    // 2、MetricsHttpAgent 在 1、ServerHttpAgent 的基础上增加了耗时监听指标
    agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
    agent.start();
    // 创建了一个长轮询的工作者对象
    worker = new ClientWorker(agent, configFilterChainManager, properties);
}
```

具体来看看 `ClientWorker` 的创建过程

```java
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager, final Properties properties) {
    this.agent = agent;
    this.configFilterChainManager = configFilterChainManager;

    // Initialize the timeout parameter

    init(properties);

    executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
            t.setDaemon(true);
            return t;
        }
    });

    executorService = Executors.newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
            t.setDaemon(true);
            return t;
        }
    });

    // 延迟 10ms 后调用 checkConfigInfo 方法，检查配置是否有变更，
    // 如果有变更，就调用监听器的 receiveConfigInfo 方法
    executor.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            try {
                checkConfigInfo();
            } catch (Throwable e) {
                LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
            }
        }
    }, 1L, 10L, TimeUnit.MILLISECONDS);
}
```

## 如何接收配置变更？

在前面 `NacosPropertySourcePostProcessor` 注册监听器时，注册的是经过包装后的 `DelegatingEventPublishingListener`，上一步中调用的 `receiveConfigInfo` 方法即为 `DelegatingEventPublishingListener` 的 `receiveConfigInfo` 方法。

```java
public void receiveConfigInfo(String content) {
    onReceived(content);
    publishEvent(content);
}

private void publishEvent(String content) {
    NacosConfigReceivedEvent event = new NacosConfigReceivedEvent(configService, dataId, groupId, content, configType);
    // 发布的事件会被 NacosValueAnnotationBeanPostProcessor#onApplicationEvent 方法监听到，
    // 从而更新 bean 中被 @NacosValue 注解的属性的值
    applicationEventPublisher.publishEvent(event);
}

private void onReceived(String content) {
    // delegate 即为 NacosPropertySourcePostProcessor#addListenerIfAutoRefreshed 方法中创建的原始 Listener，
    // 它会更新 environment 中的 NacosPropertySource 对象
    delegate.receiveConfigInfo(content);
}
```

## 加载配置的逻辑

在前面创建 `NacosPropertySource` 的过程中会调用 `NacosUtils#getContent` 获取配置文件的内容。

```java
public static String getContent(ConfigService configService, String dataId, String groupId) {
    String content = null;
    try {
        // 这里的 configService 的类型是 EventPublishingConfigService
        content = configService.getConfig(dataId, groupId, DEFAULT_TIMEOUT);
    }
    catch (NacosException e) {
        if (logger.isErrorEnabled()) {
            logger.error("Can't get content from dataId : " + dataId + " , groupId : " + groupId, e);
        }
    }
    return content;
}
```

`getContent` 方法调用 `EventPublishingConfigService` 的 `getConfig` 方法加载配置内容，下面就来看看具体的逻辑

```java
public String getConfig(String dataId, String group, long timeoutMs) throws NacosException {
    try {
        // 这里的 configService 的类型是 NacosConfigService
        return configService.getConfig(dataId, group, timeoutMs);
    }
    catch (NacosException e) {
        // 如果超时会向外发布 NacosConfigTimeoutEvent 事件
        if (NacosException.SERVER_ERROR == e.getErrCode()) { // timeout error
            publishEvent(new NacosConfigTimeoutEvent(configService, dataId, group, timeoutMs, e.getErrMsg()));
        }
        throw e; // re-throw NacosException
    }
}
```

`NacosConfigService#getConfig` 方法调用了内部的 `getConfigInner` 方法

```java
private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
    group = null2defaultGroup(group);
    ParamUtils.checkKeyParam(dataId, group);
    ConfigResponse cr = new ConfigResponse();

    cr.setDataId(dataId);
    cr.setTenant(tenant);
    cr.setGroup(group);

    // 优先使用本地配置
    String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
    if (content != null) {
        LOGGER.warn("[{}] [get-config] get failover ok, dataId={}, group={}, tenant={}, config={}", agent.getName(), dataId, group, tenant, ContentUtils.truncateContent(content));
        cr.setContent(content);
        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();
        return content;
    }

    // 其次是用服务端配置
    try {
        String[] ct = worker.getServerConfig(dataId, group, tenant, timeoutMs);
        cr.setContent(ct[0]);

        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();

        return content;
    } catch (NacosException ioe) {
        if (NacosException.NO_RIGHT == ioe.getErrCode()) {
            throw ioe;
        }
        LOGGER.warn("[{}] [get-config] get from server error, dataId={}, group={}, tenant={}, msg={}", agent.getName(), dataId, group, tenant, ioe.toString());
    }

    // 最后使用快照配置
    LOGGER.warn("[{}] [get-config] get snapshot ok, dataId={}, group={}, tenant={}, config={}", agent.getName(), dataId, group, tenant, ContentUtils.truncateContent(content));
    content = LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
    cr.setContent(content);
    configFilterChainManager.doFilter(null, cr);
    content = cr.getContent();
    return content;
}
```

方法有点长，但逻辑很简单，就是审定了配置的获取有一个优先级，目的是为了容错。
