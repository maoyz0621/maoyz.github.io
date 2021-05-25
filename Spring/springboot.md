# SpringBoot

## 注解

## @SpringBootConfiguration

- @Configuration
- @EnableAutoConfiguration
- @ComponentScan

### @EnableXxx

开启Xxx服务，配合@Import注解以及定义好的JavaConfig配置类，加在工程启动类上

### @ConditionalOnXxx注解

>使用范围

- `@Conditional`  (只有满足一些列条件之后创建一个bean) 标注在类上面，表示该类下面的所有@Bean都会启用配置，也可以标注在方法上面，只是对该方法启用配置。

- `@ConditionalOnBean`  (仅仅在当前上下文中存在某个对象时，才会实例化一个Bean)在判断list的时候，如果list没有值，返回false，否则返回true

- `@ConditionalOnMissingBean`  (仅仅在当前上下文中不存在某个对象时，才会实例化一个Bean) 在判断list的时候，如果list没有值，返回true，否则返回false，其他逻辑都一样

- `@ConditionalOnExpression`  (当表达式为true的时候，才会实例化一个Bean)

- `@ConditionalOnClass`  (某个class位于类路径上，才会实例化一个Bean)

- `@ConditionalOnMissingClass`  (某个class类路径上不存在的时候，才会实例化一个Bean)

- `@ConditionalOnNotWebApplication`（不是web应用）

- `@ConditionalOnProperty(name = "swagger.enabled", matchIfMissing = true)`

  

## SpringFactoriesLoader

Spring提供的SPI实现

从指定的配置文件META-INF/spring.factories加载配置，即根据@EnableAutoConfiguration的完整类名org.springframework.boot.autoconfigure.EnableAutoConfiguration作为查找的Key，获取对应的一组@Configuration类



## Environment.resolveRequiredPlaceholders(key)

获取application.properties的配置内容
```java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface RocketMQMessageListener {
        String NAME_SERVER_PLACEHOLDER = "${rocketmq.name-server:}";
    }

    获取属性值:
    applicationContext.getEnvironment().
                      resolveRequiredPlaceholders(rocketMQMessageListener.customizedTraceTopic())
```

## 引用外部属性文件

`@ImportResource(locations = {"classpath:other.properties"})`

## 注解@Cacheable 和@Transactional 失效原因

在有些情形下注解式缓存@Cacheable和事务@Transactional是不起作用的：例如同一个bean内部方法调用，子类调用父类中有缓存注解的方法等。因为注解调用走的都是增强代理类; 后者不起作用是因为缓存切面必须走代理才有效，这时可以手动使用CacheManager来获得缓存效果。

```java
    @Transactional(rollbackFor = Exception.class)
    @Override
    public void save() {
        userRepository.deleteById(1L);
        int i = 1 / 0;
        UserPO userPO = new UserPO("aaaa", "1");
    }

    @Override
    public void execute() {
        // 相当于this.save(),this并不是增强代理类,事务无效
        save();
    }
```

解决办法一:ApplicationContext
```java
    @Component
    public class ApplicationContextHolder implements ApplicationContextAware {

        protected static final Logger logger = LoggerFactory.getLogger(ApplicationContextHolder.class);

        private static ApplicationContext applicationContext;

        public static ApplicationContext getContext() {
            return applicationContext;
        }

        @Override
        public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
            ApplicationContextHolder.applicationContext = applicationContext;
        }
    }

    /**
     * 内部调用,如果直接使用getMenuList()，相当于this.getMenuList(),不走增强代理
     * <p>
     * 方法一 ApplicationContext
     */
    public List<String> getRecommendsWithContext() {
        MenuService proxy = ApplicationContextHolder.getContext().getBean(getClass());
        return proxy.save();
    }
```

解决方法二:AopContext
```java

    /**
     * 方法二 AopContext.currentProxy()
     */
    public List<String> getRecommendsWithAopContext() {
        // java.lang.IllegalStateException: Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true' to make it available.
        // 解决办法 @EnableAspectJAutoProxy(proxyTargetClass = true, exposeProxy = true),默认值都是false
        return ((MenuService) AopContext.currentProxy()).getMenuList();
    }
```

## 自动配置

注解 @EnableAutoConfiguration, @Configuration, @ConditionalOnClass 就是自动配置的核心，首先它得是一个配置文件，其次根据类路径下是否有这个类去自动配置

自动装配在springboot-autoconfigure工程，/META-INF/spring.factories中获取EnableAutoConfiguration指定的值

#### 监视器

