---
layout:     post
title:      Spring IOC 依赖查找
subtitle:   Spring IOC 依赖查找
date:       2020-05-17
author:     Huairho
header-img: img/spring-bean-base.jpg
catalog: true
tags:
    - Spring IOC 篇
---

# 前世今生

- 单一类型查找
  - JNDI - `javax.naming.Context#lookup(javax.naming.Name)`
  - JavaBeans - `java.beans.beancontext.BeanContext`
- 集合类型查找
  - `java.beans.beancontext.BeanContext`
- 层次性依赖查找
  - `java.beans.beancontext.BeanContext`

# 单一类型查找

## 查找接口 BeanFactory

- 根据 Bean 名称查找
  - getBean(String)
  - Spring 2.5 覆盖默认参数：getBean(String,Object...)
    - 返回原 Bean，并将传入的 Object 作为 Bean 覆盖原 Bean，因此不建议使用
- 根据 Bean 类型查找
  - Bean 实时查找
    - Spring 3.0 getBean(Class)
    - Spring 4.1 覆盖默认参数：getBean(Class,Object...)
      - 返回原 Bean，并将传入的 Object 作为 Bean 覆盖原 Bean，因此不建议使用
  - Spring 5.1 Bean 延迟查找
    - getBeanProvider(Class)
    - getBeanProvider(ResolvableType)
- 根据 Bean 名称 + 类型查找：getBean(String,Class)

## Demo

```java
public class ObjectProviderDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(ObjectProviderDemo.class);
        applicationContext.refresh();
        
        lookupByObjectProvider(applicationContext);
        lookupIfAvailable(applicationContext);
        lookupByStreamOps(applicationContext);
        
        applicationContext.close();
    }

    private static void lookupByStreamOps(AnnotationConfigApplicationContext applicationContext) {
        ObjectProvider<String> objectProvider = applicationContext.getBeanProvider(String.class);
//        for (String str  : objectProvider) {
//            System.out.println(str);
//        }
        objectProvider.stream().forEach(System.out::println);
    }

    private static void lookupIfAvailable(AnnotationConfigApplicationContext applicationContext) {
        ObjectProvider<User> userObjectProvider = applicationContext.getBeanProvider(User.class);
        User user = userObjectProvider.getIfAvailable(User::createUser);
        System.out.println(user);
    }

    @Bean
    @Primary
    public String helloWorld() {
        return "hello world";
    }

    @Bean
    public String message() {
        return "Message";
    }

    private static void lookupByObjectProvider(AnnotationConfigApplicationContext applicationContext) {
        ObjectProvider<String> objectProvider = applicationContext.getBeanProvider(String.class);
        System.out.println(objectProvider.getObject());
    }

}
```

# 集合类型查找

## 查找接口 ListableBeanFactory

- 根据 Bean 类型查找
  - 获取同类型 Bean 名称列表
    - `getBeanNamesForType(Class)`
    - Spring 4.2 `getBeanNamesForType(ResolvableType)`
  - 获取同类型 Bean 实例列表
    - `getBeansOfType(Class)` 以及重载方法
- 通过注解类型查找
  - Spring 3.0 获取标注类型 Bean 名称列表
    - `getBeanNamesForAnnotation(Class<? extends Annotation>)`
  - Spring 3.0 获取标注类型 Bean 实例列表
    - `getBeansWithAnnotation(Class<? extends Annotation>)`
  - Spring 3.0 获取指定名称 + 标注类型 Bean 实例
    - `findAnnotationOnBean(String,Class<? extends Annotation>)`

# 层次性查找

## 查找接口 HierarchicalBeanFactory

- 双亲 BeanFactory：getParentBeanFactory()
- 层次性查找
  - 根据 Bean 名称查找
    - 基于 `containsLocalBean` 方法实现
  - 根据 Bean 类型查找实例列表
    - 单一类型：BeanFactoryUtils#beanOfType()
    - 集合类型：BeanFactoryUtils#beansOfTypeIncludingAncestors()
  - 根据 Java 注解查找名称列表
    - BeanFactoryUtils#beanNamesForTypeIncludingAncestors()

