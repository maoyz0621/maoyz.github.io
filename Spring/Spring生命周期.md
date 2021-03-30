# Bean生命周期

|                                           |
| :---------------------------------------: |
| ![Bean生命周期](.\image\Bean生命周期.jpg) |

## ApplicationContextInitializer

> org.springframework.context.ApplicationContextInitializer

Spring容器刷新refresh()之前初始化`ConfigurableApplicationContext`的回调接口，容器刷新之前调用此类的`initialize`()方法。

可以在整个spring容器还没被初始化之前做一些事情。在最开始激活一些配置，或者利用这时候class还没被类加载器加载的时机，进行动态字节码注入等操作

源码：

|                                                              |
| ------------------------------------------------------------ |
| ![Bean生命周期](.\image\ApplicationContextInitializer-1.png) |
| ![Bean生命周期](.\image\ApplicationContextInitializer-2.png) |
| ![Bean生命周期](.\image\ApplicationContextInitializer-3.png) |

扩展的生效，有以下三种方式：

- 在启动类中用`springApplication.addInitializers(new XxxApplicationContextInitializer())`语句加入
- 配置文件配置`context.initializer.classes=com.example.demo.XxxApplicationContextInitializer`
- Spring SPI扩展，在spring.factories中加入`org.springframework.context.ApplicationContextInitializer=com.example.demo.XxxApplicationContextInitializer`



## BeanDefinitionRegistryPostProcessor

> org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor

在读取项目中的`beanDefinition`之后执行，提供一个补充的扩展点

使用场景：你可以在这里动态注册自己的`beanDefinition`，可以加载classpath之外的bean



## **BeanFactoryPostProcessor**

> org.springframework.beans.factory.config.BeanFactoryPostProcessor

beanFactory的扩展接口，Spring在读取beanDefinition信息之后，实例化bean之前，可以用在修改已经注册的beanDefinition的元信息



## BeanNameAware

> org.springframework.beans.factory.BeanNameAware

bean初始化之前



## BeanFactoryAware

> org.springframework.beans.factory.BeanFactoryAware

发生在bean的实例化之后，注入属性之前。



## ApplicationContextAware

> org.springframework.context.ApplicationContextAware





## BeanPostProcessor

bean已经实例化完成，`BeanPostProcess`接口只在bean的初始化阶段进行扩展（注入spring上下文前后）

- postProcessBeforeInitialzation( Object bean, String beanName ) 
当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。 先于InitialzationBean执行，因此称为前置处理。 所有Aware接口的注入就是在这一步完成的。

- postProcessAfterInitialzation( Object bean, String beanName ) 
当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。 在InitialzationBean完成后执行，因此称为后置处理。



## InstantiationAwareBeanPostProcessor

> org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor

继承BeanPostProcessor，**把可扩展的范围增加了实例化阶段和属性注入阶段。**其调用顺序：

- `postProcessBeforeInstantiation`：实例化bean之前，相当于new这个bean之前
- `postProcessAfterInstantiation`：实例化bean之后，相当于new这个bean之后
- `postProcessPropertyValues`：bean已经实例化完成，在属性注入阶段触发，`@Autowired`,`@Resource`等注解原理基于此方法实现
- `postProcessBeforeInitialization`：初始化bean之前，相当于把bean注入spring上下文之前
- `postProcessAfterInitialization`：初始化bean之后，相当于把bean注入spring上下文之后

使用场景：对实现了某一类接口的bean在各生命周期收集，或对莫个类型的bean进行统一设值





## **SmartInstantiationAwareBeanPostProcessor**

> org.springframework.beans.factory.config.SmartInstantiationAwareBeanPostProcessor

继承`InstantiationAwareBeanPostProcessor`，

- `predictBeanType`：该触发点发生在`postProcessBeforeInstantiation`之前(在图上并没有标明，因为一般不太需要扩展这个点)，这个方法用于预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null；当你调用`BeanFactory.getType(name)`时当通过bean的名字无法得到bean类型信息时就调用该回调方法来决定类型信息。

- `determineCandidateConstructors`：该触发点发生在`postProcessBeforeInstantiation`之后，用于确定该bean的构造函数之用，返回的是该bean的所有构造函数列表。用户可以扩展这个点，来自定义选择相应的构造器来实例化这个bean。

