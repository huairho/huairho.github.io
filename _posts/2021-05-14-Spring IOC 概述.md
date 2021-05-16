---
layout:     post
title:      Spring IOC 概述
subtitle:   Spring IOC 概述
date:       2021-05-14
author:     Huairho
header-img: img/spring-bean-base.jpg
catalog: true
tags:
    - Spring IOC 篇
---

# 依赖查询

- 根据 Bean 名称查找
    - 实时查找
    - 延迟查找
- 根据 Bean 类型查找
    - 单个查找
    - 集合查找
- 根据 Bean 名称 + 类型方式查找
- 根据 java 注解查找
    - 单个查找
    - 集合查找

# Demo

## xml 配置

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="user" class="com.arno.ioc.overview.domain.User">
        <property name="id" value="1"/>
        <property name="name" value="乌拉"/>
        <property name="city" value="CHONGQING"/>
        <property name="configFileResource" value="classpath:/META-INF/user-config.properties"/>
        <property name="lifeCities" value="HANGZHOU,CHONGQING"/>
        <property name="workCities" value="CHONGQING,GUANGZHOU"/>
    </bean>

    <bean id="superUser" class="com.arno.ioc.overview.domain.SuperUser" parent="user" primary="true">
        <property name="address" value="杭州"/>
    </bean>

    <bean id="objectFactory" class="org.springframework.beans.factory.config.ObjectFactoryCreatingFactoryBean">
        <property name="targetBeanName" value="user"/>
    </bean>

</beans>
```

## model

```java
public class User implements BeanNameAware {
    private Long id;
    private String name;
    private City city;
    private Resource configFileResource;
    private City[] workCities;
    private List<City> lifeCities;
 // getter
 // setter
 // toString
}

@Super
public class SuperUser extends User {
    private String address;

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    @Override
    public String toString() {
        return "SuperUser{" +
                "address='" + address + '\'' +
                "} " + super.toString();
    }
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Super {
}
```

## Main

```java
public class DependencyLookupDemo {

    public static void main(String[] args) {
        // 配置 xml
        // 启动 spring 应用上下文
        BeanFactory beanFactory = new ClassPathXmlApplicationContext("classpath:/META-INF/dependency-lookup-context.xml");
        // 根据名称实时查找
        lookupInRealTime(beanFactory);
        // 根据名称延迟查找
        lookupInLazyTime(beanFactory);
        // 根据类型查找单个 bean
        lookupByType(beanFactory);
        // 根据类型查找集合 bean
        lookupCollectionByType(beanFactory);
        // 根据注解查询
        lookupByAnnotationType(beanFactory);
    }

    @SuppressWarnings("unchecked")
    private static void lookupByAnnotationType(BeanFactory beanFactory) {
        if (beanFactory instanceof ListableBeanFactory) {
            ListableBeanFactory listableBeanFactory = (ListableBeanFactory) beanFactory;
            Map<String, User> userMap = (Map)listableBeanFactory.getBeansWithAnnotation(Super.class);
            System.out.println("查找 @Super 注解的所有对象：" + userMap);
        }
    }

    private static void lookupCollectionByType(BeanFactory beanFactory) {
        if (beanFactory instanceof ListableBeanFactory) {
            ListableBeanFactory listableBeanFactory = (ListableBeanFactory) beanFactory;
            Map<String, User> userMap = listableBeanFactory.getBeansOfType(User.class);
            System.out.println("查找指定类型的所有对象：" + userMap);
        }
    }

    private static void lookupByType(BeanFactory beanFactory) {
        User user = beanFactory.getBean(User.class);
        System.out.println("类型查找：" + user);
    }

    private static void lookupInRealTime(BeanFactory beanFactory) {
        User user = (User) beanFactory.getBean("user");
        System.out.println("实时查找：" + user);
    }

