# Spring面试问题

## Spring容器

1. 谈谈对IoC的理解？

   Inversion of Control控制反转、依赖注入。创建对象实例交由Spring框架进行管理，由Spring框架进行对象创建和管理各实例之间的依赖关系。IOC容器实际上是一个Map。

   使用java的反射机制，运行时动态的创建和管理对象。

   

2. 手动注入方式？

   - 构造器注入：不允许创建bean对象之间的循环依赖关系

   - setter方法注入：Spring会通过默认的空参构造方法来实例化对象，没有在初始化的时候就注入，如果使用基于constructor注入，CGLIB不能创建代理；对象不能设置成final

   - 工厂方法注入

   
> 注：在Spring 4.3 以后，如果我们的类中只有单个构造函数，那么Spring就会实现一个隐式的自动注入
   

   
3. 自动注入autowiring属性模型？

   仅只对于xml配置文件

   - byType：按类型装配，可以根据属性的类型，在容器中寻找跟该类型匹配的bean。如果发现多个，那么将会抛出异常。如果没有找到，即属性值为null
   - byName：按名称装配，可以根据属性的名称，在容器中寻找跟该属性名相同的bean，如果没有找到，即属性值为null。
   - constructor：与byType的方式类似，不同之处在于它应用于构造器参数。如果在容器中没有找到与构造器参数类型一致的bean，那么将会抛出异常。
   - no
   

   
4. 基于注解的自动装配方式：@Resource 和 @Autowired？

   @Resource：默认按照name，当找不到与名称匹配的bean时按照type，可以配合@Qualifier使用指定具体名称的bean

   > 实现原理：CommonAnnotationBeanPostProcessor

   @Autowired：默认按照type，当找不到或找到多个同类型的按照name，默认情况下它要求依赖对象必须存在（可以设置它required属性为false），不能存在相同类型的bean

   > 实现原理：后置处理器AutowiredAnnotationBeanPostProcessor
   >
   > 注：@Autowired  !=  default-autowire="byType"

   

5. 注册bean的注解？

   @Component：可以用于注册所有bean

   @Repository：主要用于注册dao层的bean

   @Controller：主要用于注册控制层的bean

   @Service：主要用于注册服务层的bean

   

6. bean的配置方法？

   - 基于xml

   - 基于注解：四种注解

   - 基于java类定义：标注@Configuration注解，就可以为Spring容器提供Bean定义的信息，每个标注@Bean的类方法都相当于提供了一个Bean的定义信息。

     

7. Spring解决循环依赖问题？

   1）通过构造方法依赖注入：@Lazy

   2）通过setter方法依赖注入prototype

   3）**通过setter方法依赖注入singleton**：三级缓存

   

8. 谈谈对AOP的理解？

   Spring AOP是基于动态代理的，如果要代理的对象实现了某个接口，那么Spring AOP就会使用JDK动态代理去创建代理对象，可以强制使用CGLIB实现AOP；而对于没有实现接口的对象，就无法使用JDK动态代理，转而使用CGlib动态代理生成一个被代理对象的子类来作为代理。

   **默认使用 JDK动态代理，proxy-target-class=false**

   

9. Spring AOP 和 AspectJ AOP有什么区别？

   Spring AOP运行时增强，基于动态代理；AspectJ AOP编译时增强，基于字节码操作

   

10. Spring的bean的作用域有哪些？

    singleton

    prototype

    request

    session

    application

    websocket

    

11. 单例bean线程安全问题？

    不安全。

    1）尽量避免定义可变成员变量；

    2）使用ThreadLocal保存可变成员变量

    

12. bean的生命周期？

    1）实例化Bean：
        对于BeanFactory容器，当客户向容器请求一个尚未初始化的bean时，或初始化bean的时候需要注入另一个尚未初始化的依赖时，容器就会调用createBean进行实例化。
        对于ApplicationContext容器，当容器启动结束后，通过获取BeanDefinition对象中的信息，实例化所有的bean。

    2）设置属性（依赖注入）：
        实例化后的对象被封装在BeanWrapper对象中，紧接着，Spring根据BeanDefinition中的信息以及通过BeanWrapper提供的设置属性的接口完成属性设置与依赖注入。

    3）处理Aware接口：
        实现了BeanNameAware接口，会调用它实现的setBeanName(String beanId)方法，传入Bean的名字；
        实现了BeanClassLoaderAware接口，调用setBeanClassLoader()方法，传入ClassLoader对象的实例。
        实现了BeanFactoryAware接口，会调用它实现的setBeanFactory()方法，传递的是Spring工厂自身。
        实现了EnvironmentAware接口，会调用setEnvironment()
        实现了EmbeddedValueResolverAware接口，会调用setEmbeddedValueResolver()
        实现了ResourceLoaderAware接口，会调用setResourceLoader()
        实现了ApplicationEventPublisherAware接口，会调用setApplicationEventPublisher()
        实现了MessageSourceAware接口，会调用setMessageSource()
        实现了ApplicationContextAware接口，会调用setApplicationContext(ApplicationContext)方法，传入Spring上下文；

    4）BeanPostProcessor前置处理：
       postProcessBeforeInitialization(Object obj, String s)

    5）InitializingBean：
      初始化阶段，afeterPropertiesSet() 

    6）init-method：自定义初始化方法

    7）BeanPostProcessor后置处理：
      postProcessAfterInitialization(Object obj, String s)方法；由于这个方法是在Bean初始化结束时调用的，所以可以被应用于内存或缓存技术 

    **Bean创建完成，可以使用** 

    8）DisposableBean：
      清理阶段destroy() 

    9）destroy-method：