## 说明

**传入 `bean name` 后，从子 `BeanFactory` 开始查找当前 `name` 的 `bean`，子 `BeanFactory` 不存在时则在父 `BeanFactory`。一直往上直到查询到当前 name 的 bean 为止。**

```java
public class HierarchicalDependencyLookupDemo {

    public static void main(String[] args) {
        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        // 将当前类 ObjectProviderDemo 作为配置类（Configuration Class）
        applicationContext.register(ObjectProviderDemo.class);

        // 1. 获取 HierarchicalBeanFactory <- ConfigurableBeanFactory <- ConfigurableListableBeanFactory
        ConfigurableListableBeanFactory beanFactory = applicationContext.getBeanFactory();
//        System.out.println("当前 BeanFactory 的 Parent BeanFactory ： " + beanFactory.getParentBeanFactory());

        // 2. 设置 Parent BeanFactory
        HierarchicalBeanFactory parentBeanFactory = createParentBeanFactory();
        beanFactory.setParentBeanFactory(parentBeanFactory);
//        System.out.println("当前 BeanFactory 的 Parent BeanFactory ： " + beanFactory.getParentBeanFactory());

        displayContainsLocalBean(beanFactory, "user");
        displayContainsLocalBean(parentBeanFactory, "user");

        displayContainsBean(beanFactory, "user");
        displayContainsBean(parentBeanFactory, "user");

        // 启动应用上下文
        applicationContext.refresh();

        // 关闭应用上下文
        applicationContext.close();

    }

    private static void displayContainsBean(HierarchicalBeanFactory beanFactory, String beanName) {
        System.out.printf("当前 BeanFactory[%s] 是否包含 Bean[name : %s] : %s\\n", beanFactory, beanName,
                containsBean(beanFactory, beanName));
    }

    private static boolean containsBean(HierarchicalBeanFactory beanFactory, String beanName) {
        BeanFactory parentBeanFactory = beanFactory.getParentBeanFactory();
        if (parentBeanFactory instanceof HierarchicalBeanFactory) {
            HierarchicalBeanFactory parentHierarchicalBeanFactory = HierarchicalBeanFactory.class.cast(parentBeanFactory);
            if (containsBean(parentHierarchicalBeanFactory, beanName)) {
                return true;
            }
        }
        return beanFactory.containsLocalBean(beanName);
    }

    private static void displayContainsLocalBean(HierarchicalBeanFactory beanFactory, String beanName) {
        System.out.printf("当前 BeanFactory[%s] 是否包含 Local Bean[name : %s] : %s\\n", beanFactory, beanName,
                beanFactory.containsLocalBean(beanName));
    }

    private static ConfigurableListableBeanFactory createParentBeanFactory() {
        // 创建 BeanFactory 容器
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
        // XML 配置文件 ClassPath 路径
        String location = "classpath:/META-INF/dependency-lookup-context.xml";
        // 加载配置
        reader.loadBeanDefinitions(location);
        return beanFactory;
    }

}
```

## Spring 层次性查找

