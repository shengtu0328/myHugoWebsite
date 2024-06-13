---
title: "Springboot启动流程"
date: 2023-05-24T10:38:27+08:00
draft: true
---



## 几个重要的类

AnnotationConfigServletWebServerApplicationContext 包含了beanFactory 

beanFactory 包含了singletonObjects



## 总结springboot 启动流程  

```
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
```

1. new SpringApplication()

   首先會創建一个 SpringApplication对象，在new SpringApplication() 构造方法中，

   ```
   this.bootstrapRegistryInitializers = new ArrayList<>(
         getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
   ```

   这里是第一次调用SgetSpringFactoriesInstances 方法，首次调用时，  SpringFactoriesLoader.loadSpringFactories 会解析当前ClassLoader中所有jar包下的spring.factories文件，包括我们项目里自定义的spring.factories文件。将文件中的内容即 "接口全限定名-实现类全限定名（1对多的关系）"，

   SpringFactoriesLoader.loadSpringFactories 将这些className 的List<String>放入缓存的名为cache 的map中，static final Map<ClassLoader, Map<String, List<String>>> cache = new ConcurrentReferenceHashMap<>();**以供后续使用，**

   ```
   String[] getBeanNamesForType 返回的是字符串
   
   ```

   并且还实例化了几个默认的bean。

