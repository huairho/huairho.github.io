---
layout:     post
title:      Spring Bean 基础
subtitle:   Spring Bean 基础分析
date:       2021-05-16
author:     Huairho
header-img: img/spring-bean-base.jpg
catalog: true
tags:
    - Spring
---

# BeanDefinition

BeanDefinition 是 spring 定义 Bean 的配置元信息接口，包含

- Bean 的类名
    - 全类名
    - 必须为实现类，不可为抽象类或接口
- Bean 行为配置元素，如作用域、自动绑定模式、生命周期回调等
- 其他 Bean 引用，又可称作合作者（collaborators）或者依赖（dependencies）
- 配置设置，比如 Bean 属性（Properties）

BeanDefinition 并非是 Bean 的终态

[BeanDefinition 元信息](Spring%20Bean%20%E5%9F%BA%E7%A1%80%20afab693aa0ed4c1bb60fdf1541d00054/BeanDefinition%20%E5%85%83%E4%BF%A1%E6%81%AF%209f608a069a4b4855b4f7db0cf6d95041.csv)

## BeanDefinition 构建方式

- BeanDefinitionBuilder 方式
- AbstractBeanDefinition 以及派生类

## Demo

```java
public class BeanDefinitionDemo {

    public static void main(String[] args) {
        // 1. 通过 BeanDefinitionBuilder 构建 bean
        BeanDefinitionBuilder definitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(User.class);
        definitionBuilder
                .addPropertyValue("id", 3L)
                .addPropertyValue("name", "definition");
        // 获取 BeanDefinition 实例，该实例并非 bean 的终态
        BeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();

        // 2. 通过 AbstractBeanDefinition 构建 bean
        GenericBeanDefinition genericBeanDefinition = new GenericBeanDefinition();
        genericBeanDefinition.setBeanClass(User.class);

        MutablePropertyValues propertyValues = new MutablePropertyValues();
//        propertyValues.addPropertyValue("id", 3L);
//        propertyValues.addPropertyValue("name", "definition");
        propertyValues.add("id", 3L)
                .add("name", "definition");
        genericBeanDefinition.setPropertyValues(propertyValues);
    }
}
```

# 命名 Spring Bean

- 每个 Bean 拥有一个或多个标识符（identifiers），这些标识符在 Bean 所在的容器必须是唯一的。通常，一个 Bean 仅有一个标识符，如果需要额外的，可考虑使用别名（Alias）来扩充。
- 在基于 XML 的配置元信息中，开发人员可用 id 或者 name 属性来规定 Bean 的 标识符。通常Bean 的 标识符由字母组成，允许出现特殊字符。如果要想引入 Bean 的别名的话，可在name 属性使用半角逗号（“,”）或分号（“;”) 来间隔。
- Bean 的 id 或 name 属性并非必须制定，如果留空的话，容器会为 Bean 自动生成一个唯一的名称。Bean 的命名尽管没有限制，不过官方建议采用驼峰的方式，更符合 Java 的命名约定。

## Bean 名称的生成方式

- Bean 名称生成器（BeanNameGenerator）
- 由 Spring Framework 2.0.3 引入，框架內建两种实现：
    - DefaultBeanNameGenerator：默认通用 BeanNameGenerator 实现
    - AnnotationBeanNameGenerator：基于注解扫描的 BeanNameGenerator 实现

## 别名

- 复用现有的 BeanDefinition
- 更具有场景化的命名方法，比如：

    ```xml
    <alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
    <alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
    ```

# Bean 的注册

- XMl 配置元信息

```xml
<bean id ="user" class="com.xxx.User"/>
```

- Java 注解配置元信息
    - @Component
    - @Bean
    - @Import
- Java API 配置元信息
    - 命名方式：**BeanDefinitionRegistry.registerBeanDefinition(String, BeanDefinition)**
    - 非命名方式：**BeanDefinitionReaderUtils#registerWithGeneratedName(AbstractBeanDefinition,BeanDefinitionRegistry)**
    - 配置类方式：**AnnotatedBeanDefinitionReader#register(Class...)**
- 外部单例对象注册
    - SingletonBeanRegistry#registerSingleton

## Demo

