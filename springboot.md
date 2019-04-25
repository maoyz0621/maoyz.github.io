###### 一、@ConditionalOnXxx注解

>使用范围

>`@Conditional`  (只有满足一些列条件之后创建一个bean) 标注在类上面，表示该类下面的所有@Bean都会启用配置，也可以标注在方法上面，只是对该方法启用配置。

>`@ConditionalOnBean`  (仅仅在当前上下文中存在某个对象时，才会实例化一个Bean)在判断list的时候，如果list没有值，返回false，否则返回true

>`@ConditionalOnMissingBean`  (仅仅在当前上下文中不存在某个对象时，才会实例化一个Bean) 在判断list的时候，如果list没有值，返回true，否则返回false，其他逻辑都一样

>`@ConditionalOnExpression`  (当表达式为true的时候，才会实例化一个Bean)

>`@ConditionalOnClass`  (某个class位于类路径上，才会实例化一个Bean)

>`@ConditionalOnMissingClass`  (某个class类路径上不存在的时候，才会实例化一个Bean)

>`@ConditionalOnNotWebApplication`（不是web应用）
