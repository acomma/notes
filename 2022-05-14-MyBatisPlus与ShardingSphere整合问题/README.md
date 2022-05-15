## 前景提要

随着业务的发展需要在一个使用 mybatis-plus 的项目中引入分库分表的功能，经过比较选中了 shardingsphere，使用到的主要依赖及版本为

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.5.7</version>
    <relativePath/>
</parent>

<dependencies>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.5.1</version>
    </dependency>
    <dependency>
        <groupId>io.shardingsphere</groupId>
        <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
        <version>3.1.0</version>
    </dependency>
</dependencies>
```

项目的启动类非常简单

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

其他需要的配置均按照 mybatis-plus 和 shardingsphere 官方文档进行配置。

同时，为了把 springboot 的详细日志展示出来将 `org.springframework.boot` 的日志级别改为 `trace`

```yml
logging:
  level:
    org.springframework.boot: trace
```

## BeanDefinition 覆盖异常

遇到的第一个问题是 `BeanDefinitionOverrideException`，详细错误信息如下

```
java.lang.IllegalStateException: Failed to load ApplicationContext

	at org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate.loadContext(DefaultCacheAwareContextLoaderDelegate.java:132)
	at org.springframework.test.context.support.DefaultTestContext.getApplicationContext(DefaultTestContext.java:124)
	at org.springframework.test.context.web.ServletTestExecutionListener.setUpRequestContextIfNecessary(ServletTestExecutionListener.java:190)
	at org.springframework.test.context.web.ServletTestExecutionListener.prepareTestInstance(ServletTestExecutionListener.java:132)
	at org.springframework.test.context.TestContextManager.prepareTestInstance(TestContextManager.java:248)
	at org.springframework.test.context.junit.jupiter.SpringExtension.postProcessTestInstance(SpringExtension.java:138)
	// 省略了...
Caused by: org.springframework.beans.factory.support.BeanDefinitionOverrideException: Invalid bean definition with name 'dataSource' defined in class path resource [io/shardingsphere/shardingjdbc/spring/boot/SpringBootConfiguration.class]: Cannot register bean definition [Root bean: class [null]; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=io.shardingsphere.shardingjdbc.spring.boot.SpringBootConfiguration; factoryMethodName=dataSource; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [io/shardingsphere/shardingjdbc/spring/boot/SpringBootConfiguration.class]] for bean 'dataSource': There is already [Root bean: class [null]; scope=; abstract=false; lazyInit=null; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration$Hikari; factoryMethodName=dataSource; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Hikari.class]] bound.
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.registerBeanDefinition(DefaultListableBeanFactory.java:995)
	at org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod(ConfigurationClassBeanDefinitionReader.java:295)
	at org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForConfigurationClass(ConfigurationClassBeanDefinitionReader.java:153)
	at org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader.loadBeanDefinitions(ConfigurationClassBeanDefinitionReader.java:129)
	at org.springframework.context.annotation.ConfigurationClassPostProcessor.processConfigBeanDefinitions(ConfigurationClassPostProcessor.java:343)
	at org.springframework.context.annotation.ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(ConfigurationClassPostProcessor.java:247)
	at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(PostProcessorRegistrationDelegate.java:311)
	at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(PostProcessorRegistrationDelegate.java:112)
	at org.springframework.context.support.AbstractApplicationContext.invokeBeanFactoryPostProcessors(AbstractApplicationContext.java:746)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:564)
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:765)
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:445)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:338)
	at org.springframework.boot.test.context.SpringBootContextLoader.loadContext(SpringBootContextLoader.java:120)
	at org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate.loadContextInternal(DefaultCacheAwareContextLoaderDelegate.java:99)
	at org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate.loadContext(DefaultCacheAwareContextLoaderDelegate.java:124)
	... 67 more
```

这个异常信息是说已经存在了一个由 `o.s.b.a.j.DataSourceConfiguration.Hikari` 引入的名为 dataSource 的 BeanDefinition，不能再由 `i.s.s.s.b.SpringBootConfiguration` 再次引入名为 dataSource 的 BeanDefinition。

对允许自动配置（`@EnableAutoConfiguration`）的项目，经过 `AutoConfigurationSorter#getInPriorityOrder` 方法对自动配置类排序后 `DataSourceAutoConfiguration`、`MybatisPlusAutoConfiguration`、`SpringBootConfiguration` 的顺序为

1. org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
2. com.baomidou.mybatisplus.autoconfigure.MybatisPlusAutoConfiguration
3. io.shardingsphere.shardingjdbc.spring.boot.SpringBootConfiguration