```java
@Import(AnnotationBeanDefinitionDemo.Config.class)
public class AnnotationBeanDefinitionDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        // 注册配置类
        applicationContext.register(AnnotationBeanDefinitionDemo.class);
        // 启动上下文
        applicationContext.refresh();

        // 通过 BeanDefinition API 实现注册 bean
        // 通过命名方式注册
        registerBeanDefinition(applicationContext,"wanglaosan-user", User.class);
        // 通过非命名方式注册
        registerBeanDefinition(applicationContext, User.class);

        System.out.println("Config bean " + applicationContext.getBeansOfType(Config.class));
        System.out.println("User bean " + applicationContext.getBeansOfType(User.class));
    }

    /**
     * 通过命名方式注册
     * @param registry
     * @param beanName
     * @param clazz
     */
    public static void registerBeanDefinition(BeanDefinitionRegistry registry, String beanName, Class<?> clazz) {
        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(clazz);
        beanDefinitionBuilder.addPropertyValue("id", 33L).addPropertyValue("name", "wanglaosan");
        if (StringUtils.hasText(beanName)) {
            registry.registerBeanDefinition(beanName, beanDefinitionBuilder.getBeanDefinition());
        } else {
            BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinitionBuilder.getBeanDefinition(), registry);
        }
    }

    public static void registerBeanDefinition(BeanDefinitionRegistry registry, Class<?> clazz) {
        registerBeanDefinition(registry, null, clazz);
    }

    // 通过 @Component 方式
    @Component
    public static class Config {
        // 通过 bean 定义的方式
        @Bean(name = {"user", "arno-user"})
        public User user() {
            User user = new User();
            user.setId(333L);
            user.setName("arno");
            return user;
        }
    }
}
```

# Bean 实例化

- 常规方式
    - 通过构造器实例化（配置元信息：XML、注解或 Java API）
    - 通过静态工厂方法（配置元信息：XML、Java API）
    - 通过 Bean 工厂方法（配置元信息：XML、Java API）
    - 通过 FactoryBean （配置元信息：XML、Java 注解 、Java API）
- 特殊方式
    - 通过 ServiceLoaderFactoryBean （配置元信息：XML、Java 注解、Java API）
    - 通过 AutowireCapableBeanFactory#createBean(java.lang.Class, int, boolean)
    - 通过 BeanDefinitionRegistry#registerBeanDefinition(String,BeanDefinition)

## 常规方式

### Basic Class