2. 在run方法中

   1. createApplicationContext()

      创建上下文对象 

      AnnotationConfigUtils.registerAnnotationConfigProcessors 注册以下几个**BeanDefinition**

      1.org.springframework.context.annotation.internalConfigurationAnnotationProcessor（也就是**ConfigurationClassPostProcessor** 实现了BeanFactoryPostProcessor）

      2.org.springframework.context.annotation.internalAutowiredAnnotationProcessor
   
      3.org.springframework.context.annotation.internalCommonAnnotationProcessor
   
      4.org.springframework.context.event.internalEventListenerProcessor
   
      5.org.springframework.context.event.internalEventListenerFactory
   
      
   
   2. prepareEnvironment() 
   
      系统的Environment，不是自定义的那种properties类
   
   3. prepareContext() 准备上下文
      1. 加载当前系统的Environment的一些变量
      
      2. ApplicationContextInitializer实现类方法调用

      3. 调用load(context, sources.toArray(new Object[0]));创建BeanDefinitionLoader  
      
         主要就是 注册 **启动类** 的 **BeanDefinition**   **用BeanDefinitionLoader注册**
      
         **BeanDefinition**   就是一个map和一个list
   
   ```
   private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
   private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
   ```
   
   1. **refreshContext()** 
   
      **spring的核心方法，springboot也是依托于此**
   
      1. prepareRefresh();
   
          创建 environment对象，一些系统的，java 系统的 ，自定义的key value信息，
   
      2. obtainFreshBeanFactory();  
   
         获取BeanFactory
   
      3. prepareBeanFactory(beanFactory);
   
      3. postProcessBeanFactory(beanFactory);   
   
            对BeanFactory set属性
   
      5. **invokeBeanFactoryPostProcessors（）** 
   
         **实例化BeanFactoryPostProcessors。  扫描需要放入容器中的bean，放入bd里**。也算是**springboot 自动装配**的起点 
   
         1. beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false)
   
            找到了实现了**BeanDefinitionRegistryPostProcessor** 接口 的**ConfigurationClassPostProcessor** 的 String
   
            String[] getBeanNamesForType 返回的是字符串
      
         1. currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
            
            进行实例化，currentRegistryProcessors就是 ConfigurationClassPostProcessor
            
         3. invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
   
            **先解析 启动类包下的 再解析自动装配的 类**
   
            ![img](deferredImportSelectorHandler.process().png)
   
            **概括一下 就是  在ConfigurationClassPostProcessor 类中用   ConfigurationClassParser 解析启动类**
   
            3.1 **ConfigurationClassPostProcessor**.postProcessBeanDefinitionRegistry()
   
            3.1.1 ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory))
   
            通过bd是否有被@Configuration 修饰 ，过滤出  **启动类**的  **String名字**
   
            
   
            3.1.1.1 **ConfigurationClassParser** parser.parse(candidates);
   
            ConfigurationClassParser.解析 **启动类**
   
            **ConfigurationClassParser**.doProcessConfigurationClass 方法 解析启动类上的@Configuration注解，  @ComponentScan @PropertySource 注解，
   
            **ConfigurationClassParser**.collectImports 解析启动类上的 import注解，并获取import@import注解里的value  即 **AutoConfigurationImportSelector**，然后会对AutoConfigurationImportSelector实例化
   
            
   
            **概括一下 就是  在AutoConfigurationImportSelector类中用  注册BeanDefinition**
   
            
   
            DeferredImportSelectorHandler.process()
   
            DeferredImportSelectorHandler.processGroupImports()
   
            1. 调用**AutoConfigurationImportSelector&AutoConfigurationGroup类的 **process()方法，从SpringFactoriesLoader.loadFactoryNames 缓存加载**EnableAutoConfiguration.class**的 自动配置实现类，加载到**BeanDefinition**缓存
   
         4. 实例化一些第三方的、自定义的 BeanFactoryPostProcessors ,如Mybatis 的**MapperScannerConfigurer**。并且这些BeanFactoryPostProcessors如果没有创建bean，就会在此时为他们实例化。 并且调用他们的**postProcessBeanDefinitionRegistry**()方法，**主要作用是加载BeanDefinition**
   
         5. 1
   
         6. 5
   
         7. 888
   
         8. 99
   
      5. **registerBeanPostProcessors(beanFactory);** 
   
         实例化 BeanPostProcessor 并注册到 BeanFactory 的 beanPostProcessors 缓存中，如**AutowiredAnnotationBeanPostProcessor**
   
      6. **onRefresh();** 
   
          启动内嵌的**tomcat**
   
      7. **finishBeanFactoryInitialization(beanFactory);**
   
         实例化BeanDefinition里的bean
   
         普通bean的 **按以下顺序依次执行**
   
         1. 构造方法调用       
   
         2. populateBean(beanName, mbd, instanceWrapper);  
   
            也就是 AutowiredAnnotationBeanPostProcessor.postProcessProperties()   会调用 beanFactory.getBean(beanName)  获取需要autowired 的对象，如果还没有实例化，就实例化需要autowired的对象
   
         3. initializeBean 
   
            3.1 invokeAwareMethods 
   
            ((BeanNameAware) bean).setBeanName(beanName);
   
            =================
   
            3.2 iapplyBeanPostProcessorsBeforeInitialization	
   
            1.((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
   
            2.@PostConstruct
   
            =================
   
            3.3 invokeInitMethods
   
            1.InitializingBean. afterPropertiesSet()   初始化方法
   
            =================
   
            3.4 applyBeanPostProcessorsAfterInitialization
   
        8. **finishRefresh();**
   
             对 ApplicationListener  onApplicationEvent方法 进行回调
      
        
      
         
      
      
   

## 7. SpringBoot 自动配置原理

**要求**

* 掌握 SpringBoot 自动配置原理

**自动配置原理**

@SpringBootConfiguration 是一个组合注解，由 @ComponentScan、@EnableAutoConfiguration 和 @SpringBootConfiguration 组成

1. @SpringBootConfiguration 与普通 @Configuration 相比，唯一区别是前者要求整个 app 中只出现一次
2. @ComponentScan
   * excludeFilters - 用来在组件扫描时进行排除，也会排除自动配置类

3. @EnableAutoConfiguration 也是一个组合注解，由下面注解组成
   * @AutoConfigurationPackage – 用来记住扫描的起始包
   * @Import(AutoConfigurationImportSelector.class) 用来加载 `META-INF/spring.factories` 中的自动配置类

**为什么不使用 @Import 直接引入自动配置类**

有两个原因：

1. 让主配置类和自动配置类变成了强耦合，主配置类不应该知道有哪些从属配置
2. 直接用 `@Import(自动配置类.class)`，引入的配置解析优先级较高，自动配置类的解析应该在主配置没提供时作为默认配置

因此，采用了 `@Import(AutoConfigurationImportSelector.class)`

* 由 `AutoConfigurationImportSelector.class` 去读取 `META-INF/spring.factories` 中的自动配置类，实现了弱耦合。
* 另外 `AutoConfigurationImportSelector.class` 实现了 DeferredImportSelector 接口，让自动配置的解析晚于主配置的解析



beanFactory是获取bean的

```
currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
```
