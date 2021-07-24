---
layout:     post
title:      Spring IOC 依赖注入一
subtitle:   Spring IOC 依赖注入讲解一
date:       2020-05-17
author:     Huairho
header-img: img/spring-bean-base.jpg
catalog: true
tags:
    - Spring IOC 篇
---

# 注入模式和类型

- 手动模式，配置或者编程的方式，提前安排注入规则
  - XML 资源配置元信息
  - Java 注解配置元信息
  - API 配置元信息
- 自动模式，实现方提供依赖自动关联的方式，按照內建的注入规则
  - Autowiring（自动绑定）

## 依赖注入类型

- Setter 注入
  - `<proeprty name="user" ref="userBean"/>`
- 构造器注入
  - `<constructor-arg name="user" ref="userBean" />`
- 字段注入
  - `@Autowired User user;`
- 方法注入
  - `@Autowired public void user(User user) { ... }`
- 接口回调
  - `class MyBean implements BeanFactoryAware { ... }`

# 自动绑定

## 定义说明

The Spring container can autowired relationships between collaborating beans. You can let Spring resolve collaborators (other beans) automatically for your bean by inspecting the contents of the ApplicationContext.

## 优点

- Autowiring can significantly reduce the need to specify properties or constructor arguments.
- Autowiring can update a configuration as your objects evolve.

## 自动绑定模式

- no
  - 默认值，未激活 Autowiring，需要手动指定依赖注入对象
- byName
  - 根据被注入属性的名称作为 Bean 名称进行依赖查找，并将对象设置到该属性
- byType
  - 根据被注入属性的类型作为依赖类型进行查找，并将对象设置到该属性
- constructor
  - 特殊 byType 类型，用于构造器参数

`**org.springframework.beans.factory.annotation.Autowire`**

```
**org.springframework.beans.factory.config.AutowireCapableBeanFactory**
```

## 自动绑定局限性

- Explicit dependencies in `property` and `constructor-arg` settings always override autowiring. You cannot autowire simple properties such as primitives, `Strings`, and `Classes` (and arrays of such simple properties). This limitation is by-design.
- Autowiring is less exact than explicit wiring. Although, as noted in the earlier table, Spring is careful to avoid guessing in case of ambiguity that might have unexpected results. The relationships between your Spring-managed objects are no longer documented explicitly.
- Wiring information may not be available to tools that may generate documentation from a Spring container.
- Multiple bean definitions within the container may match the type specified by the setter method or constructor argument to be autowired. For arrays, collections, or `Map` instances, this is not necessarily a problem. However, for dependencies that expect a single value, this ambiguity is not arbitrarily resolved. If no unique bean definition is available, an exception is thrown.
- 属性和构造函数参数设置中的显式依赖项总是重写自动连接。不能自动装配简单属性，如原语、字符串和类(以及这种简单属性的数组)。这种限制是有意为之。
- 自动绑定不如直接绑定精确。尽管如前面的表所示，Spring 在出现可能产生意外结果的模糊性时会小心避免猜测。Spring 管理的对象之间的关系不再被显式记录。
- 对于可能从 Spring 容器生成文档的工具，连接信息可能不可用。
- 容器中的多个 bean 定义可能与 setter 方法或要自动连接的构造函数参数指定的类型相匹配。对于数组、集合或 Map 实例，这不一定是问题。但是，对于期望单个值的依赖项，不能随意解决这种不确定性。如果没有唯一的 bean 定义可用，则引发异常

# Setter 方法注入

## 手动模式

- XML 配置元信息

```xml
<bean class="org.geekbang.thinking.in.spring.ioc.dependency.injection.UserHolder">
	 <property name="user" ref="superUser" />
</bean>
```

- Java 注解配置元信息

