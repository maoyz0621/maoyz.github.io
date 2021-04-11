# Spring AOP

## AOP的理念：

就是将**分散在各个业务逻辑代码中相同的代码通过横向切割的方式**抽取到一个独立的模块中！

## 底层原理：

**动态代理**，如果要代理的对象实现了某个接口，那么Spring AOP就会使用JDK动态代理去创建代理对象，**可以强制使用CGLIB实现AOP**；而对于没有实现接口的对象，就无法使用JDK动态代理，转而使用CGlib动态代理生成一个被代理对象的子类来作为代理。

##  方式：

- JDK动态代理（默认使用）

- CGLib动态代理

  |                           |
  | :-----------------------: |
  | ![](.\image\AopProxy.png) |

## Spring AOP 和 AspectJ

  ![](.\image\Aspect.jpg)

## 继承关系
|                              |
| :--------------------------: |
| ![](.\image\AOP类继承图.png) |
|                              |
|                              |

## 执行流程

### 从@EnableAspectJAutoProxy注解开始

```java
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

   boolean proxyTargetClass() default false;

   boolean exposeProxy() default false;
}
```

|      AspectJAdvice      |
| :---------------------: |
| ![](.\image\Advice.png) |

对应@Before、 @AfterReturning、  @AfterThrowing、 @After、 @Around

### AspectJAutoProxyRegistrar

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

   @Override
   public void registerBeanDefinitions(
         AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

      AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

      AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
      if (enableAspectJAutoProxy != null) {
         if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
         }
         if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
         }
      }
   }

}
```

实现**ImportBeanDefinitionRegistrar**接口，向IOC容器注册bean，**AspectJAutoProxyRegistrar**注册bean

```
   1）、传入配置类，创建ioc容器
   2）、注册配置类，调用refresh（）刷新容器；
   3）、registerBeanPostProcessors(beanFactory);注册bean的后置处理器来方便拦截bean的创建；
       1）、先获取ioc容器已经定义了的需要创建对象的所有BeanPostProcessor
       2）、给容器中加别的BeanPostProcessor
       3）、优先注册实现了PriorityOrdered接口的BeanPostProcessor；
       4）、再给容器中注册实现了Ordered接口的BeanPostProcessor；
       5）、注册没实现优先级接口的BeanPostProcessor；
       6）、注册BeanPostProcessor，实际上就是创建BeanPostProcessor对象，保存在容器中；
           创建internalAutoProxyCreator的BeanPostProcessor【AnnotationAwareAspectJAutoProxyCreator】
           1）、创建Bean的实例
           2）、populateBean；给bean的各种属性赋值
           3）、initializeBean：初始化bean；
                   1）、invokeAwareMethods()：处理Aware接口的方法回调
                   2）、applyBeanPostProcessorsBeforeInitialization()：应用后置处理器的postProcessBeforeInitialization（）
                   3）、invokeInitMethods()；执行自定义的初始化方法
                   4）、applyBeanPostProcessorsAfterInitialization()；执行后置处理器的postProcessAfterInitialization（）；
           4）、BeanPostProcessor(AnnotationAwareAspectJAutoProxyCreator)创建成功；--》aspectJAdvisorsBuilder
       7）、把BeanPostProcessor注册到BeanFactory中；
           beanFactory.addBeanPostProcessor(postProcessor);  
```

| org.springframework.aop.config.AopConfigUtils#registerOrEscalateApcAsRequired |
| ------------------------------------------------------------ |
| ![](.\image\AnnotationAwareAspectJAutoProxyCreator.png)      |

*IOC容器中注入了一个internalAutoProxyCreator=AnnotationAwareAspectJAutoProxyCreator的bean，*到此可以得出结论，@EnableAspectJAutoProxy给容器中注册一个AnnotationAwareAspectJAutoProxyCreator。

### AspectJAwareAdvisorAutoProxyCreator 



### AnnotationAwareAspectJAutoProxyCreator

这个类实现了**SmartInstantiationAwareBeanPostProcessor**和**BeanFactoryAware**接口

```java
BeanPostProcessor
	InstantiationAwareBeanPostProcessor
		SmartInstantiationAwareBeanPostProcessor   BeanFactoryAware
			AbstractAutoProxyCreator
				AbstractAdvisorAutoProxyCreator
					AspectJAwareAdvisorAutoProxyCreator
						AnnotationAwareAspectJAutoProxyCreator