    private static void lookupInLazyTime(BeanFactory beanFactory) {
        ObjectFactory<User> objectFactory = (ObjectFactory<User>) beanFactory.getBean("objectFactory");
        User user = objectFactory.getObject();
        System.out.println("延迟查找：" + user);
    }

}
```

# 依赖注入

- 根据名称注入
- 根据 Bean 类型注入
    - 单个 Bean 注入
    - 集合注入
- 注入内建 Bean 对象
- 注入非 Bean 对象
- 注入类型
    - 延迟注入
    - 实时注入

## XML 配置

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/util https://www.springframework.org/schema/util/spring-util.xsd">

    <import resource="dependency-lookup-context.xml"/>

    <bean id="userRepository" class="com.arno.ioc.overview.repository.UserRepository" autowire="byType">

<!--        <property name="users">-->
<!--            <util:list>-->
<!--                <ref bean="superUser"/>-->
<!--                <ref bean="user"/>-->
<!--            </util:list>-->
<!--        </property>-->
    </bean>
</beans>
```

## Model

```java
public class UserRepository {

    private Collection<User> users;
    private BeanFactory beanFactory;
    private ObjectFactory<User> userFactory;
    private ObjectFactory<ApplicationContext> objectFactory;

    public BeanFactory getBeanFactory() {
        return beanFactory;
    }

    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    public Collection<User> getUsers() {
        return users;
    }

    public void setUsers(Collection<User> users) {
        this.users = users;
    }

    public ObjectFactory<User> getUserFactory() {
        return userFactory;
    }

    public void setUserFactory(ObjectFactory<User> userFactory) {
        this.userFactory = userFactory;
    }

    public ObjectFactory<ApplicationContext> getObjectFactory() {
        return objectFactory;
    }

    public void setObjectFactory(ObjectFactory<ApplicationContext> objectFactory) {
        this.objectFactory = objectFactory;
    }

    @Override
    public String toString() {
        return "UserRepository{" +
                "users=" + users +
                '}';
    }
}
```

## Main

```java
public static void main(String[] args) {
        // 配置 xml
        // 启动 spring 应用上下文
        BeanFactory beanFactory = new ClassPathXmlApplicationContext("classpath:/META-INF/dependency-injection-context.xml");

        // 来源1 为自定义的 bean
        UserRepository userRepository = beanFactory.getBean(UserRepository.class);
        System.out.println(userRepository);

        // 当前 beanFactory 和 userRepository 里面的 beanFactory 不一样
        System.out.println(beanFactory == userRepository.getBeanFactory());
        // 依赖注入 来源2 内建的依赖
        System.out.println(userRepository.getBeanFactory());
        System.out.println(beanFactory);
        // 依赖查找，报错无 BeanFactory 类型的 bean
//        System.out.println(beanFactory.getBean(BeanFactory.class));
        // 疑惑点: 依赖查找为什么查不到，而依赖注入可以输出？ 注入和查找的源不同？
        // 解答：bean的来源三个： 1. 自定义的 bean 2. 容器内建的 bean 3. 内建的依赖

        System.out.println(userRepository.getUserFactory());

        ObjectFactory objectFactory = userRepository.getObjectFactory();
        System.out.println(objectFactory.getObject());

        // 和当前的 beanFactory 一样
        System.out.println(beanFactory == objectFactory.getObject());

        // 来源3 容器内建的 bean 对象
        Environment environment = beanFactory.getBean(Environment.class);
        System.out.println("查找 environment 类型的 bean ：" + environment);

    }

}
```

# Spring IoC 依赖来源

- 自定义 Bean
    - 如上依赖注入的 UserRepository，为自定义 Bean
- 容器内建的 Bean
    - 如 Environment 为容器内建的 bean
- 容器内建依赖
    - userRepository.getBeanFactory()，为容器内建的依赖

# Spring IoC 配置元信息

- Bean 定义配置
    - 基于 XML 配置
    - 基于 Properties 配置
    - 基于 Java 注解
    - 基于 Java API
- IoC 容器配置
    - 基于 XML 配置
    - 基于 Java 注解
    - 基于 Java API
- 外部化属性配置
    - 基于 Java 注解

# BeanFactory 和 ApplicationContext 谁才是 Spring IoC 容器？

The `org.springframework.beans` and `org.springframework.context` packages are the basis for Spring Framework’s IoC container. The `[BeanFactory](https://docs.spring.io/spring-framework/docs/5.2.2.RELEASE/javadoc-api/org/springframework/beans/factory/BeanFactory.html)` interface provides an advanced configuration mechanism capable of managing any type of object. `[ApplicationContext](https://docs.spring.io/spring-framework/docs/5.2.2.RELEASE/javadoc-api/org/springframework/context/ApplicationContext.html)` is a sub-interface of `BeanFactory`. It adds:

- Easier integration with Spring’s AOP features
- Message resource handling (for use in internationalization)
- Event publication
- Application-layer specific contexts such as the `WebApplicationContext` for use in web applications.

In short, the `BeanFactory` provides the configuration framework and basic functionality, and the `ApplicationContext` adds more enterprise-specific functionality. The `ApplicationContext` is a complete superset of the `BeanFactory` and is used exclusively in this chapter in descriptions of Spring’s IoC container. For more information on using the `BeanFactory` instead of the `ApplicationContext`

- BeanFactory 是 spring 底层的 IoC 容器
- ApplicationContext 是具备应用特性的 BeanFactory 超集
    - 面向切面（AOP）
    - 配置元信息（Configuration Metadata）
    - 资源管理（Resources）
    - 事件（Events）
    - 国际化（i18n）
    - 注解（Annotations）
    - Environment 抽象（Environment Abstraction）

# 功能对比

![png](https://tva1.sinaimg.cn/large/008i3skNly1gqklnolb49j31i40p2tc0.jpg)

## BeanFactory Basic IoC

```java
public class BeanFactoryIoCContainerDemo {

    public static void main(String[] args) {
        // 创建 BeanFactory 容器
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

        // 加载配置文件
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
        String location = "classpath:/META-INF/dependency-lookup-context.xml";
        int definitionsCount = reader.loadBeanDefinitions(location);
        System.out.println("bean 加载的数量 ： " + definitionsCount);
				lookupByAnnotationType(beanFactory);

    }

    @SuppressWarnings("unchecked")
    private static void lookupByAnnotationType(BeanFactory beanFactory) {
        if (beanFactory instanceof ListableBeanFactory) {
            ListableBeanFactory listableBeanFactory = (ListableBeanFactory) beanFactory;
            Map<String, User> userMap = (Map)listableBeanFactory.getBeansWithAnnotation(Super.class);
            System.out.println("查找 @Super 注解的所有对象：" + userMap);
        }
    }
}
```

## ApplicationContext Basic IoC

```java
public class ApplicationContextIoCContainerDemo {

    public static void main(String[] args) {
        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        // 将当前类作为配置类
        applicationContext.register(ApplicationContextIoCContainerDemo.class);
        // 启动应用上下文
        applicationContext.refresh();
				lookupByAnnotationType(applicationContext);
				applicationContext.close();
    }

    @Bean
    public User user() {
        User user = new User();
        user.setId(333L);
        user.setName("arno");
        return user;
    }

    @SuppressWarnings("unchecked")
    private static void lookupByAnnotationType(BeanFactory beanFactory) {
        User user = (User) beanFactory.getBean("user");
        System.out.println("实时查找：" + user);
    }
}
```

# 小结

## BeanFactory And FactoryBean

- BeanFactory 是 IoC 底层容器
- FactoryBean 是 创建 Bean 的一种方式，帮助实现复杂的初始化逻辑
    - 比如对象通过 Builder 方式创建，无法通过反射直接创建
    - 创建的 Bean 不会被 IoC 管理生命周期