出现在前面的自动配置类优先生效，所以 `DataSourceAutoConfiguration` 先导入了名为 dataSource 的 BeanDefinition，而在 `SpringBootConfiguration` 中 dataSource 的定义为

```java
// i.s.s.s.b.SpringBootConfiguration.java
@Bean
public DataSource dataSource() throws SQLException {
    // 省略了...
}
```

这个定义上面没有任何 `@Conditional` 注解，因此它不可以被跳过，而 springboot 又不允许重复的 BeanDefinition，所以上面的异常就抛出了。

解决这个问题有 6 个办法

1. 排除 `DataSourceAutoConfiguration`，即
    ```java
    @SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
    ```
2. 排除 `SpringBootConfiguration`，即
    ```java
    @SpringBootApplication(exclude = {SpringBootConfiguration.class})
    ```
3. 在 `SpringBootConfiguration` 类的 `dataSource()` 的方法上加上合适的 `@Conditional` 注解，比如 `@ConditionalOnMissingBean`
4. 因为 `DataSourceAutoConfiguration` 的 `dataSource()` 方法上有 `@ConditionalOnMissingBean` 注解
    ```java
    // o.s.b.a.j.DataSourceConfiguration.Hikari.java
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(HikariDataSource.class)
    @ConditionalOnMissingBean(DataSource.class)
    @ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource", matchIfMissing = true)
    static class Hikari {
        @Bean
        @ConfigurationProperties(prefix = "spring.datasource.hikari")
        HikariDataSource dataSource(DataSourceProperties properties) {
            // 省略了...
        }
    }
    ```
    因此使用 `@AutoConfigureBefore` 注解来提高 `i.s.s.s.b.SpringBootConfiguration` 的顺序
5. 和 4 反过来，使用 `@AutoConfigureAfter` 注解来降低 `DataSourceAutoConfiguration` 的顺序
6. 使用 `sharding-jdbc-core` 不使用 `sharding-jdbc-spring-boot-starter`

第 6 个办法因为需要自己去做一些额外的工作而没有那么方便，所以排除掉。在此基础上可以排除第 2 个和第 3 个办法，因为我们就是要使用 shardingsphere 的分表功能。修改 springboot 的代码并编译打包的复杂度高于修改 shardingsphere 的代码，因此排除第 5 个办法。有不修改 shardingsphere 代码的方法为啥要费心费力的去修改其他库的代码呢，编译打包发布都很麻烦，所以排除第 4 个方法，虽然最后选择的还是第 4 个方法，但那是有其他原因造成的。

最终这个问题的解决办法是排除 `DataSourceAutoConfiguration`，即

```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

以为这样就可以了吗？并不是！

## 找不到 sqlSessionFactory 或者 sqlSessionTemplate

解决了第 1 个问题，在把项目跑一遍看一看呢？控制台又是一片红色

```
java.lang.IllegalStateException: Failed to load ApplicationContext

	at org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate.loadContext(DefaultCacheAwareContextLoaderDelegate.java:132)
	at org.springframework.test.context.support.DefaultTestContext.getApplicationContext(DefaultTestContext.java:124)
	at org.springframework.test.context.web.ServletTestExecutionListener.setUpRequestContextIfNecessary(ServletTestExecutionListener.java:190)
	at org.springframework.test.context.web.ServletTestExecutionListener.prepareTestInstance(ServletTestExecutionListener.java:132)
	at org.springframework.test.context.TestContextManager.prepareTestInstance(TestContextManager.java:248)
	at org.springframework.test.context.junit.jupiter.SpringExtension.postProcessTestInstance(SpringExtension.java:138)
	// 省略了...
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'userService': Unsatisfied dependency expressed through field 'baseMapper'; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'userMapper' defined in file [E:\git\gitee\mybatis-plus-demo\target\classes\com\example\mybatis\plus\demo\mapper\UserMapper.class]: Invocation of init method failed; nested exception is java.lang.IllegalArgumentException: Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.resolveFieldValue(AutowiredAnnotationBeanPostProcessor.java:659)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:639)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:119)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessProperties(AutowiredAnnotationBeanPostProcessor.java:399)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1431)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:619)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542)
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:944)
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:918)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:583)
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:765)
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:445)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:338)
	at org.springframework.boot.test.context.SpringBootContextLoader.loadContext(SpringBootContextLoader.java:120)
	at org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate.loadContextInternal(DefaultCacheAwareContextLoaderDelegate.java:99)
	at org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate.loadContext(DefaultCacheAwareContextLoaderDelegate.java:124)
	... 67 more
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'userMapper' defined in file [E:\git\gitee\mybatis-plus-demo\target\classes\com\example\mybatis\plus\demo\mapper\UserMapper.class]: Invocation of init method failed; nested exception is java.lang.IllegalArgumentException: Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1804)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:620)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:542)
	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:335)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:234)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:333)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:208)
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:276)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1380)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1300)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.resolveFieldValue(AutowiredAnnotationBeanPostProcessor.java:656)
	... 86 more
