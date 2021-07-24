---
layout:     post
title:      Spring IOC 依赖注入二
subtitle:   Spring IOC 依赖注入讲解二
date:       2020-05-18
author:     Huairho
header-img: img/spring-bean-base.jpg
catalog: true
tags:
    - Spring IOC 篇
---

# 依赖处理过程

## 基础知识

- 入口方法: `DefaultListableBeanFactory#resolveDependency`
- 依赖描述符: `DependencyDescriptor`
- 自定绑定候选对象处理器: `AutowireCandidateResolver`

## `AutowireCapableBeanFactory#resolveDependency()`

```java
/**
 * DependencyDescriptor 依赖描述符
 * requestingBeanName 所需要注入的 bean name
 * autowiredBeanNames 查找的 bean name list
 * TypeConverter 类型转换器
 */
@Nullable
Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException;
```

## 依赖描述符: `DependencyDescriptor`

```java
public class DependencyDescriptor extends InjectionPoint implements Serializable {

  // 被注入的类
	private final Class<?> declaringClass;
  
  // 注入方法
	@Nullable
	private String methodName;
  
  // 参数类型，参数注入时使用
	@Nullable
	private Class<?>[] parameterTypes;

  // 参数下标
	private int parameterIndex;
  
  // 参数名称
	@Nullable
	private String fieldName;
  
  // 声明是否需要带注释的依赖项
	private final boolean required;
  
  // 是否饥饿的，对应 lazy
	private final boolean eager;
  
  // 级别，用于标记是否是嵌套类
	private int nestingLevel = 1;

  // 包含的类
	@Nullable
	private Class<?> containingClass;

  // 泛型类型
	@Nullable
	private transient volatile ResolvableType resolvableType;
  
  // 类型描述
	@Nullable
	private transient volatile TypeDescriptor typeDescriptor;
... ...
}
```

## **自动绑定候选对象处理器**

```
org.springframework.beans.factory.support.AutowireCandidateResolver
```

## **注入注解顺序**

@Resource > @Autowired > @Value > @Inject

@Resource 处理类 `org.springframework.context.annotation.CommonAnnotationBeanPostProcessor`

## **Autowired**

```java
 @Autowired
 private Map<String, User> nonQualifierUserMap;
```

上述方式可以替换applicationContext.getBean();applicationContext.getBean(); 依赖查找Autowired 注入一个map中，key为bean name， value 为bean，因此可以通过依赖注入的方式替代依赖查找。

**AutowiredAnnotationBeanPostProcessor 为注解的初始化处理类。**

### **新增自定义的注入注解**

前提说明，`AnnotationConfigUtils#registerAnnotationConfigProcessors()`方法会检查注册的bean是否存在，如果存在则不再处理。

```java
 if(!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def,  AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
 }
```

实现方案

```java
     @Bean(AnnotationConfigUtils.AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)
     // 替换 spring 自己声明名称为AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME
     // 的 AutowiredAnnotationBeanPostProcessor bean
     public static AutowiredAnnotationBeanPostProcessor beanPostProcessor() {
         AutowiredAnnotationBeanPostProcessor postProcessor = new AutowiredAnnotationBeanPostProcessor();
 //        Set<Class<? extends Annotation>> annotationTypes = new HashSet<>(asList(Autowired.class, Value.class, MyInject.class));
 //        postProcessor.setAutowiredAnnotationTypes(annotationTypes);
         postProcessor.setAutowiredAnnotationType(MyInject.class);
         return postProcessor;
     }
     @Bean
     public static AutowiredAnnotationBeanPostProcessor beanPostProcessor2() {
         AutowiredAnnotationBeanPostProcessor postProcessor = new AutowiredAnnotationBeanPostProcessor();
         postProcessor.setAutowiredAnnotationType(MyInject.class);
         return postProcessor;
     }
```

## **依赖注入方式如下**

### **spring**

- 构造器注入
- setter 注入
- 方法注入
- 字段注入

### **其他**

- 接口回调注入

## **setter 和 构造器注入的选择**

两种方式均可， 推荐使用构造器注入，避免线程并发问题，setter可选。减少注解依赖注入。