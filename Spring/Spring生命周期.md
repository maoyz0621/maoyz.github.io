# Bean生命周期

|                                          |
| :--------------------------------------: |
| ![Bean生命周期](\image\Bean生命周期.jpg) |

## ApplicationContextInitializer

> org.springframework.context.ApplicationContextInitializer

Spring容器刷新之前初始化`ConfigurableApplicationContext`的回调接口，容器刷新之前调用此类的`initialize`()方法。

可以在整个spring容器还没被初始化之前做一些事情。在最开始激活一些配置，或者利用这时候class还没被类加载器加载的时机，进行动态字节码注入等操作

源码：

|                                                             |
| ----------------------------------------------------------- |
| ![Bean生命周期](\image\ApplicationContextInitializer-1.png) |
| ![Bean生命周期](\image\ApplicationContextInitializer-2.png) |
| ![Bean生命周期](\image\ApplicationContextInitializer-3.png) |

扩展的生效，有以下三种方式：

- 在启动类中用`springApplication.addInitializers(new TestApplicationContextInitializer())`语句加入
- 配置文件配置`context.initializer.classes=com.example.demo.TestApplicationContextInitializer`
- Spring SPI扩展，在spring.factories中加入`org.springframework.context.ApplicationContextInitializer=com.example.demo.TestApplicationContextInitializer`



## BeanDefinitionRegistryPostProcessor

> org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor

在读取项目中的`beanDefinition`之后执行，提供一个补充的扩展点

使用场景：你可以在这里动态注册自己的`beanDefinition`，可以加载classpath之外的bean