```java
/**
 * 通过 bean 的方法创建实例
 */
public interface UserFactory {

    default User createUser () {
        User user = new User();
        user.setId(321L);
        user.setName("三哥");
        return user;
    }

    void initUserFactory();
    void destroyUserFactory();

}

public class UserFactoryImpl implements UserFactory {
    @Override
    public User createUser() {
        User user = new User();
        user.setId(99L);
        user.setName("UserFactoryImpl");
        return user;
    }

    @Override
    public void initUserFactory() {

    }

    @Override
    public void destroyUserFactory() {

    }

    public void initUserFactoryByBeanDefinition() {

    }
}

/**
 * 
 */
public class DefaultUserFactory implements UserFactory, InitializingBean, DisposableBean {

    @PostConstruct
    public void init() {
        System.out.println("@PostConstruct init DefaultUserFactory");
    }

    public void initUserFactory() {
        System.out.println("自定义 init DefaultUserFactory");
    }

    public void initUserFactoryByBeanDefinition() {
        System.out.println("BeanDefinition init DefaultUserFactory");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean init DefaultUserFactory");
    }

    @Override
    public void destroyUserFactory() {
        System.out.println("自定义 destroy DefaultUserFactory");
    }

    @PreDestroy
    public void doDestroy() {
        System.out.println("@PreDestroy destroy DefaultUserFactory");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("DisposableBean#destroy() destroy DefaultUserFactory");
    }
}

public class UserFactoryBean implements FactoryBean<User> {

    @Override
    public User getObject() throws Exception {
        return User.createUser();
    }

    @Override
    public Class<?> getObjectType() {
        return User.class;
    }
}

public class User implements BeanNameAware {
    private Long id;
    private String name;
    private City city;
    private Resource configFileResource;
    private City[] workCities;
    private List<City> lifeCities;
    private String beanName;

    public User() {
    }
		// 静态方法
    public static User createUser() {
        User user = new User();
        user.setId(321L);
        user.setName("王三想");
        return user;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public City getCity() {
        return city;
    }

    public void setCity(City city) {
        this.city = city;
    }

    public Resource getConfigFileResource() {
        return configFileResource;
    }

    public void setConfigFileResource(Resource configFileResource) {
        this.configFileResource = configFileResource;
    }

    public City[] getWorkCities() {
        return workCities;
    }

    public void setWorkCities(City[] workCities) {
        this.workCities = workCities;
    }

    public List<City> getLifeCities() {
        return lifeCities;
    }

    public void setLifeCities(List<City> lifeCities) {
        this.lifeCities = lifeCities;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", city=" + city +
                ", configFileResource=" + configFileResource +
                ", workCities=" + Arrays.toString(workCities) +
                ", lifeCities=" + lifeCities +
                '}';
    }

    @PostConstruct
    public void init() {
        System.out.println("user bean name 为 " + beanName +  " 对象初始化中...");
    }

    @PreDestroy
    public void destroy() {
        System.out.println("user bean name 为 " + beanName +  " 对象销毁中");
    }

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
    }
}
```

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    https://www.springframework.org/schema/beans/spring-beans.xsd">
    <!-- 通过 静态方法 构建 bean -->
    <bean id="static-method-user" class="com.arno.ioc.overview.domain.User"
          factory-method="createUser"/>

    <!-- 通过 实例（bean）方法 构建 bean -->
    <bean id="factory-method-user" factory-bean="userFactory" factory-method="createUser"/>
		<!-- 通过工厂方法创建 -->
    <bean id="userFactory" class="com.arno.bean.factory.DefaultUserFactory"/>

    <!-- 通过 FactoryBean 构建 bean -->
    <bean id="factory-bean-user" class="com.arno.bean.factory.UserFactoryBean"/>
</beans>
```

### Main

```java
public class BeanInstantiationDemo {
    public static void main(String[] args) {
        BeanFactory beanFactory = new ClassPathXmlApplicationContext("classpath:/META-INF/bean-instantiation-context.xml");
        User user = beanFactory.getBean("static-method-user", User.class);
        User userByInstantMethod = beanFactory.getBean("factory-method-user", User.class);
        User userByFactoryBean = beanFactory.getBean("factory-bean-user", User.class);

        System.out.println(user);
        System.out.println(userByInstantMethod);
        System.out.println(userByFactoryBean);

        System.out.println(user == userByInstantMethod);
        System.out.println(user == userByFactoryBean);
    }
}
```

## 特殊方式创建

```yaml
# META-INF/services/com.arno.bean.factory.UserFactory
com.arno.bean.factory.DefaultUserFactory
com.arno.bean.factory.DefaultUserFactory
com.arno.bean.factory.DefaultUserFactory
com.arno.bean.factory.DefaultUserFactory
com.arno.bean.factory.UserFactoryImpl
com.arno.bean.factory.UserFactoryImpl
com.arno.bean.factory.UserFactoryImpl
```

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userServiceLoaderFactoryBean" class="org.springframework.beans.factory.serviceloader.ServiceLoaderFactoryBean">
        <property name="serviceType" value="com.arno.bean.factory.UserFactory"/>
    </bean>
</beans>
```

### Main

```java
public class SpecialBeanInstantiationDemo {
    public static void main(String[] args) {
        ApplicationContext beanFactory = new ClassPathXmlApplicationContext("classpath:/META-INF/special-bean-instantiation-context.xml");

        jdkServiceLoadDemo();

        // 通过 ServiceLoaderFactoryBean 创建 UserFactory bean
        ServiceLoader<UserFactory> serviceLoader = beanFactory.getBean("userServiceLoaderFactoryBean", ServiceLoader.class);
        displayServiceLoader(serviceLoader);

        // 通过 AutowireCapableBeanFactory 创建 UserFactory bean
        AutowireCapableBeanFactory capableBeanFactory = beanFactory.getAutowireCapableBeanFactory();
        UserFactory userFactory = capableBeanFactory.createBean(DefaultUserFactory.class);
        System.out.println(userFactory.createUser());
    }

    private static void jdkServiceLoadDemo() {
        ServiceLoader<UserFactory> serviceLoader = load(UserFactory.class, Thread.currentThread().getContextClassLoader());
        displayServiceLoader(serviceLoader);
    }

    private static void displayServiceLoader(ServiceLoader<UserFactory> serviceLoader) {
        for (UserFactory userFactory : serviceLoader) {
            System.out.println(userFactory.createUser());
        }
    }
}
```