```java
/**
	 * Return all beans of the given type or subtypes, also picking up beans defined in
	 * ancestor bean factories if the current bean factory is a HierarchicalBeanFactory.
	 * The returned Map will only contain beans of this type.
	 * <p>Does consider objects created by FactoryBeans if the "allowEagerInit" flag is set,
	 * which means that FactoryBeans will get initialized. If the object created by the
	 * FactoryBean doesn't match, the raw FactoryBean itself will be matched against the
	 * type. If "allowEagerInit" is not set, only raw FactoryBeans will be checked
	 * (which doesn't require initialization of each FactoryBean).
	 * <p><b>Note: Beans of the same name will take precedence at the 'lowest' factory level,
	 * i.e. such beans will be returned from the lowest factory that they are being found in,
	 * hiding corresponding beans in ancestor factories.</b> This feature allows for
	 * 'replacing' beans by explicitly choosing the same bean name in a child factory;
	 * the bean in the ancestor factory won't be visible then, not even for by-type lookups.
	 * @param lbf the bean factory
	 * @param type type of bean to match
	 * @param includeNonSingletons whether to include prototype or scoped beans too
	 * or just singletons (also applies to FactoryBeans)
	 * @param allowEagerInit whether to initialize <i>lazy-init singletons</i> and
	 * <i>objects created by FactoryBeans</i> (or by factory methods with a
	 * "factory-bean" reference) for the type check. Note that FactoryBeans need to be
	 * eagerly initialized to determine their type: So be aware that passing in "true"
	 * for this flag will initialize FactoryBeans and "factory-bean" references.
	 * @return the Map of matching bean instances, or an empty Map if none
	 * @throws BeansException if a bean could not be created
	 * @see ListableBeanFactory#getBeansOfType(Class, boolean, boolean)
	 */
	public static <T> Map<String, T> beansOfTypeIncludingAncestors(
			ListableBeanFactory lbf, Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
			throws BeansException {

		Assert.notNull(lbf, "ListableBeanFactory must not be null");
		Map<String, T> result = new LinkedHashMap<>(4);
		result.putAll(lbf.getBeansOfType(type, includeNonSingletons, allowEagerInit));
		// 如果为 HierarchicalBeanFactory 的子类
		if (lbf instanceof HierarchicalBeanFactory) {
			HierarchicalBeanFactory hbf = (HierarchicalBeanFactory) lbf;
      // 如果父 BeanFactory 为 ListableBeanFactory 的子类
			if (hbf.getParentBeanFactory() instanceof ListableBeanFactory) {
        // 递归调用 beansOfTypeIncludingAncestors(),依次往上查询 bean
				Map<String, T> parentResult = beansOfTypeIncludingAncestors(
						(ListableBeanFactory) hbf.getParentBeanFactory(), type, includeNonSingletons, allowEagerInit);
						parentResult.forEach((beanName, beanInstance) -> {
						if (!result.containsKey(beanName) && !hbf.containsLocalBean(beanName)) {
						result.put(beanName, beanInstance);
					}
				});
			}
		}
		return result;
	}
```

# 延迟查找

## 查找接口

- `org.springframework.beans.factory.ObjectFactory`

- ```
  org.springframework.beans.factory.ObjectProvider
  ```

  - Spring 5 对 Java 8 特性扩展
    - 函数式接口
      - getIfAvailable(Supplier)
      - ifAvailable(Consumer)
    - Stream 扩展 - stream()

## Demo

```java
public class ObjectProviderDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(ObjectProviderDemo.class);
        applicationContext.refresh();

lookupByObjectProvider(applicationContext);
lookupIfAvailable(applicationContext);
lookupByStreamOps(applicationContext);

        applicationContext.close();
    }

    private static void lookupByStreamOps(AnnotationConfigApplicationContext applicationContext) {
        ObjectProvider<String> objectProvider = applicationContext.getBeanProvider(String.class);
//        for (String str  : objectProvider) {
//            System.out.println(str);
//        }
        objectProvider.stream().forEach(System.out::println);
    }

    private static void lookupIfAvailable(AnnotationConfigApplicationContext applicationContext) {
        ObjectProvider<User> userObjectProvider = applicationContext.getBeanProvider(User.class);
        User user = userObjectProvider.getIfAvailable(User::createUser);
        System.out.println(user);
    }

    @Bean
    @Primary
    public String helloWorld() {
        return "hello world";
    }

    @Bean
    public String message() {
        return "Message";
    }

    private static void lookupByObjectProvider(AnnotationConfigApplicationContext applicationContext) {
        ObjectProvider<String> objectProvider = applicationContext.getBeanProvider(String.class);
        System.out.println(objectProvider.getObject());
    }

}
```

