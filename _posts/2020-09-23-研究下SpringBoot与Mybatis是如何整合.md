---
layout:     post
title:      SpringBoot与Mybatis的结合
subtitle:   
date:       2020-10-12
author:     me2in
header-img: img/tuning-tomcat.jpg
catalog: true
tags:
    - mybatis
---

### 1.几个重要概念

spring部分：

mybatis利用到的就是下列4个接口，这里的接口都只有一个方法（#号后边部分），下边依次解释每个接口的作用。

```java
org.springframework.beans.factory.InitializingBean#afterPropertiesSet
org.springframework.context.ApplicationContextAware#setApplicationContext
org.springframework.beans.factory.BeanNameAware#setBeanName
org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
org.springframework.beans.factory.BeanFactoryAware#setBeanFactory
org.springframework.context.annotation.ImportBeanDefinitionRegistrar
```

1. org.springframework.beans.factory.InitializingBean#afterPropertiesSet

   官方的解释是说如果你需要在bean属性初始化后执行一些检查，或者自定义的其他初始化流程，可以实现这个接口，并复写afterPropertiesSet方法

   ```java
   interface to be implemented by beans that need to react once all their properties have been set by a BeanFactory: e.g. to perform custom initialization, or merely to check that all mandatory properties have been set.
   ```

2. org.springframework.context.ApplicationContextAware#setApplicationContext

   ```java
   Set the ApplicationContext that this object runs in. Normally this call will be used to initialize the object.
   Invoked after population of normal bean properties but before an init callback such as InitializingBean.afterPropertiesSet() or a custom init-method. Invoked after ResourceLoaderAware.setResourceLoader(org.springframework.core.io.ResourceLoader), ApplicationEventPublisherAware.setApplicationEventPublisher(org.springframework.context.ApplicationEventPublisher) and MessageSourceAware, if applicable.
   ```

   如果需要ApplicationContext对象，则可以实现该接口。spring会在bean属性初始化完成后设置applicationcontext对象（在InitializingBean#afterPropertiesSet前调用）。

3. org.springframework.beans.factory.BeanNameAware#setBeanName

   ```
   Set the name of the bean in the bean factory that created this bean.
   Invoked after population of normal bean properties but before an init callback such as InitializingBean.afterPropertiesSet() or a custom init-method.
   ```

   如果需要获取spring给bean设置的beanname，则可以实现该方法。在bean属性初始化完成后，初始化方法前调用（比如InitializingBean#afterPropertiesSet）

4. org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry

   ```
   Extension to the standard BeanFactoryPostProcessor SPI, allowing for the registration of further bean definitions before regular BeanFactoryPostProcessor detection kicks in. In particular, BeanDefinitionRegistryPostProcessor may register further bean definitions which in turn define BeanFactoryPostProcessor instances.
   对标准BeanFactoryPostProcessor SPI的扩展，允许在进行常规BeanFactoryPostProcessor检测之前注册其他Bean定义。特别是，BeanDefinitionRegistryPostProcessor可以注册其他Bean定义，这些定义又定义了BeanFactoryPostProcessor实例。
   ```

   BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor子类，执行顺序上是BeanDefinitionRegistryPostProcessor早BeanFactoryPostProcessor。两者的作用是BeanDefinitionRegistryPostProcessor用于增加新的BeanDefinition（mybatis是使用了ClassPathBeanDefinitionScanner的scan方法注册新的mapper beanDefinition），BeanFactoryPostProcessor则是允许修改BeanFactory对象的属性

5. org.springframework.context.annotation.ImportBeanDefinitionRegistrar

   

```
Interface to be implemented by types that register additional bean definitions when processing @Configuration classes. Useful when operating at the bean definition level (as opposed to @Bean method/instance level) is desired or necessary.
Along with @Configuration and ImportSelector, classes of this type may be provided to the @Import annotation (or may also be returned from an ImportSelector).
An ImportBeanDefinitionRegistrar may implement any of the following Aware interfaces, and their respective methods will be called prior to registerBeanDefinitions(org.springframework.core.type.AnnotationMetadata, org.springframework.beans.factory.support.BeanDefinitionRegistry, org.springframework.beans.factory.support.BeanNameGenerator):
由在处理@Configuration类时注册其他bean定义的类型所实现的接口。 在bean定义级别（与@Bean方法/实例级别相对）进行操作时很有用，这是必需的或必需的。
与@Configuration和ImportSelector一起，可以将此类型的类提供给@Import批注（或也可以从ImportSelector返回）。
ImportBeanDefinitionRegistrar可以实现以下任何Aware接口，并且将在registerBeanDefinitions（org.springframework.core.type.AnnotationMetadata，org.springframework.beans.factory.support.BeanDefinitionRegistry，org.springframework.beans之前调用它们的相应方法。 factory.support.BeanNameGenerator）：
```

实现此接口中的registerBeanDefinitions方法即可注入自定义的bean定义，mybatis是通过实现这个方法，注册了MapperScannerConfigurer类（其又实现第4个接口）定义，在postProcessBeanDefinitionRegistry方法中真正的增加mapper类的bean定义







### 2.mapper真正的初始化过程

1.两个核心类 ：MapperScannerConfigurer 和 MapperFactoryBean

1. MapperScannerConfigurer 
   负责在内部初始化一个ClassPathMapperScanner实例，此实例在配置的包路径下按下列三种方式扫描符合条件的接口类

   - 按注解类型，默认mapper.class
   - 按照父类类型  
   - 全部

   扫描出接口类会被封装成BeanDefinition对象，这些BeanDefinition的初始beanclass类型会是对应的mapper接口，在执行完processBeanDefinitions方法后，beanclass类型会被修改成为MapperFactoryBean

2. MapperFactoryBean
   
   从名字可以看出这个类是Spring中特殊类型的bean，这种bean有个getObject方法，此方法返回的bean也会被注册到Spring容器中去。而MapperFactoryBean的getObject方法就是返回的就是通过代理封装的各个mapper接口。同时我们知道在原先的mybatis框架中，调用mapper代理类的方法，实际上都是在内部调用了sqlSession对象的方法，所以MapperFactoryBean对象内部必然有个sqlSession对象：SqlSessionTemplate，我们执行通过Spring容器注入的mapper类，都是在调用SqlSessionTemplate的方法，在其内部完成自动的getSqlSession和closeSqlSession及事务的管理
   

2.Spring boot与Mybatis的结合

结合是基于Springboot的AutoConfiguration机制，整个配置的入口是MybatisAutoConfiguration，但对于上述初始化逻辑入口可能有两个

- 如果在application入口类中加了MapperScan注解，则优先初始化MapperScannerRegistrar类，这个类完成了上述逻辑，这个类也是之前spring与mybatis结合的类
- 如果没有在入口类上加注解，则会使用MybatisAutoConfiguration中的AutoConfiguredMapperScannerRegistrar子类，执行上面的逻辑

3.图解