Caused by: java.lang.IllegalArgumentException: Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required
	at org.springframework.util.Assert.notNull(Assert.java:201)
	at org.mybatis.spring.support.SqlSessionDaoSupport.checkDaoConfig(SqlSessionDaoSupport.java:122)
	at org.mybatis.spring.mapper.MapperFactoryBean.checkDaoConfig(MapperFactoryBean.java:73)
	at org.springframework.dao.support.DaoSupport.afterPropertiesSet(DaoSupport.java:44)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1863)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1800)
	... 96 more
```

这个问题是说在创建 `UserMapper` bean 时需要的 `sqlSessionFactory` 或者 `sqlSessionTemplate` 找不到。从控制台打印的 CONDITIONS EVALUATION REPORT 找到与 MybatisPlusAutoConfiguration 有关的部分

```
============================
CONDITIONS EVALUATION REPORT
============================


Positive matches:
-----------------

   // 省略了...


Negative matches:
-----------------

   // 省略了...

   MybatisPlusAutoConfiguration:
      Did not match:
         - @ConditionalOnSingleCandidate (types: javax.sql.DataSource; SearchStrategy: all) did not find any beans (OnBeanCondition)
      Matched:
         - @ConditionalOnClass found required classes 'org.apache.ibatis.session.SqlSessionFactory', 'org.mybatis.spring.SqlSessionFactoryBean' (OnClassCondition)

   // 省略了...


Exclusions:
-----------

    org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration


Unconditional classes:
----------------------

    // 省略了...

    io.shardingsphere.shardingjdbc.spring.boot.SpringBootConfiguration
```

发现 `MybatisPlusAutoConfiguration` 没有满足 `@ConditionalOnSingleCandidate(DataSource.class)` 这个条件。为什么呢？

排除了 `DataSourceAutoConfiguration` 后，`MybatisPlusAutoConfiguration` 依然排在 `SpringBootConfiguration` 之前

1. com.baomidou.mybatisplus.autoconfigure.MybatisPlusAutoConfiguration
2. io.shardingsphere.shardingjdbc.spring.boot.SpringBootConfiguration

在加载 `MybatisPlusAutoConfiguration` 的 BeanDefinition 时没有哪个自动配置类导入 dataSource 的 BeanDefinition，所以 `MybatisPlusAutoConfiguration` 没生效，从而在它里面定义的 `sqlSessionFactory` 和 `sqlSessionTemplate` 也没有被导入，又没有其他自动配置类或者由应用程序配置的 `sqlSessionFactory` 和 `sqlSessionTemplate` 生效，所以就报错了。

解决这个问题有 4 个方法

1. 使用 `@AutoConfigureBefore` 注解来提高 `i.s.s.s.b.SpringBootConfiguration` 的顺序，即
	```java
	@AutoConfigureBefore({MybatisPlusAutoConfiguration.class})
	public class SpringBootConfiguration implements EnvironmentAware {
		// 省略了...
	}
	```
2. 和 1 反过来，使用 `@AutoConfigureAfter` 注解来降低 `MybatisPlusAutoConfiguration` 的顺序
	```java
	@AutoConfigureAfter({SpringBootConfiguration.class})
	public class MybatisPlusAutoConfiguration implements InitializingBean {
		// 省略了...
	}
3. 由应用程序配置 `sqlSessionFactory` 和 `sqlSessionTemplate` 的 BeanDefinition
4. 引入 mybatis-spring-boot-starter

第 4 个方法引入 mybatis-spring-boot-starter 后各个自动配置类的顺序为

1. com.baomidou.mybatisplus.autoconfigure.MybatisPlusAutoConfiguration
2. io.shardingsphere.shardingjdbc.spring.boot.SpringBootConfiguration
3. org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration

这相当于放弃了使用 mybatis-plus，在没有使用 `BaseMapper` 的情况下是没问题的，比如 `public interface RoleMapper {}`，但是在使用了 `BaseMapper` 的情况下，比如 `public interface UserMapper extends BaseMapper<User> {}`，就会报错

