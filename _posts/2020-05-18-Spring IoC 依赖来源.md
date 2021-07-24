---
layout:     post
title:      Spring IOC 依赖来源
subtitle:   Spring IOC 依赖来源
date:       2020-05-18
author:     Huairho
header-img: img/spring-bean-base.jpg
catalog: true
tags:
    - Spring IOC 篇
---

# 依赖查找来源

- Spring BeanDefinition
  - xml
  - @Bean 标记
  - BeanDefinitionBuilder
- 单例对象：api 实现

# Spring 内建 BeanDefinition

```
AnnotationConfigUtils#registerAnnotationConfigProcessors()
```

| **Bean名称**                                     | **Bean实例**                         | **使用场景**                                  |
| ------------------------------------------------ | ------------------------------------ | --------------------------------------------- |
| **CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME** | ConfigurationClassPostProcessor      | 处理spring配置类                              |
| **CONFIGURATION_BEAN_NAME_GENERATOR**            | AutowiredAnnotationBeanPostProcessor | 处理Autowired 和Value注解                     |
| **COMMON_ANNOTATION_PROCESSOR_BEAN_NAME**        | CommonAnnotationBeanPostProcessor    | 处理JSR-250注解，如PostConstruct              |
| **EVENT_LISTENER_PROCESSOR_BEAN_NAME**           | EventListenerMethodProcessor         | 处理EvernListener注解标注的spring事件监听方法 |

## **Spring 内建单例对象**

```
AbstractApplicationContext#prepareBeanFactory()
```

![内建单例](https://tva1.sinaimg.cn/large/008i3skNgy1gss38bv12oj31h00lctd3.jpg)

# **依赖注入来源**

- Spring BeanDefinition
  - xml
  - @Bean 标记
  - BeanDefinitionBuilder
- 单例对象：api 实现
- 非 Spring 容器管理对象

## **非 Spring 容器管理对象**

以下四个对象通过BeanFactroy.getBean()的依赖查找方式无法获取到，但可以通过@Autowired等注解注入。其中beanFactory != applicationContext, 其他三个相等。beanFactory = ApplicationContext.getBeanFactory()

```
 beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
 beanFactory.registerResolvableDependency(ResourceLoader.class, this);
 beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
 beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

## **来源说明**

| **来源**                 | **Spring Bean** **对象** | **生命周期** | **配置元信息** | **使用场景** |
| ------------------------ | ------------------------ | ------------ | -------------- | ------------ |
| **BeanDefinition**       | 是                       | 是           | 有             | 查找和注入   |
| **单体对象**             | 是                       | 否           | 无             | 查找和注入   |
| **ResolvableDependency** | 否                       | 否           | 无             | 注入         |

### **BeanDefinition 注册**

```
org.springframework.beans.factory.support.DefaultListableBeanFactory#registerBeanDefinition
```

### **单例对象注册**

```
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#registerSingleton
```

### **ResolvableDependency注册**

```
org.springframework.beans.factory.support.DefaultListableBeanFactory#registerResolvableDependency
public class ResolvableDependencyDemo {

    @Autowired
    private String value;

    @Autowired
    private User user;

    @PostConstruct
    public void init() {
        System.out.println(value);
        System.out.println(user);
    }

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(ResolvableDependencyDemo.class);
        applicationContext.addBeanFactoryPostProcessor(beanFactory -> {
            // 注册 Resolvable Dependency
            beanFactory.registerResolvableDependency(String.class, "hello world");
            // 注册 单例对象
            beanFactory.registerSingleton("user", User.createUser());
        });
        applicationContext.refresh();
    }

}
```

## 非常规依赖来源

- 非常规的外置对象
- 非spring托管
- 不能被spring 管理生命周期
- 无法实现延迟初始化
- 无法通过依赖查找

如 @Value 注解注入的外部配置

```java
@Configuration
@PropertySource(value = "/META-INF/default.properties", encoding = "UTF-8")
public class ExternalConfigDependencyDemo {

    @Value("${user.id:-222}")
    private String userId;

    @Value("${user.resource:/ccc/ddd}")
    private Resource resource;

    @Value("${user.name.cn:我}")
    private String name;

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(ExternalConfigDependencyDemo.class);
        applicationContext.refresh();
        ExternalConfigDependencyDemo demo = applicationContext.getBean(ExternalConfigDependencyDemo.class);
        System.out.println("demo.userId" + demo.userId);
        System.out.println("demo.resource" + demo.resource);
        System.out.println("demo.name" + demo.name);
        applicationContext.close();
    }
}
```

# 问题

- 注入和查找的依赖来源是否相同？
  - 否，依赖查找的来源仅限于 Spring BeanDefinition 以及单例对象，而依赖注入的来源还包括 Resolvable Dependency 以及@Value 所标注的外部化配置。
- 单例对象是否可在容器启动后注册？
  - 可以，单例对象的注册与 BeanDefinition 不同，BeanDefinition 会被`ConfigurableListableBeanFactory#freezeConfiguration()`方法影响，从而冻结注册，单例对象则没有这个限制。
  - 详见 **ResolvableDependency 注册**代码实现
- Spring 依赖注入的来源有哪些?
  - Spring BeanDefinition
  - 单例对象
  - Resolvable Dependency
  - @Value 外部化配置