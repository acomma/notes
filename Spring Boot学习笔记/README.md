[TOC]

## 准备工作

使用 [Spring Initializr](https://start.spring.io/) 生成一个 Web 应用项目，Spring Boot 的版本选择 2.3.1.RELEASE，打包方式选择 jar。项目的启动类的代码为

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

接下来从 `main` 方法开始分析 Spring Boot 帮我们做了些什么工作。

## 创建 `SpringApplication` 实例

`SpringApplication` 提供了多种启动 Spring 应用的方式，它们都殊途同归：创建一个 `SpringApplication` 实例，然后调用它的实例方法 `public ConfigurableApplicationContext run(String... args) {}` 真正启动 Spring 应用。

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 初始化资源加载器，这里为 null
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 初始化 primary bean sources，这里是 Application.class
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断 Web 应用类型
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 设置 ApplicationContextInitializer
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 设置 ApplicationListener
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 推断 main application class，这里是 Application.class
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

primary bean sources 可以有多个，这里和 main application class 指向同一个类是因为启动方式导致的巧合。

Web 应用类型有三个枚举值：`NONE`、`SERVLET`、`REACTIVE`，推断的过程是判断相应的类在 classpath 中是否存在。以 `SERVLET` 为例，它要求类 `Servlet` 和 `ConfigurableWebApplicationContext` 同时存在。

`ApplicationContextInitializer` 和 `ApplicationListener` 都是接口，它们的具体实现类是从 `META-INF/spring.factories` 文件读取并实例化的。

主应用类的推断用到了一个新的知识点，从堆栈轨迹中找到方法名 `main` 所在的类。

```java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```

## 创建并刷新 `ApplicationContext`