```xml
<beans xmlns="<http://www.springframework.org/schema/beans>"
       xmlns:xsi="<http://www.w3.org/2001/XMLSchema-instance>"
       xsi:schemaLocation="<http://www.springframework.org/schema/beans>
    <https://www.springframework.org/schema/beans/spring-beans.xsd>">

    <bean id="user" class="com.arno.ioc.overview.domain.User">
        <property name="id" value="1"/>
        <property name="name" value="哇啦"/>
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
public class AnnotationDependencySettingInjectionDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(AnnotationDependencySettingInjectionDemo.class);

        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(applicationContext);
        String xmlPath = "classpath:/META-INF/dependency-lookup-context.xml";
        reader.loadBeanDefinitions(xmlPath);

        applicationContext.refresh();

        UserHolder userHolder = applicationContext.getBean(UserHolder.class);
        System.out.println(userHolder);

        applicationContext.close();
    }

    @Bean
    public UserHolder userHolder(User user) {
        UserHolder userHolder = new UserHolder();
        userHolder.setUser(user);
        return userHolder;
    }
}
```

- API 配置元信息

```java
public class ApiDependencySettingInjectionDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(ApiDependencySettingInjectionDemo.class);

				// 生成 userHolder 的 BeanDefinition，并注册
        BeanDefinition beanDefinition = createUserHolderBeanDefinition();
        applicationContext.registerBeanDefinition("userHolder", beanDefinition);

        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(applicationContext);
        String xmlPath = "classpath:/META-INF/dependency-lookup-context.xml";
        reader.loadBeanDefinitions(xmlPath);

        applicationContext.refresh();
        System.out.println(applicationContext.getBean(UserHolder.class));
        applicationContext.close();
    }

    private static BeanDefinition createUserHolderBeanDefinition() {
				// 通过 api 配置元信息
        BeanDefinitionBuilder definitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(UserHolder.class);
        definitionBuilder.addPropertyReference("user", "superUser");
        return definitionBuilder.getBeanDefinition();
    }
}
```

## 自动模式

- byName

```xml
<bean class="com.arno.spring.dependency.injection.UserHolder" autowire="byName"/>
```

- byType

```xml
<bean class="com.arno.spring.dependency.injection.UserHolder" autowire="byType"/>
```

# 构造器注入

## 手动模式

- XML 资源配置元信息

```xml
<bean class="org.geekbang.thinking.in.spring.ioc.dependency.injection.UserHolder">
        <constructor-arg name="user" ref="superUser" />
</bean>
```

- Java 注解配置元信息

```java
public class AnnotationDependencySettingInjectionDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(AnnotationDependencySettingInjectionDemo.class);

        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(applicationContext);
        String xmlPath = "classpath:/META-INF/dependency-lookup-context.xml";
        reader.loadBeanDefinitions(xmlPath);

        applicationContext.refresh();

        UserHolder userHolder = applicationContext.getBean(UserHolder.class);
        System.out.println(userHolder);

        applicationContext.close();
    }

    @Bean
    public UserHolder userHolder(User user) {
        return new UserHolder(user);
    }
}
```

- API 配置元信息

```java
public class ApiDependencyConstructorInjectionDemo {

    public static void main(String[] args) {

        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();

        // 生成 UserHolder 的 BeanDefinition
        BeanDefinition userHolderBeanDefinition = createUserHolderBeanDefinition();
        // 注册 UserHolder 的 BeanDefinition
        applicationContext.registerBeanDefinition("userHolder", userHolderBeanDefinition);

        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(applicationContext);

        String xmlResourcePath = "classpath:/META-INF/dependency-lookup-context.xml";
        // 加载 XML 资源，解析并且生成 BeanDefinition
        beanDefinitionReader.loadBeanDefinitions(xmlResourcePath);

        // 启动 Spring 应用上下文
        applicationContext.refresh();

        // 依赖查找并且创建 Bean
        UserHolder userHolder = applicationContext.getBean(UserHolder.class);
        System.out.println(userHolder);

        // 显示地关闭 Spring 应用上下文
        applicationContext.close();
    }

    /**
     * 为 {@link UserHolder} 生成 {@link BeanDefinition}
     *
     * @return
     */
    private static BeanDefinition createUserHolderBeanDefinition() {
        BeanDefinitionBuilder definitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(UserHolder.class);
        definitionBuilder.addConstructorArgReference("superUser");
        return definitionBuilder.getBeanDefinition();
    }
```

## 自动模式

- constructor

```xml
<bean class="org.geekbang.thinking.in.spring.ioc.dependency.injection.UserHolder" autowire="constructor"/>
```

