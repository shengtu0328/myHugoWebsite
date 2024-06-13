---
title: "Spring"
date: 2022-10-12T17:55:52+08:00
draft: false
---





### spring 循环依赖

#### set依赖

一级缓存：成品对象

二级缓存：半成品对象

三级缓存：创建对象的工厂对象





1. a 依赖了 b， b依赖了a， 
2. new A()实例化时; 就会放入三级缓存
3. a.set(b) 

   1. new B()实例化时; 就会放入三级缓存

   2. b.set(a) 

   3. 去缓存找a ， 一级缓存没有，二级缓存没有，三级缓存有。把a再创建出来，如果a是代理，则在此时创建的是代理对象。把a 放入二级缓存，删除三季缓存中的a。

   4. b创建结束，放入一级缓存
4. 查看在二级缓存中有没有a ，有的话就用二级缓存的a。（如果a是代理对象，被别的对象依赖了，那二级缓存是会有a的，并且和第2步new A的对象肯定不是同一个人）
5. a创建结束





```
AbstractAutowireCapableBeanFactory.class

@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides.
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
       //==================doCreateBean==================
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
         logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```





```
AbstractAutowireCapableBeanFactory.class


protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      //================== bean 的实例创建==============================
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }


   
   //==================== 往三级缓存（工厂map）存入当前对象 ==============================
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      //==================== 往三级缓存（工厂map）存入当前对象 存入的lambda表达式==============================
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   Object exposedObject = bean;
   try {
       //==================== 依赖注入 ==============================
      populateBean(beanName, mbd, instanceWrapper);
      //==================== 执行初始化方法 ==============================
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   if (earlySingletonExposure) {
   
      //==================== 如果a依赖b b依赖a，并且a是代理对象  a一开始会createBeanInstance ，但到这一步时，b已经创建完成了，a会去检查半成品缓存中有没有在b创建对象时候，创建好的代理对象a ===================
      //==============第二个参数为false，不会去三级缓存拿==============
      Object earlySingletonReference = getSingleton(beanName, false);
      // ==================== 是否发生循环依赖==================== 
      if (earlySingletonReference != null) {
          //================如果当前bean是个 需要代理的对象 exposedObject ！= bean==============================
         if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
         }
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
            if (!actualDependentBeans.isEmpty()) {
               throw new BeanCurrentlyInCreationException(beanName,"B....“”      }
         }
      }
   }

   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }

   return exposedObject;
}
```



```
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   if (bw == null) {
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         // Skip property population phase for null instance.
         return;
      }
   }

   // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
   // state of the bean before properties are set. This can be used, for example,
   // to support styles of field injection.
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
         if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
            return;
         }
      }
   }
```

```
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
   
     // =============BeanNameAware,BeanClassLoaderAware,BeanFactoryAware方法================
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
       //================初始化前方法================
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
      //================初始化后方法================
        //==================生成动态代理代理对象==================
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```



![](applyBeanPostProcessorsBeforeInitialization.png)

#### 构造循环依赖

### springmvc





总: 使用了SpringMvc后，所有的请求都需要经过DispatcherServlet前端控制器，该类中提供了一个doDispatch方法，有关请求处理和结果响应的所有流程都在该方法中完成

分：
         首先，借助于HandlerMapping处理器映射器得到处理器执行链，里面封装了HandlerMethod代表目标Controller的方法，同时还通过一个集合记录了要执行的拦截器
         接下来，会根据HandlerMethod获取对应的HandlerAdapter处理器适配器，里面封装了参数解析器以及结果处理器
         然后，执行拦截器的preHandle方法
         接下来是核心，通过HandlerAdapter处理器适配器执行目标Controller的方法，在这个过程中会通过参数解析器和结果处理器分别解析浏览器提交的数据以及处理Controller方法返回的结果
          然后，执行拦截器的postHandle方法，
          最后处理响应，在这个过程中如果有异常抛出，会执行异常的逻辑，这里还会执行全局异常处理器的逻辑，并通过视图解析器ViewResolver解析视图，再渲染视图，最后再执行拦截器的afterCompletion



  ![](springmvc.png)