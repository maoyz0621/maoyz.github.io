# Spring三级缓存

前景概要：解决对象间的循环依赖

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class PrototypeIocA {

    @Autowired
    private PrototypeIocB prototypeIocB;

    public PrototypeIocA() {
        System.out.println("IocA .....");
    }

}

@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class PrototypeIocB {

    @Autowired
    private PrototypeIocA prototypeIocA;

    public PrototypeIocB() {
        System.out.println("IocB ...");
    }
}
```

原型（prototype）不支持循环依赖

> Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'prototypeIocA': Requested bean is currently in creation: Is there an unresolvable circular reference?



```java
@Component
public class ConstructorIocA {

    private ConstructorIocB constructorIocB;

    public ConstructorIocA() {
        System.out.println("IocA .....");
    }

    @Autowired
    public ConstructorIocA(ConstructorIocB constructorIocB) {
        this.constructorIocB = constructorIocB;
    }
}

@Component
public class ConstructorIocB {

    private ConstructorIocA setterIocA;

    public ConstructorIocB() {
        System.out.println("IocB ...");
    }

    @Autowired
    public ConstructorIocB(ConstructorIocA setterIocA) {
        this.setterIocA = setterIocA;
    }
}

```

构造方法注入不支持循环依赖

> Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'constructorIocA': Requested bean is currently in creation: Is there an unresolvable circular reference?



```
@Component
public class SetterIocA {

    @Autowired
    private SetterIocB setterIocB;

    public SetterIocA() {
        System.out.println("IocA .....");
    }
}

@Component
public class SetterIocB {

    @Autowired
    private SetterIocA setterIocA;

    public SetterIocB() {
        System.out.println("IocB ...");
    }
}
```



<img src=".\image\Spring三级缓存源代码执行流程图.jpg" style="zoom:200%;" />

> org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
>

```java
// 一级缓存，单例池，存放已经经历完整生命周期的单例bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
// 二级缓存，存放早起暴露出来的bean对象，已经实例化，属性未赋值的 单例bean
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
// 三级缓存, 存放可以生成单例bean的工厂
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
// 已经注册的单例池里的beanName
private final Set<String> registeredSingletons = new LinkedHashSet<>(256);
// 正在创建中的beanName
private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
            // 三级缓存，在doCreateBean()中创建了bean的实例后，封装ObjectFactory放入缓存
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               // 创建未赋值的bean
               singletonObject = singletonFactory.getObject();
               // 存放二级缓存
               this.earlySingletonObjects.put(beanName, singletonObject);
               // 从三级缓存删除
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```



> AbstractAutowireCapableBeanFactor#doCreateBean
>

```java
// 创建Bean
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // Instantiate the bean.
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      // 
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      // 实例化对象
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   final Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            
         }
         mbd.postProcessed = true;
      }
   }

   // Eagerly cache singletons to be able to resolve circular references
   // even when triggered by lifecycle interfaces like BeanFactoryAware.
   // 判断是否允许提前暴露对象，如果允许，添加一个ObjectFactory放入三级缓存
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      // 添加三级缓存
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
      // 填充属性
      populateBean(beanName, mbd, instanceWrapper);
      // 初始化方法，并创建代理
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
        
      }
   }

   if (earlySingletonExposure) {
      // 若存在循环依赖则代理对象提前生成，需要把后续原始bean置换成代理bean
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
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
               
            }
         }
      }
   }

   // Register bean as disposable.
   try {
      // 注册bean的销毁方法
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }

   return exposedObject;
}

```

> Spring一开始提前暴露的并不是实例化的bean，而是将bean包装成ObjectFactory



> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#getEarlyBeanReference

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
	Object exposedObject = bean;
	// 
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
            // 如果是SmartInstantiationAwareBeanPostProcessor，提前生成代理对象
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                // 如果需要代理，返回代理对象，否者返回元素对象
				exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
			}
		}
	}
	return exposedObject;
}
```

三级缓存的bean工厂getObject方式，实际执行的是getEarlyBeanReference，如果对象需要被代理(存在beanPostProcessors -> SmartInstantiationAwareBeanPostProcessor)，则提前生成代理对象。



> org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#getEarlyBeanReference

```java
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
	Object cacheKey = getCacheKey(bean.getClass(), beanName);
	if (!this.earlyProxyReferences.contains(cacheKey)) {
		this.earlyProxyReferences.add(cacheKey);
	}
	return wrapIfNecessary(bean, beanName, cacheKey);
}
```





> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean

