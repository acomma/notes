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

按照 [Nacos Spring Boot 快速开始](https://nacos.io/zh-cn/docs/quick-start-spring-boot.html) 中启动配置管理部分操作，`nacos-config-spring-boot-starter` 使用的版本为 `0.2.10`。

`nacos-config-spring-boot-starter` 为项目引入了下面几个依赖

* `com.alibaba.nacos:nacos-client:2.0.2`：Nacos 的 [Java SDK](https://nacos.io/zh-cn/docs/sdk.html)，提供了与 Nacos 交互的原始 API
* `com.alibaba.nacos:nacos-spring-context:1.1.1`：[Nacos 与 Spring 集成支持](https://nacos.io/zh-cn/docs/nacos-spring.html)，可以在 Spring 项目中使用 Nacos
* `com.alibaba.boot:nacos-config-spring-boot-autoconfigure:0.2.10`：Nacos 自动配置
* `com.alibaba.boot:nacos-spring-boot-base:0.2.10`：提供 `NacosFailureAnalyzer`

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

总结一下，在当前这个示例项目中，未从 Nacos 服务器加载配置。