- `getEarlyBeanReference`：该触发点发生在`postProcessAfterInstantiation`之后，当有循环依赖的场景，当bean实例化好之后，为了防止有循环依赖，会提前暴露回调方法，用于bean实例化的后置处理。这个方法就是在提前暴露的回调方法中触发。

  

## ApplicationContextAwareProcessor

> org.springframework.context.support.ApplicationContextAwareProcessor

BeanPostProcessor的实现类，bean实例化之后，初始化之前

|                                                   |
| :-----------------------------------------------: |
| ![](.\image\ApplicationContextAwareProcessor.png) |

- `EnvironmentAware`：用于获取`EnviromentAware`的一个扩展类，这个变量非常有用， 可以获得系统内的所有参数。当然个人认为这个Aware没必要去扩展，因为spring内部都可以通过注入的方式来直接获得。

- `EmbeddedValueResolverAware`：用于获取`StringValueResolver`的一个扩展类， `StringValueResolver`用于获取基于`String`类型的properties的变量，一般我们都用`@Value`的方式去获取，如果实现了这个Aware接口，把`StringValueResolver`缓存起来，通过这个类去获取`String`类型的变量，效果是一样的。

- `ResourceLoaderAware`：用于获取`ResourceLoader`的一个扩展类，`ResourceLoader`可以用于获取classpath内所有的资源对象，可以扩展此类来拿到`ResourceLoader`对象。

- `ApplicationEventPublisherAware`：用于获取`ApplicationEventPublisher`的一个扩展类，`ApplicationEventPublisher`可以用来发布事件，结合`ApplicationListener`来共同使用，下文在介绍`ApplicationListener`时会详细提到。这个对象也可以通过spring注入的方式来获得。

- `MessageSourceAware`：用于获取`MessageSource`的一个扩展类，`MessageSource`主要用来做国际化。

- `ApplicationContextAware`：用来获取`ApplicationContext`的一个扩展类，`ApplicationContext`应该是很多人非常熟悉的一个类了，就是spring上下文管理器，可以手动的获取任何在spring上下文注册的bean，我们经常扩展这个接口来缓存spring上下文，包装成静态方法。同时`ApplicationContext`也实现了`BeanFactory`，`MessageSource`，`ApplicationEventPublisher`等接口，也可以用来做相关接口的事情。

  


## @PostConstruct

bean初始化阶段，如果一个方法有此注解，会先调用这个方法。发生在`BeanPostProcessor.postProcessBeforeInitialization`之后，`InitializingBean.afterPropertiesSet`之前。



## InitializingBean

> org.springframework.beans.factory.InitializingBean

初始化bean，`BeanPostProcessor.postProcessBeforeInitialzation`的前置处理完成后，发生在`BeanPostProcessor.postProcessAfterInitialization`之前，可以在系统启动的时候一些业务指标的初始化工作。

- afterPropertiesSet()

这一阶段也可以在bean正式构造完成前增加我们自定义的逻辑，但它与前置处理不同，由于该函数并不会把当前bean对象传进来，因此在这一步没办法处理对象本身，只能增加一些额外的逻辑。 
若要使用它，我们需要让bean实现该接口，并把要增加的逻辑写在该函数中。然后Spring会在前置处理完成后检测当前bean是否实现了该接口，并执行afterPropertiesSet函数。




## FactoryBean

> org.springframework.beans.factory.FactoryBean

工厂类接口，用户可以扩展这个类，来为要实例化的bean作一个代理，比如为该对象的所有的方法作一个拦截





## SmartInitializingSingleton

> org.springframework.beans.factory.SmartInitializingSingleton

在Spring容器管理的所有单例对象（非懒加载对象）初始化完成之后调用的回调接口。其触发时机为``BeanPostProcessor.postProcessAfterInitialization`之后。



## DisposableBean

> org.springframework.beans.factory.DisposableBean

销毁对象



## ApplicationListener

> org.springframework.context.ApplicationListener

可以监听某个事件的`event`，触发时机可以穿插在业务方法执行过程中，用户可以自定义某个业务事件。Spring内部也有一些内置事件，这种事件，可以穿插在启动调用中。我们也可以利用这个特性，来自己做一些内置事件的监听器来达到和前面一些触发点大致相同的事情。



## CommandLineRunner

> org.springframework.boot.CommandLineRunner