# 字段注入

- Java 注解配置元信息
  - @Autowired
  - @Resource
  - @Inject（可选）

```java
public class AnnotationDependencyFieldInjectionDemo {

    @Autowired
    private
//    static // @Autowired 会忽略掉静态字段
            UserHolder userHolder;

    @Resource
    private UserHolder userHolder2;

    public static void main(String[] args) {

        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        // 注册 Configuration Class（配置类） -> Spring Bean
        applicationContext.register(AnnotationDependencyFieldInjectionDemo.class);

        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(applicationContext);

        String xmlResourcePath = "classpath:/META-INF/dependency-lookup-context.xml";
        // 加载 XML 资源，解析并且生成 BeanDefinition
        beanDefinitionReader.loadBeanDefinitions(xmlResourcePath);

        // 启动 Spring 应用上下文
        applicationContext.refresh();

        // 依赖查找 AnnotationDependencyFieldInjectionDemo Bean
        AnnotationDependencyFieldInjectionDemo demo = applicationContext.getBean(AnnotationDependencyFieldInjectionDemo.class);

        // @Autowired 字段关联
        UserHolder userHolder = demo.userHolder;
        System.out.println(userHolder);
        System.out.println(demo.userHolder2);

        System.out.println(userHolder == demo.userHolder2);

        // 显示地关闭 Spring 应用上下文
        applicationContext.close();
    }

    @Bean
    public UserHolder userHolder(User user) {
        return new UserHolder(user);
    }
}
```

# 方法注入

- Java 注解配置元信息
  - @Autowired
  - @Resource
  - @Inject（可选）
  - @Bean

```java
public class AnnotationDependencyMethodInjectionDemo {

    private UserHolder userHolder;

    private UserHolder userHolder2;

    @Autowired
    public void init1(UserHolder userHolder) {
        this.userHolder = userHolder;
    }

    @Resource
    public void init2(UserHolder userHolder2) {
        this.userHolder2 = userHolder2;
    }

    @Bean
    public UserHolder userHolder(User user) {
        return new UserHolder(user);
    }

    public static void main(String[] args) {

        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        // 注册 Configuration Class（配置类） -> Spring Bean
        applicationContext.register(AnnotationDependencyMethodInjectionDemo.class);

        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(applicationContext);

        String xmlResourcePath = "classpath:/META-INF/dependency-lookup-context.xml";
        // 加载 XML 资源，解析并且生成 BeanDefinition
        beanDefinitionReader.loadBeanDefinitions(xmlResourcePath);

        // 启动 Spring 应用上下文
        applicationContext.refresh();

        // 依赖查找 AnnotationDependencyFieldInjectionDemo Bean
        AnnotationDependencyMethodInjectionDemo demo = applicationContext.getBean(AnnotationDependencyMethodInjectionDemo.class);

        // @Autowired 字段关联
        UserHolder userHolder = demo.userHolder;
        System.out.println(userHolder);
        System.out.println(demo.userHolder2);

        System.out.println(userHolder == demo.userHolder2);

        // 显示地关闭 Spring 应用上下文
        applicationContext.close();
    }

}
```

# 接口回调注入

## Aware 系列接口回调

```
**org.springframework.beans.factory.Aware**
```

**内建接口**

- BeanFactoryAware
  - 获取 IoC 容器 - BeanFactory
- ApplicationContextAware
  - 获取 Spring 应用上下文 - ApplicationContext 对象
- EnvironmentAware
  - 获取 Environment 对象
- ResourceLoaderAware
  - 获取资源加载器 对象 - ResourceLoader
- BeanClassLoaderAware
  - 获取加载当前 Bean Class 的 ClassLoader
- BeanNameAware
  - 获取当前 Bean 的名称
- MessageSourceAware
  - 获取 MessageSource 对象，用于 Spring 国际化
- ApplicationEventPublisherAware
  - 获取 ApplicationEventPublishAware 对象，用于 Spring 事件
- EmbeddedValueResolverAware
  - 获取 StringValueResolver 对象，用于占位符处理