13. 启动Bean加载过程？

    

14. Spring中使用了哪些设计模式？举例？

    （1）工厂模式：Spring使用工厂模式，通过BeanFactory和ApplicationContext来创建对象

    （2）单例模式：Bean默认为单例模式

    （3）策略模式：例如Resource的实现类，针对不同的资源文件，实现了不同方式的资源获取策略

    （4）代理模式：Spring的AOP功能用到了JDK的动态代理和CGLIB字节码生成技术

    （5）模板方法：可以将相同部分的代码放在父类中，而将不同的代码放入不同的子类中，用来解决代码重复的问题。比如RestTemplate, JmsTemplate, JpaTemplate

    （6）适配器模式：Spring AOP的增强或通知（Advice）使用到了适配器模式，Spring MVC中也是用到了适配器模式适配Controller

    （7）观察者模式：Spring事件驱动模型就是观察者模式的一个经典应用。

    （8）桥接模式：可以根据客户的需求能够动态切换不同的数据源。比如我们的项目需要连接多个数据库，客户在每次访问中根据需要会去访问不同的数据库
    
15. @Component 和 @Bean的区别？

    1）使用对象不同：@Component作用于类，@Bean作用于方法

    2）@Component通过类扫描自动装配到Spring容器，



15. BeanFactory 和 FactoryBean的区别？

    BeanFactory ：管理bean的工厂类，

    FactoryBean：是一个工厂bean，工厂类接口，实现该接口可以改变bean，例如:SqlSessionFactoryBean

    

16. Beanfactory和ApplicationContext区别？

    Spring的容器

    ApplicationContext是BeanFactory的子类，额外提供了：

    1）继承MessageSource，支持国际化

    2）资源文件访问

    3）载入多个上下文

    4）提供在监听器中注册bean的事件（ApplicationEventPublisher）

    BeanFactory采用延迟加载注入bean，只有在使用某个bean时，才会进行加载实例化；缺点：不能提前发现配置问题

    ApplicationContext在容器时就创建了所有的bean，

    都支持BeanPostProcessor、BeanFactoryPostProcessor，但是BeanFactory需要手动注册，而ApplicationContext则是自动注册

    BeanFactory以编程方式创建，ApplicationContext以声明方式创建，如使用ContextLoader

17. 

    

## Spring事务

1. Spring事务原理？

   AOP，切面增强。

   生成代理对象

   

2. Spring事务实现方式有哪几种？

   编程式事务：使用TransactionTemplate或者直接使用PlatformTransactionManager，Spring推荐使用TransactionTemplate

   声明式事务：基于XML的声明式事务和基于注解的声明式事务，建立在AOP基础上

   

3. Spring事务的隔离级别有哪几种？

   - DEFAULT：使用后端数据库默认的隔离级别，Mysql默认采用的REPEATABLE_READ隔离级别；Oracle默认采用的READ_COMMITTED隔离级别。
   - READ_UNCOMMITTED：最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读。
   - READ_COMMITTED：读已提交，可以阻止脏读，但是幻读或不可重复读仍有可能发生（Oracle）
   - REPEATABLE_READ：可重复度，可以阻止脏读和不可重复读，但幻读仍有可能发生。（MySQL）
   - SERIALIZABLE：最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

   

4. Spring事务的传播机制有哪几种？

   支持当前事务的情况：

   - REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
   - SUPPORTS： 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
   - MANDATORY： 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。

   不支持当前事务的情况：

   - REQUIRES_NEW： 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
   - NOT_SUPPORTED： 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
   - NEVER： 以非事务方式运行，如果当前存在事务，则抛出异常。

   其他情况：

   - NESTED： 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行，外围父事务回滚，子事务一定回滚，而内部子事务可以单独回滚而不影响外围父事务和其它子事务；如果当前没有事务，则等价于REQUIRED。

区别和联系：

+ `NESTED`和`REQUIRED`修饰的内部方法都属于外围方法事务，如果外围方法抛出异常，这两种方法的事务都会被回滚。但是`REQUIRED`是加入外围方法事务，所以和外围事务同属于一个事务，一旦`REQUIRED`事务抛出异常被回滚，外围方法事务也将被回滚。
而`NESTED`是外围方法的子事务，有单独的保存点，所以`NESTED`方法抛出异常被回滚，不会影响到外围方法的父事务

+ `NESTED`和`REQUIRES_NEW`都可以做到内部方法事务回滚而不影响外围方法事务。但是因为`NESTED`是嵌套事务（称之为一个子事务），所以外围方法父事务回滚之后，作为外围方法事务的子事务也会被回滚。而`REQUIRES_NEW`是通过开启新的事务实现的，与原有事务无关，内部事务和外围事务是两个事务，外围事务回滚不会影响内部事务。

+ 简单来说，对于`REQUIRES_NEW`，儿子犯错，老子无恙，老子犯错，儿子无恙；对于`REQUIRED`，儿子犯错，老子牵连，老子犯错，儿子牵连；对于`NESTED`，儿子犯错，老子无恙，老子犯错，儿子牵连

  

4. Spring事务失效？
   1. 自调用，this对象不是代理类，而是对象本身
   2. 方法不是public，可以开启AspectJ代理模式
   3. 数据库本身就不支持事务
   4. 没有被spring管理
   5. 异常没有被抛出，事务也就不会回滚

## Spring MVC

1. Spring MVC的工作原理？