# Bean 初始化

## 初始化方式

- @PostConstruct 标注方法
- 实现 InitializingBean 接口的 afterPropertiesSet() 方法
- 自定义初始化方法
    - XML 配置：<bean init-method=”init” ... />
    - Java 注解：@Bean(initMethod=”init”)
    - Java API：AbstractBeanDefinition#setInitMethodName(String)

Q：**以上方式执行是什么顺序？**

A：@PostConstruct  > InitializingBean > 自定义方法

## 延迟初始化方式

- XML 配置：<bean lazy-init=”true” ... />
- Java 注解：@Lazy(true)

Q：**延迟初始化对象与非延迟的对象存在怎样的差异？**

A：实时初始化实在上下文启动前初始化，延迟初始化实在上下文启动之后

## Demo

```java
@Configuration // configuration class
public class BeanInitializationDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(BeanInitializationDemo.class);
        // 启动 spring 应用上下文
        applicationContext.refresh();
        System.out.println("Spring 应用上下文启动成功...");

//        BeanDefinitionBuilder definitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(UserFactory.class);
//        definitionBuilder.setInitMethodName("initUserFactoryByBeanDefinition");
//        applicationContext.registerBean(DefaultUserFactory.class, definitionBuilder.getBeanDefinition());

        UserFactory userFactory = applicationContext.getBean(UserFactory.class);
        System.out.println(userFactory);
        // 关闭 Spring 应用上下文
        System.out.println("Spring 应用上下文准备关闭...");
        applicationContext.close();
        System.out.println("Spring 应用上下文准备成功...");
    }

    @Bean(initMethod = "initUserFactory", destroyMethod = "destroyUserFactory")
    @Lazy
    public UserFactory userFactory() {
        return new DefaultUserFactory();
    }

}
```

# Bean 的销毁

## 销毁方式

- @PreDestroy 标注方法
- 实现 DisposableBean 接口的 destroy() 方法
- 自定义销毁方法
    - XML 配置：<bean destroy=”destroy” ... />
    - Java 注解：@Bean(destroy=”destroy”)
    - Java API：AbstractBeanDefinition#setDestroyMethodName(String)

执行顺序：@PreDestroy > DisposableBean > 自定义销毁方法

# Bean GC 阶段

1. 关闭 Spring 容器（应用上下文）
2. 执行 GC
3. Spring Bean 覆盖的 finalize() 方法被回调， finalize()方法不一定每次都调用，非稳定使用

# 小结

## Spring Bean 注册

- 通过 BeanDefinition 和外部单体对象来注册

```java
// 外部单体对象注册 demo
public class SingletonBeanRegistryDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        UserFactory userFactory = new DefaultUserFactory();
        // 注册外部单例对象 ConfigurableListableBeanFactory or SingletonBeanRegistry
        ConfigurableListableBeanFactory listableBeanFactory = applicationContext.getBeanFactory();
        listableBeanFactory.registerSingleton("userFactory", userFactory);
//        SingletonBeanRegistry singletonBeanRegistry = applicationContext.getBeanFactory();
//        singletonBeanRegistry.registerSingleton("userFactory", userFactory);
        applicationContext.refresh();
//        UserFactory userFactoryByLookUp = applicationContext.getBean("userFactory", UserFactory.class);
        UserFactory userFactoryByLookUp2 = listableBeanFactory.getBean("userFactory", UserFactory.class);
//        System.out.println("userFactory == userFactoryByLookUp : " + (userFactory == userFactoryByLookUp));
        System.out.println("userFactory == userFactoryByLookUp2 : " + (userFactory == userFactoryByLookUp2));
        applicationContext.close();
    }

}
```