```java
public class AwareInterfaceDependencyInjectionDemo implements BeanFactoryAware, ApplicationContextAware {

    private static BeanFactory beanFactory;

    private static ApplicationContext applicationContext;

    public static void main(String[] args) {

        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        // 注册 Configuration Class（配置类） -> Spring Bean
        context.register(AwareInterfaceDependencyInjectionDemo.class);

        // 启动 Spring 应用上下文
        context.refresh();

        System.out.println(beanFactory == context.getBeanFactory());
        System.out.println(applicationContext == context);

        // 显示地关闭 Spring 应用上下文
        context.close();
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        AwareInterfaceDependencyInjectionDemo.beanFactory = beanFactory;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        AwareInterfaceDependencyInjectionDemo.applicationContext = applicationContext;
    }
}
```

# 注入选型

- 低依赖：构造器注入
  - 构造器参数少时选用
- 多依赖：Setter 方法注入
  - 构造器参数过多时选用
  - 如果内部属性存在依赖关系，或有顺序要求，不建议 Setter 注入
- 便利性：字段注入
  - @Autowired 注解
- 声明类：方法注入
  - @Bean 的方式使用

# 注入类型

## 基础参数注入

- 原生类型（Primitive）：boolean、byte、char、short、int、float、long、double
- 标量类型（Scalar）：Number、Character、Boolean、Enum、Locale、Charset、Currency、Properties、UUID
- 常规类型（General）：Object、String、TimeZone、Calendar、Optional 等
- Spring 类型：Resource、InputSource、Formatter 等

## 集合类型注入

- 数组类型（Array）：原生类型、标量类型、常规类型、Spring 类型
- 集合类型（Collection）
  - Collection：List、Set（SortedSet、NavigableSet、EnumSet）
  - Map：Properties

## Demo

```xml
<bean id="user" class="org.geekbang.thinking.in.spring.ioc.overview.domain.User">
        <property name="id" value="1"/>
        <property name="name" value="小马哥"/>
        <property name="city" value="HANGZHOU"/>
        <property name="workCities" value="BEIJING,HANGZHOU"/>
        <property name="lifeCities">
            <list>
                <value>BEIJING</value>
                <value>SHANGHAI</value>
            </list>
        </property>
        <property name="configFileLocation" value="classpath:/META-INF/user-config.properties"/>
    </bean>
```

## 其他说明

`java.lang.Class` ,数组 Array 中的类型可通过 `getComponentType` 方法获取，而集合 Collection 存在泛型擦写的处理，存储的都会变成 Object 类型。

```java
/**
     * Returns the {@code Class} representing the component type of an
     * array.  If this class does not represent an array class this method
     * returns null.
     *
     * @return the {@code Class} representing the component type of this
     * class if this class is an array
     * @see     java.lang.reflect.Array
     * @since JDK1.1
     */
    public native Class<?> getComponentType();
```

# 限定注入

## 使用注解 @Qualifier 限定

- 通过 Bean 名称限定
- 通过分组限定

## 基于注解 @Qualifier 扩展限定

- 自定义注解 - 如 Spring Cloud @LoadBalanced

## Demo

