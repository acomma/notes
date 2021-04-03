刷新容器的第五步是调用 BeanFactory 后置处理器，`AbstractApplicationContext.invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory)`。

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}
```

这个方法将具体的处理逻辑交给了代理类 `PostProcessorRegistrationDelegate` 的 `invokeBeanFactoryPostProcessors` 方法，这个方法的代码有点多，但可以归纳一下。

1. 处理 `BeanDefinitionRegistryPostProcessor` 类型的后置处理器。
2. 处理 `BeanFactoryPostProcessor` 类型的后置处理器。

在处理的过程中先处理实现了 `PriorityOrdered` 接口的后置处理器，然后处理实现了 `Ordered` 接口的处理器，最后处理其他后置处理器。对每一类后置处理器都是先进行排序，然后依次调用每一个后置处理器。

完~
