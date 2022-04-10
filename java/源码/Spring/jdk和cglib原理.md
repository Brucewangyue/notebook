## JDK动态代理

实例代码：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class JdkProxyDemo {
    public static void main(String[] args) {
        // 将生成的动态代理存为文件
        System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
        Calculator proxy = getProxy(new MyCalculator());
        proxy.add(1,1);
        System.out.println(proxy.getClass());
    }

    public static Calculator getProxy(Calculator calculator){
        ClassLoader classLoader = calculator.getClass().getClassLoader();
        Class<?>[] interfaces = calculator.getClass().getInterfaces();
        InvocationHandler invocationHandler = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Object result = null;
                System.out.println("执行add方法前");
                result = method.invoke(calculator,args);
                return result;
            }
        };

        // 调试入口
        Object proxy = Proxy.newProxyInstance(classLoader,interfaces,invocationHandler);
        return (Calculator) proxy;
    }

    private interface Calculator{
        int add(int a, int b);
    }

    private static class MyCalculator implements  Calculator{

        @Override
        public int add(int a, int b) {
            return a + b;
        }
    }
}

```

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    Objects.requireNonNull(h);
    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }
    
    // 创建代理
    Class<?> cl = getProxyClass0(loader, intfs);
    /*
     * Invoke its constructor with the designated invocation handler.
     */
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    // 一次最多传入2个字节数量的接口
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    // 缓存中是否有代理，没有就要创建
    // Supplier BiFunction
    return proxyClassCache.get(loader, interfaces);
}
```



## Cglib动态代理