```java
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Qualifier
public @interface UserGroup {
}

public class QualifierInjectionDemo {

    @Autowired // supperUser 的 primary 为 true 所有此处注入为 supperUser
    private User supperUser;

    @Autowired
    @Qualifier("user")
    private User user;

    @Autowired // 无 Qualifier 的一组用户
    private Map<String, User> nonQualifierUserMap;

    @Autowired // 有 Qualifier 的一组用户
    @Qualifier
    private Map<String, User> qualifierUserMap;

    @Autowired // 有 UserGroup 的一组用户
    @UserGroup
    private Map<String, User> userGroupUserMap;

    @MyAutowired
    private Map<String, User> myAutowiredUserMap;

    @MyInject
    private Map<String, User> myInjectUserMap;

    @Bean
    @Qualifier // 初始化 bean 时 将 bean 分组为带 Qualifier 和不带 Qualifier的
    public User user1() {
        return creatUser(7L);
    }

    @Bean
    @Qualifier // 初始化 bean 时 将 bean 分组为带 Qualifier 和不带 Qualifier的
    public User user2() {
        return creatUser(8L);
    }

    @Bean
    @UserGroup // 初始化 bean 时 将 bean 分组为 userGroup
    public User user3() {
        return creatUser(9L);
    }

    @Bean
    @UserGroup // 初始化 bean 时 将 bean 分组为 userGrou
    public User user4() {
        return creatUser(10L);
    }

    private static User creatUser(Long id) {
        User user = new User();
        user.setId(id);
        return user;
    }

    @Bean(AnnotationConfigUtils.AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)
    // 替换 spring 自己声明名称为AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME
    // 的 AutowiredAnnotationBeanPostProcessor bean
    // static 可将当前 Bean 提前加载
    public static AutowiredAnnotationBeanPostProcessor beanPostProcessor() {
        AutowiredAnnotationBeanPostProcessor postProcessor = new AutowiredAnnotationBeanPostProcessor();
//        Set<Class<? extends Annotation>> annotationTypes = new HashSet<>(asList(Autowired.class, Value.class, MyInject.class));
//        postProcessor.setAutowiredAnnotationTypes(annotationTypes);
        postProcessor.setAutowiredAnnotationType(MyInject.class);
        return postProcessor;
    }

//    @Bean
//    public static AutowiredAnnotationBeanPostProcessor beanPostProcessor2() {
//        AutowiredAnnotationBeanPostProcessor postProcessor = new AutowiredAnnotationBeanPostProcessor();
//        postProcessor.setAutowiredAnnotationType(MyInject.class);
//        return postProcessor;
//    }

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(QualifierInjectionDemo.class);

        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(applicationContext);
        String xmlPath = "classpath:/META-INF/dependency-lookup-context.xml";
        reader.loadBeanDefinitions(xmlPath);

        applicationContext.refresh();

        QualifierInjectionDemo demo = applicationContext.getBean(QualifierInjectionDemo.class);
        System.out.println("superUser :" + demo.supperUser);
        System.out.println("user :" + demo.user);
        System.out.println("nonQualifierUserMap :" + demo.nonQualifierUserMap);
        System.out.println("qualifierUserMap :" + demo.qualifierUserMap);
        System.out.println("userGroupUserMap :" + demo.userGroupUserMap);
        System.out.println("myAutowiredUserMap :" + demo.myAutowiredUserMap);
        System.out.println("myInjectUserMap :" + demo.myInjectUserMap);

        applicationContext.close();
    }

}
```

# 延迟依赖注入

## 使用 API ObjectFactory 延迟注入

- 单一类型
- 集合类型

## 使用 API ObjectProvider 延迟注入（推荐）

- 单一类型
- 集合类型

## Demo

```java
public class LazyAnnotationDependencyInjectionDemo {

    @Autowired
    @Qualifier("user")
    private User user; // 实时注入

    @Autowired
    private ObjectProvider<User> userObjectProvider; // 延迟注入

    @Autowired
    private ObjectFactory<Set<User>> usersObjectFactory;

    public static void main(String[] args) {

        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        // 注册 Configuration Class（配置类） -> Spring Bean
        applicationContext.register(LazyAnnotationDependencyInjectionDemo.class);

        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(applicationContext);

        String xmlResourcePath = "classpath:/META-INF/dependency-lookup-context.xml";
        // 加载 XML 资源，解析并且生成 BeanDefinition
        beanDefinitionReader.loadBeanDefinitions(xmlResourcePath);

        // 启动 Spring 应用上下文
        applicationContext.refresh();

        // 依赖查找 QualifierAnnotationDependencyInjectionDemo Bean
        LazyAnnotationDependencyInjectionDemo demo = applicationContext.getBean(LazyAnnotationDependencyInjectionDemo.class);

        // 期待输出 superUser Bean
        System.out.println("demo.user = " + demo.user);
        // 期待输出 superUser Bean
        System.out.println("demo.userObjectProvider = " + demo.userObjectProvider.getObject()); // 继承 ObjectFactory
        // 期待输出 superUser user Beans
        System.out.println("demo.usersObjectFactory = " + demo.usersObjectFactory.getObject());

        demo.userObjectProvider.forEach(System.out::println);

        // 显示地关闭 Spring 应用上下文
        applicationContext.close();
    }

}
```