```java
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
	boolean continueWithPropertyPopulation = true;

    // 判断是否有InstantiationAwareBeanPostProcessor，提前生成代理对象
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        // 遍历BeanPostProcessor
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
            // 如果有InstantiationAwareBeanPostProcessor，提前生成代理对象
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 如果返回false，代表不需要进行后续的属性设值，也不需要再经过其他的 BeanPostProcessor 的处理
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					continueWithPropertyPopulation = false;
					break;
				}
			}
		}
	}

	if (!continueWithPropertyPopulation) {
		return;
	}

	PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

	if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
		// Add property values based on autowire by name if applicable.
        // 通过名字找到所有属性值，如果是 bean 依赖，先初始化依赖的 bean。记录依赖关系
		if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}
		// Add property values based on autowire by type if applicable.
        // 通过类型装配
		if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}
		pvs = newPvs;
	}

    // 判断是否有InstantiationAwareBeanPostProcessor，提前生成代理对象
	boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
	boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

	if (hasInstAwareBpps || needsDepCheck) {
		if (pvs == null) {
			pvs = mbd.getPropertyValues();
		}
		PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
		if (hasInstAwareBpps) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
                // 
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    // 执行@Resource、@Autowired
                    // BeanPostProcessor: CommonAnnotationBeanPostProcessor和AutowiredAnnotationBeanPostProcessor
                    // 对采用 @Autowired、@Value 注解的依赖进行设值
					pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
					if (pvs == null) {
						return;
					}
				}
			}
		}
		if (needsDepCheck) {
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}
	}

	if (pvs != null) {
        // 设置 bean 实例的属性值
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
}
```



> org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingletonFactory

```java

public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
	Assert.notNull(beanName, "Bean name must not be null");
	Assert.notNull(singletonObject, "Singleton object must not be null");
	synchronized (this.singletonObjects) {
		Object oldObject = this.singletonObjects.get(beanName);
		if (oldObject != null) {
			throw new IllegalStateException("Could not register object [" + singletonObject +
					"] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
		}
		addSingleton(beanName, singletonObject);
	}
}

// 添加到三级缓存
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(singletonFactory, "Singleton factory must not be null");
	synchronized (this.singletonObjects) {
		if (!this.singletonObjects.containsKey(beanName)) {
            // 一级缓存没有，放入三级缓存
			this.singletonFactories.put(beanName, singletonFactory);
            // 从二级缓存删除，确保二级缓存没有该bean？
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
}

// 添加到一级缓存中
protected void addSingleton(String beanName, Object singletonObject) {
	synchronized (this.singletonObjects) {
        // 放入一级缓存
		this.singletonObjects.put(beanName, singletonObject);
        // 从三级缓存删除
		this.singletonFactories.remove(beanName);
        // 从二级缓存删除
		this.earlySingletonObjects.remove(beanName);
		this.registeredSingletons.add(beanName);
	}
}
```



singletonFactories -> earlySingletonObjects -> singletonObjects

1. getSingleton()：存放二级缓存，删除三级缓存

2. addSingletonFactory()：存放三级缓存，删除二级缓存

3. addSingleton()：存放一级缓存，删除二级缓存、三级缓存



A，B循环依赖，先初始化A，先暴露一个半成品A，再去初始化依赖的B，初始化B时如果发现B依赖A，也就是循环依赖，就注入半成品A，之后初始化完毕B，再回到A的初始化过程时就解决了循环依赖，在这里只需要一个Map能缓存半成品A就行了，也就是二级缓存就够了，但是这个二级缓存存的是Bean对象，如果这个对象存在代理，那应该注入的是代理，而不是Bean，此时二级缓存无法既缓存Bean，又缓存代理，因此三级缓存做到了缓存 工厂 ，也就是生成代理。二级缓存就能解决缓存依赖，三级缓存解决的是代理。



总结：

1、调用 doGetBean() 方法，想要获取 beanA ，于是调用 getSingleton() 方法从缓存中查找 beanA

2、在 getSingleton() 方法中，从一级缓存中查找，没有，返回 null

3、doGetBean() 方法中获取到 beanA 为 null ，于是走对应的处理逻辑，调用 getSingleton() 的重载方法(参数为 ObjectFactory 的)

4、在 getSingleton()方法中，先将 beanA_name 添加到一个集合中，用于标记该 bean 正在创建中，然后回调匿名内部类的 createBean 方法

5、进入 AbstractAutowireCapableBeanFactory#doCreateBean，先反射调用构造器创建出 beanA 的实例，然后判断，是否为单例，是否允许提前暴露引用（对于单例一般为true）、是否正在创建中（即是否是在第四步的集合中）判断为 true 则将 beanA 添加到【三级缓存】中

6、对 beanA 进行属性填充，此时检测到 beanA 依赖于 beanB ，于是查找 beanB

7、调用 doGetBean() 方法，和上面 beanA 的过程一样，到缓存中查询 beanB ，没有则创建，然后给 beanB 填充属性

8、此时 beanB 依赖于 beanA ，调用 getSingleton() 获取 beanA ,依次从一级、二级、三级缓存中找、此时从三级缓存中获取到 beanA 的创建工厂，通过创建工厂获取到 singletonObject ，此时这个 singletonObject 指向的就是上面在 doCreateBean() 方法中实例化的 beanA

9、这样 beanB 就获取到了 beanA 的依赖，于是 beanB 顺利完成初始化，并将 beanA 从三级缓存移动到二级缓存中

10、随后 beanA 继续他的属性填充工作，此时也获取到了 beanB ，beanA 也随之完成了创建，回到 getSingleton() 方法中继续向下执行，将 beanA 从二级缓存移动到一级缓存中·