# 安全性依赖查找

## 依赖查找安全性说明

- 非安全查找定义
  - 当被查找的 Bean 不存在时会出现 NoSuchBeanDefinitionException
- 安全查找
  - 被查找的 Bean 不存在时返回空，而不是抛出异常

## 安全性对比

| **依赖查找类型** | **代表实现**                       | **是否安全** |
| ---------------- | ---------------------------------- | ------------ |
| **单一类型查找** | BeanFactory#getBean                | 否           |
|                  | ObjectFactory#getObject            | 否           |
|                  | ObjectProvider#getIfAvailable      | 是           |
| **集合类型查找** | ListableBeanFactory#getBeansOfType | 是           |
|                  | ObjectProvider#stream              | 是           |

## Demo

```java
public class TypeSafetyLookupDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(TypeSafetyLookupDemo.class);
        applicationContext.refresh();

        // 根据 BeanFactory#getBean() 查找 bean
        lookupByBeanFactoryGetBean(applicationContext);
        // 根据 ObjectFactory#getObject() 查找 bean
        lookupByObjectFactoryGetObject(applicationContext);
        // 根据 ObjectProvider#getIfAvailable() 查找 bean
        lookupByObjectProviderGetIfAvailable(applicationContext);
        // 根据 ListableBeanFactory#
        lookupByListableBeanFactoryGetBeansOfType(applicationContext);
        // 根据 ObjectProvider#stream() 查找 bean
        lookupByObjectProviderStreamOps(applicationContext);
        applicationContext.close();
    }

    private static void lookupByListableBeanFactoryGetBeansOfType(ListableBeanFactory applicationContext) {
        printException("lookupByListableBeanFactoryGetBeansOfType", () -> applicationContext.getBeansOfType(User.class));
    }

    private static void lookupByObjectProviderStreamOps(AnnotationConfigApplicationContext applicationContext) {
        ObjectProvider<User> userObjectProvider = applicationContext.getBeanProvider(User.class);
        printException("lookupByObjectProviderStreamOps", () -> userObjectProvider.stream().forEach(System.out :: println));
    }

    private static void lookupByObjectProviderGetIfAvailable(AnnotationConfigApplicationContext applicationContext) {
        ObjectProvider<User> userObjectProvider = applicationContext.getBeanProvider(User.class);
        printException("lookupByObjectProviderGetIfAvailable", () -> userObjectProvider.getIfAvailable());
    }

    private static void lookupByObjectFactoryGetObject(AnnotationConfigApplicationContext applicationContext) {
        ObjectFactory<User> userObjectFactory = applicationContext.getBeanProvider(User.class);
        printException("lookupByObjectFactoryGetObject", () -> userObjectFactory.getObject());
    }

    private static void lookupByBeanFactoryGetBean(BeanFactory beanFactory) {
        printException("lookupByBeanFactoryGetBean", () -> beanFactory.getBean(User.class));
    }

    private static void printException(String source, Runnable runnable) {
        System.err.println("===============================");
        System.err.println("source from : " + source);
        try {
            runnable.run();
        } catch (BeansException e) {
            e.printStackTrace();
        }
    }

}
```

**输出**结果