```java
org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): com.example.mybatis.plus.demo.mapper.UserMapper.selectList

	at org.apache.ibatis.binding.MapperMethod$SqlCommand.<init>(MapperMethod.java:235)
	at org.apache.ibatis.binding.MapperMethod.<init>(MapperMethod.java:53)
	at org.apache.ibatis.binding.MapperProxy.lambda$cachedInvoker$0(MapperProxy.java:108)
	at java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:1660)
	at org.apache.ibatis.util.MapUtil.computeIfAbsent(MapUtil.java:35)
	at org.apache.ibatis.binding.MapperProxy.cachedInvoker(MapperProxy.java:95)
	at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:86)
	at com.sun.proxy.$Proxy65.selectList(Unknown Source)
	at com.example.mybatis.plus.demo.ApplicationTests.test1(ApplicationTests.java:32)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	// 省略了...
```

项目中又不得不使用 mybatis-plus，因此只能放弃第 4 个方法。

第 3 个方法是否有效没有进行测试，相当于要把 `MybatisPlusAutoConfiguration` 拷贝出来进行单独的修改。第 1 个方法和第 2 个方法的复杂度是相当的，因为已经改造过 `sharding-core` 的源码，所以修改 shardingsphere 的源码看起来更合理一点。

## 解决方案

修改 shardingsphere 源码，在 `i.s.s.s.b.SpringBootConfiguration` 类上加上 `@AutoConfigureBefore({DataSourceAutoConfiguration.class})`，即

```java
@Configuration
@EnableConfigurationProperties({
        SpringBootShardingRuleConfigurationProperties.class, SpringBootMasterSlaveRuleConfigurationProperties.class, 
        SpringBootConfigMapConfigurationProperties.class, SpringBootPropertiesConfigurationProperties.class
})
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
@RequiredArgsConstructor
public class SpringBootConfiguration implements EnvironmentAware {
	// 省略了...
}
```

这也是 sharding-jdbc-spring-boot-starter 4.1.1 的解决办法。另外启动类上也不用再排除 `DataSourceAutoConfiguration` 自动配置类。

遗憾的是在 shardingsphere 的仓库中已经找不到 sharding-jdbc-spring-boot-starter 3.1.0 的代码了，只能从 sharding-jdbc-spring-boot-starter-3.1.0-sources.jar 中把代码拷贝出来。

拷贝出来的代码无法下载 spring-boot-test-support 1.5.0.RELEASE

```
Could not find artifact org.springframework.boot:spring-boot-test-support:pom:1.5.0.RELEASE in alimaven (http://maven.aliyun.com/nexus/content/groups/public)
```

因此需要在 pom.xml 文件中加入如下配置

```xml
<properties>
    <spring-boot.version>1.5.22.RELEASE</spring-boot.version>
    <springframework.version>4.3.25.RELEASE</springframework.version>
</properties>
```

spring-boot.version 和 springframework.version 的值是配对的，即 1.5.22.RELEASE 的 spring-boot 使用的 springframework 版本就是 4.3.25.RELEASE。

在执行 mvn install 安装到本地时出现如下问题

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-gpg-plugin:1.6:sign (sign-artifacts) on project sharding-jdbc-spring-boot-starter: Unable to execute gpg command: Error while executing process. Cannot run program "gpg.exe": CreateProcess error=2, 系统找不到指定的文件。 -> [Help 1]
```

因此需要在 pom.xml 文件中加入如下配置

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-gpg-plugin</artifactId>
            <configuration>
                <skip>true</skip>
            </configuration>
        </plugin>
    </plugins>
</build>
```

也就是跳过 maven-gpg-plugin 的执行，因为我还不知道怎么使用 gpg，而且它是无关紧要的。

参考IDEA 使用maven编译，控制台乱码解决乱码问题，也就是在 File | Settings | Build, Execution, Deployment | Build Tools | Maven | Runner 中找到 VM Options，在它后面的输入框中输入“-Dfile.encoding=GBK”即可解决，当然这个小问题无关紧要，不影响编译结果。

完整的 pom.xml 如下

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.shardingsphere</groupId>
        <artifactId>sharding-jdbc-spring</artifactId>
        <version>3.1.0</version>
    </parent>
    <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
    <name>${project.artifactId}</name>

    <properties>
        <spring-boot.version>1.5.22.RELEASE</spring-boot.version>
        <springframework.version>4.3.25.RELEASE</springframework.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-gpg-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

完~
