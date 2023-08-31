# 动态代理

## CGLIG

**Code Generation Library**

强大、高性能的代码生成库。使用ASM（字节码操作框架）在内存中**动态的生成被代理的子类**，代理类无需实现接口。

ASM：字节码操作框架，以二进制的形式修改已有类或动态生成类

前提：不能对final类以及final方法进行代理

net.sf.cglib.proxy.MethodInterceptor


## JDK

Java自带的动态代理，使用**反射原理**。代理对象必须**实现接口**。

java.lang.reflect.InvocationHandler

+ Factory

```java
public class DynamicProxyFactory<T> {

    private final Class<T> target;

    public DynamicProxyFactory(Class<T> target) {
        this.target = target;
    }

    public T newInstance() {
        Class<?>[] interfaces = target.getInterfaces();

        if (interfaces.length == 0) {
            return new CglibClient<T>(target).newInstance();
        } else {
            return new ProxyClient<T>(target).newInstance();
        }
    }

    public static void main(String[] args) {
        // 基于类
        DynamicProxyFactory<TargetObject> proxyFactory0 = new DynamicProxyFactory<>(TargetObject.class);
        proxyFactory0.newInstance();

        // 基于接口实现的类
        DynamicProxyFactory<RealSubject> proxyFactory1 = new DynamicProxyFactory<>(RealSubject.class);
        proxyFactory1.newInstance();
    }
}
```

+ CGLIB

```java
public class CglibClient<T> {

    private final Class<T> target;

    public CglibClient(Class<T> target) {
        if (target.isInterface()) {
            throw new IllegalArgumentException("target must be Class");
        }
        this.target = target;
    }

    public T newInstance(MethodInterceptor interceptor) {
        // Enhancer是CGLIB的字节码增强器，可以很方便的对类进行拓展
        Enhancer enhancer = new Enhancer();
        // 设置父类
        enhancer.setSuperclass(target);
        // 设置拦截类
        enhancer.setCallback(interceptor);
        return (T) enhancer.create();
    }
}

// 方法拦截
public class TargetInterceptor implements MethodInterceptor {

    /**
     * @param obj         目标对象
     * @param method      目标方法
     * @param args        参数
     * @param methodProxy 方法代理对象
     */
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        Object result = null;
        // 调用方法
        try {
            // 指定特定方法进行 method 进行增强操作
            if (DEFAULT_METHOD.equalsIgnoreCase(method.getName())) {
                // 通过代理子类调用父类的方法, 不要使用invoke(), 不要使用invoke(),不要使用invoke()
                result = methodProxy.invokeSuper(obj, args);
                result += " cglib";
            } else {
                // 直接放行，不做处理
                result = methodProxy.invokeSuper(obj, args);
            }
        } catch (Exception ex) {
            LOGGER.error("", ex);
        }
        return result;
    }
}
```

+ JDK

```java
public class ProxyClient<T> {

    private final Class<T> interfaceClazz;

    public ProxyClient(Class<T> interfaceClazz) {
        this.interfaceClazz = interfaceClazz;
    }

    public T newInstance() {
        final JdkInvocationHandler<T> jdkProxy = new JdkInvocationHandler<>(interfaceClazz);
        return newInstance(jdkProxy);
    }

    protected T newInstance(JdkInvocationHandler<T> jdkProxy) {
        // interfaceClazz 分为 接口和Class
        if (interfaceClazz.isInterface()) {
            // T 为接口
            return (T) Proxy.newProxyInstance(
                    interfaceClazz.getClassLoader(),
                    new Class[]{interfaceClazz},
                    jdkProxy);
        } else {
            // T 为实现类
            return (T) Proxy.newProxyInstance(
                    interfaceClazz.getClassLoader(),
                    interfaceClazz.getInterfaces(),
                    jdkProxy);
        }
    }
}

// 处理反射
public class JdkInvocationHandler<T> implements InvocationHandler {

    /**
     * 此对象为实现接口的被代理类
     */
    private Class<T> target;

    public JdkInvocationHandler(Class<T> target) {
        this.target = target;
    }

    public Class<T> getTarget() {
        return target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result;
        // 鉴于methodName
        if ("process".equals(method.getName())) {
            // 执行obj反射方法
            result = method.invoke(target.newInstance(), args);
        } else {
            result = method.invoke(target.newInstance(), args);
        }
        return result;
    }
}
```

