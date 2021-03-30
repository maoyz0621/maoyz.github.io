# Mybatis插件原理

## 基于拦截器

|                         |
| :---------------------- |
| ![](.\image\拦截器.png) |



- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)

  拦截执行器的方法

- ParameterHandler (getParameterObject, setParameters)

  请求参数处理

- ResultSetHandler (handleResultSets, handleOutputParameters)

  返回结果集处理

- StatementHandler (prepare, parameterize, batch, update, query)

  Sql语法构建的处理
  
  
  
## 拦截器注解规则

```java
@Intercepts({
        @Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class}),
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
implements Interceptor {
    
    // 处理拦截到的对象
	@Override
    public Object intercept(Invocation invocation) throws Throwable {
    	Object val = invocation.proceed();
        return val;
    }

    @Override
    public Object plugin(Object target) {
        // 要判断一下目标类型，是本插件要拦截的对象时才执行Plugin.wrap方法，否则的话，直接返回目标本身。
		if (target instanceof Executor) {
            return Plugin.wrap(target, this);
        }
        return target;    }

    @Override
    public void setProperties(Properties properties) {
    }
    
}
```

1. @Intercepts：标识该类是一个拦截器
2. @Signature指指明自定义拦截器需要拦截哪一个类型，哪一个方法；

- type： 对应Executor,ParameterHandler,ResultSetHandler,StatementHandler中的一种
- method：对应type指定类中的方法（因为可能存在重载方法）
- args：对应method方法中的参数值

**`需要注意的是：每经过一个拦截器对象都会调用插件的plugin方法，也就是说，该方法会调用4次。根据@Intercepts注解来决定是否进行拦截处理。`**

```java
public class Invocation {
  // 被代理对象
  private final Object target;
  // 代理方法
  private final Method method;
  // 方法参数
  private final Object[] args;
    
  public Invocation(Object target, Method method, Object[] args) {
    this.target = target;
    this.method = method;
    this.args = args;
  }
  ...
  public Object proceed() throws InvocationTargetException, IllegalAccessException {
    return method.invoke(target, args);
  }
}
```

invocation.getArgs() 与 @Signature中args参数对应

> 问题1：`Plugin.wrap(target, this)`方法的作用？

判断是否拦截这个类型对象（根据@Intercepts注解决定），然后决定是返回一个代理对象还是返回原对象。



> 问题2：拦截器代理对象可能经过多层代理，如何获取到真实的拦截器对象？



## 源码分析

拦截器使用了责任链模式+动态代理+反射机制

|                         |
| :---------------------: |
| ![](.\image\插件_1.png) |
| ![](.\image\插件_2.png) |

```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}
```



**Plugin动态代理**

|                               |
| :---------------------------: |
| ![](.\image\插件执行时机.png) |



```java
public class Plugin implements InvocationHandler {

  private final Object target;
  private final Interceptor interceptor;
  private final Map<Class<?>, Set<Method>> signatureMap;

  private Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }

  // 包装插件
  public static Object wrap(Object target, Interceptor interceptor) {
    // 从拦截器的注解中获取拦截的类名和方法信息
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    // 取得要改变行为的类(ParameterHandler|ResultSetHandler|StatementHandler|Executor)
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    // 产生代理，@Intercepts注解的接口的实现类才会产生代理
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(type.getClassLoader(), interfaces, new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      // 获取需要拦截的方法
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      // Interceptor实现类注解的方法才会拦截处理
      if (methods != null && methods.contains(method)) {
        // 执行拦截器的拦截方法
        return interceptor.intercept(new Invocation(target, method, args));
      }
      // 执行原来方法
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

  // 取得签名Map,就是获取Interceptor实现类上面的注解，要拦截的是那个类（Executor ,ParameterHandler， ResultSetHandler，StatementHandler）的那个方法 
  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());
    }
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.computeIfAbsent(sig.type(), k -> new HashSet<>());
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
      }
    }
    return signatureMap;
  }

  // 取得接口
  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }

}
```