```
===============================
source from : lookupByBeanFactoryGetBean
org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'com.arno.ioc.overview.domain.User' available
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:351)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:342)
	at org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:1126)
	at com.arno.spring.dependency.lookup.TypeSafetyLookupDemo.lambda$lookupByBeanFactoryGetBean$4(TypeSafetyLookupDemo.java:57)
	at com.arno.spring.dependency.lookup.TypeSafetyLookupDemo.printException(TypeSafetyLookupDemo.java:65)
	at com.arno.spring.dependency.lookup.TypeSafetyLookupDemo.lookupByBeanFactoryGetBean(TypeSafetyLookupDemo.java:57)
	at com.arno.spring.dependency.lookup.TypeSafetyLookupDemo.main(TypeSafetyLookupDemo.java:25)
===============================
source from : lookupByObjectFactoryGetObject
org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'com.arno.ioc.overview.domain.User' available
	at org.springframework.beans.factory.support.DefaultListableBeanFactory$1.getObject(DefaultListableBeanFactory.java:370)
	at com.arno.spring.dependency.lookup.TypeSafetyLookupDemo.lambda$lookupByObjectFactoryGetObject$3(TypeSafetyLookupDemo.java:53)
	at com.arno.spring.dependency.lookup.TypeSafetyLookupDemo.printException(TypeSafetyLookupDemo.java:65)
	at com.arno.spring.dependency.lookup.TypeSafetyLookupDemo.lookupByObjectFactoryGetObject(TypeSafetyLookupDemo.java:53)
	at com.arno.spring.dependency.lookup.TypeSafetyLookupDemo.main(TypeSafetyLookupDemo.java:27)
===============================
source from : lookupByObjectProviderGetIfAvailable
===============================
source from : lookupByListableBeanFactoryGetBeansOfType
===============================
source from : lookupByObjectProviderStreamOps
```

# IoC 容器内建的 Bean 查找

## 哪些内建可查找

### AbstractApplicationContext 内建可查找的依赖

| **Bean** **名称**               | **Bean** **实例**                | **使用场景**            |
| ------------------------------- | -------------------------------- | ----------------------- |
| **environment**                 | Environment 对象                 | 外部化配置以及 Profiles |
| **systemProperties**            | java.util.Properties 对象        | Java 系统属性           |
| **systemEnvironment**           | java.util.Map 对象               | 操作系统环境变量        |
| **messageSource**               | MessageSource 对象               | 国际化文案              |
| **lifecycleProcessor**          | LifecycleProcessor 对象          | Lifecycle Bean 处理器   |
| **applicationEventMulticaster** | ApplicationEventMulticaster 对象 | Spring 事件广播器       |

### 注解驱动 Spring 应用上下文内建可查找的依赖（部分）

**详见 `AnnotationConfigUtils`**

| **Bean**                                                     | **Bean** **实例**                           | **使用场景**                                          |
| ------------------------------------------------------------ | ------------------------------------------- | ----------------------------------------------------- |
| **org.springframework.context.annotation.internalConfigurationAnnotationProcessor** | ConfigurationClassPostProcessor 对象        | 处理 Spring 配置类                                    |
| **org.springframework.context.annotation.internalAutowiredAnnotationProcessor** | AutowiredAnnotationBeanPostProcessor 对象   | 处理 @Autowired 以及 @Value注解                       |
| **org.springframework.context.annotation.internalCommonAnnotationProcessor** | CommonAnnotationBeanPostProcessor 对象      | （条件激活）处理 JSR-250 注解，如 @PostConstruct 等   |
| **org.springframework.context.event.internalEventListenerProcessor** | EventListenerMethodProcessor 对象           | 处理标注 @EventListener 的Spring 事件监听方法         |
| **org.springframework.context.event.internalEventListenerFactory** | DefaultEventListenerFactory 对象            | @EventListener 事件监听方法适配为 ApplicationListener |
| **org.springframework.context.annotation.internalPersistenceAnnotationProcessor** | PersistenceAnnotationBeanPostProcessor 对象 | （条件激活）处理 JPA 注解场景                         |

# 依赖查找中常见的异常

- NoSuchBeanDefinitionException
  - 当查找 Bean 不存在于 IoC 容器时
  - BeanFactory#getBean 或 ObjectFactory#getObject
- NoUniqueBeanDefinitionException
  - 类型依赖查找时，IoC 容器存在多个 Bean 实例时
  - BeanFactory#getBean(Class)
- BeanInstantiationException
  - 当 Bean 所对应的类型非具体类时
  - BeanFactory#getBean
- BeanCreationException
  - 当 Bean 初始化过程中
  - Bean 初始化方法执行异常时