```



发生在初始化之后

> org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessAfterInitialization

```
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
       if (bean != null) {
              Object cacheKey = getCacheKey(bean.getClass(), beanName);
              if (!this.earlyProxyReferences.contains(cacheKey)) {
              		 // 这里根据原来的bean，创建动态代理，并返回给ioc容器
                     return wrapIfNecessary(bean, beanName, cacheKey);
              }
       }
       return bean;
}
```



### 创建AOP 代理



> org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#createProxy

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
       if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
              return bean;
       }
       if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
              return bean;
       }
       if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
              this.advisedBeans.put(cacheKey, Boolean.FALSE);
              return bean;
       }

       // 获取Advisor对象，并保存到Object[] specificInterceptors 里
       Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
       if (specificInterceptors != DO_NOT_PROXY) {
              this.advisedBeans.put(cacheKey, Boolean.TRUE);
              // 创建代理，注意，Advisor数组，已经被传进去了。
              Object proxy = createProxy(
                            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
              this.proxyTypes.put(cacheKey, proxy.getClass());
              return proxy;
       }

       this.advisedBeans.put(cacheKey, Boolean.FALSE);
       return bean;
}
// 创建代理
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
              @Nullable Object[] specificInterceptors, TargetSource targetSource) {

       if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
              AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
       }

       ProxyFactory proxyFactory = new ProxyFactory();
       proxyFactory.copyFrom(this);

       if (!proxyFactory.isProxyTargetClass()) {
              if (shouldProxyTargetClass(beanClass, beanName)) {
                     proxyFactory.setProxyTargetClass(true);
              }
              else {
                     evaluateProxyInterfaces(beanClass, proxyFactory);
              }
       }

       Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
       proxyFactory.addAdvisors(advisors);
       proxyFactory.setTargetSource(targetSource);
       customizeProxyFactory(proxyFactory);

       proxyFactory.setFrozen(this.freezeProxy);
       if (advisorsPreFiltered()) {
              proxyFactory.setPreFiltered(true);
       }

       return proxyFactory.getProxy(getProxyClassLoader());
}
```



创建代理对象

> org.springframework.aop.framework.DefaultAopProxyFactory#createAopProxy

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
       if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
              Class<?> targetClass = config.getTargetClass();
              if (targetClass == null) {
                     throw new AopConfigException("TargetSource cannot determine target class: " +
                                   "Either an interface or a target is required for proxy creation.");
              }
              if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                     // jdk动态代理；
                     return new JdkDynamicAopProxy(config);
              }
              // cglib的动态代理；
              return new ObjenesisCglibAopProxy(config);
       }
       else {
           	  // 默认jdk动态代理；
              return new JdkDynamicAopProxy(config);
       }
}
```



|                              |
| ---------------------------- |
| ![](.\image\createProxy.png) |

> 实现了多个切面，如何保证切面执行顺序？

实现了AOP之后，执行方法之前进入**JdkDynamicAopProxy#invoke**或**CglibAopProxy.DynamicAdvisedInterceptor#intercept**，都会获取调用链**advised.getInterceptorsAndDynamicInterceptionAdvice()**，之后执行方法proceed()，拦截器的触发过程，

![](.\image\Aspect调用链.png)

> org.springframework.aop.framework.DefaultAdvisorChainFactory

```java
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
      Advised config, Method method, @Nullable Class<?> targetClass) {
   List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
   Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
   boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
   AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

   for (Advisor advisor : config.getAdvisors()) {
      if (advisor instanceof PointcutAdvisor) {
         // Add it conditionally.
         PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
         if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
            MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
            if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
               MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
               if (mm.isRuntime()) {
                  // Creating a new object instance in the getInterceptors() method
                  // isn't a problem as we normally cache created chains.
                  for (MethodInterceptor interceptor : interceptors) {
                     interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                  }
               }
               else {
                  interceptorList.addAll(Arrays.asList(interceptors));
               }
            }
         }
      }
      else if (advisor instanceof IntroductionAdvisor) {
         IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
         if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
         }
      }
      else {
         // 获取所有增强器
         Interceptor[] interceptors = registry.getInterceptors(advisor);
         interceptorList.addAll(Arrays.asList(interceptors));
      }
   }

   return interceptorList;
}
```



## 总结

1. 开启AOP注解@EnableAspectJAutoProxy 或 <aop:aspectj-autoproxy proxy-target-class="true"/>

2. 开启AOP之后，给容器注册一个后置组件：AnnotationAwareAspectJAutoProxyCreator

3. AnnotationAwareAspectJAutoProxyCreator是一个后置处理器，执行postProcessAfterInitialization()

4. 容器的创建过程：

   1）registerBeanPostProcessors()注册后置处理器，创建AnnotationAwareAspectJAutoProxyCreator

   2）finishBeanFactoryInitialization())初始化剩下的单实例bean

   ​	1）创建逻辑组件和切面组件

   ​	2）AnnotationAwareAspectJAutoProxyCreator拦截组件创建过程

   ​	3）创建完成之后，判断你组件是否需要增强，是：切面的通知方法包装成增强器（Advisor），给逻辑组件创建代理对象

5. 执行目标方法

   1）代理对象执行方法

   2）JdkDynamicAopProxy#invoke或CglibAopProxy.DynamicAdvisedInterceptor#intercept
   
   ​	1）获取目标方法的拦截器链
   
   ​	2）利用拦截器的链式机制，依次进入每个拦截器进行执行
   
   ​	3）效果：
   
   ​		正常：环绕前置通知 -> 前置通知 -> 目标方法 -> 环绕后置通知 -> 后置通知 -> 返回通知
   
   ​		异常：环绕前置通知 -> 前置通知 -> 目标方法 -> 环绕后置通知 ->  -> 后置通知 -> 异常通知
   
   ​		Spring-5.2.8.RELEASE之后：
   
   ​		正常：环绕前置通知 -> 前置通知 -> 目标方法  -> 返回通知 -> 后置通知- > 环绕后置通知 
   
   ​		异常：环绕前置通知 -> 前置通知 -> 目标方法  -> 异常通知 -> 后置通知 -> 环绕后置通知