- BeanDefinitionStoreException
  - 当 BeanDefinition 配置元信息非法时
  - XML 配置资源无法打开时

### Demo

**NoUniqueBeanDefinitionException**

```java
public class NoUniqueBeanDefinitionExceptionDemo {

    public static void main(String[] args) {
        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        // 将当前类 NoUniqueBeanDefinitionExceptionDemo 作为配置类（Configuration Class）
        applicationContext.register(NoUniqueBeanDefinitionExceptionDemo.class);
        // 启动应用上下文
        applicationContext.refresh();

        try {
            // 由于 Spring 应用上下文存在两个 String 类型的 Bean，通过单一类型查找会抛出异常
            applicationContext.getBean(String.class);
        } catch (NoUniqueBeanDefinitionException e) {
            System.err.printf(" Spring 应用上下文存在%d个 %s 类型的 Bean，具体原因：%s%n",
                    e.getNumberOfBeansFound(),
                    String.class.getName(),
                    e.getMessage());
        }

        // 关闭应用上下文
        applicationContext.close();
    }

    @Bean
    public String bean1() {
        return "1";
    }

    @Bean
    public String bean2() {
        return "2";
    }

    @Bean
    public String bean3() {
        return "3";
    }
}
```

**BeanInstantiationException**

```java
public class BeanInstantiationExceptionDemo {

    public static void main(String[] args) {
        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();

        // 注册 BeanDefinition Bean Class 是一个 CharSequence 接口
        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(CharSequence.class);
        applicationContext.registerBeanDefinition("errorBean", beanDefinitionBuilder.getBeanDefinition());

        // 启动应用上下文
        applicationContext.refresh();

        // 关闭应用上下文
        applicationContext.close();
    }

}
```

**BeanCreationException**

```java
public class BeanCreationExceptionDemo {

    public static void main(String[] args) {
        // 创建 BeanFactory 容器
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();

        // 注册 BeanDefinition Bean Class 是一个 POJO 普通类，不过初始化方法回调时抛出异常
        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(POJO.class);
        applicationContext.registerBeanDefinition("errorBean", beanDefinitionBuilder.getBeanDefinition());

        // 启动应用上下文
        applicationContext.refresh();

        // 关闭应用上下文
        applicationContext.close();
    }

    static class POJO implements InitializingBean {

        @PostConstruct // CommonAnnotationBeanPostProcessor
        public void init() throws Throwable {
            throw new Throwable("init() : For purposes...");
        }

        @Override
        public void afterPropertiesSet() throws Exception {
            throw new Exception("afterPropertiesSet() : For purposes...");
        }
    }
}
```

# 小结

## ObjectFactory 和 BeanFactory

- 都具备依赖查找能力
- ObjectFactory 仅关注一个或一种类型的 Bean 查找，并且自身不具备依赖查找的能力，能力则由 BeanFactory 输出
- BeanFactory 则提供了单一类型、集合类型以及层次性等多种依赖查找方式

```java
@SuppressWarnings("serial")
	private static class TargetBeanObjectFactory implements ObjectFactory<Object>, Serializable {

		private final BeanFactory beanFactory;

		private final String targetBeanName;

		public TargetBeanObjectFactory(BeanFactory beanFactory, String targetBeanName) {
			this.beanFactory = beanFactory;
			this.targetBeanName = targetBeanName;
		}

		@Override
		public Object getObject() throws BeansException {
			// 最终使用 BeanFactory 输出查找 bean
			return this.beanFactory.getBean(this.targetBeanName);
		}
	}
```

## BeanFactory getBean 的操作是否线程安全

BeanFactory.getBean 方法的执行是线程安全的，超过过程中会增加互斥锁

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {
		 ... ... 
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					removeManualSingletonName(beanName);
				}
			}
		 ... ...
	}
```

**注: java6 之后会有偏向锁，因此 Spring 在主线程中完成，否则会出现偏向锁和锁的竞争，可能会导致死锁**