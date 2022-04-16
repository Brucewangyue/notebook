[s中文文档地址](https://github.com/DocsHome/spring-docs/blob/master/SUMMARY.md)

[TOC]

**问题：** 

3. Spring AOP的底层实现原理
4. Spring的事务是如何回滚的
5. 谈一下Spring的事务传播
8. 自动装配原理



## Spring 有哪些模块

![image-20220406210440572](assets\image-20220406210440572.png)



## 描述下BeanFactory

BeanFactory是bean工厂，spring的顶级核心接口，工厂只负责按照要求生产bean，bean的定义信息，要生产成什么样，由下家ApplicationContext决定



## BeanFactory和ApplicationContext的区别

相当于工厂和4S店的区别

ApplicationContext面向的是用户，所以需要提供更好的服务，不仅由BeanFactory生产Bean的功能，还提供了国际化、加载Bean定义、监听器等



## BeanFactory和FactoryBean有什么区别

BeanFactory是spring的顶级接口，是生产Bean的工厂

FactoryBean也是一个接口，继承他的Bean会成为一个特殊的Bean，通过BeanFactory生产这类Bean的时候，每次都是调用FactoryBean的getObject方法返回，也可以通过特殊的一个字符串让BeanFactory返回FactoryBean本身



## 谈谈Spring IOC的理解，原理和实现

1. 资源集中管理
2. 降低使用资源双方的依赖程度，也就是低耦合

控制反转ioc：原来对象是由使用者自己控制管理，现在是把对象交给第三方系统(spring)来管理

依赖注入di：通过把对应属性的值注入到具体对象中，populateBean方法完成对象属性的注入

容器：通过使用map存储对象，是spring的三级缓存中的 singletonObjects map对象



## 简述SpringIoC的加载过程







## @Scope指定的作用域方法取值

- a) singleton 单实例的(默认) 
- b) prototype 多实例的 
- c) request 同一次请求 
- d) session 同一个会话级别



## 什么是bean的生命周期

1. 实例化Bean对象，这个时候Bean的对象是非常低级的，基本不能够被我们使用，因为连最基本的属性都没有设置，可以理解为连Autowired注解都是没有解析的； 
2. 填充属性，当做完这一步，Bean对象基本是完整的了，可以理解为Autowired注解已经解析完毕，依赖注入完成了； 
3. 如果Bean实现了BeanNameAware接口，则调用setBeanName方法； 
4. 如果Bean实现了BeanClassLoaderAware接口，则调用setBeanClassLoader方法； 
5.  如果Bean实现了BeanFactoryAware接口，则调用setBeanFactory方法； 
6. 调用BeanPostProcessor的postProcessBeforeInitialization方法； 
7. 如果Bean实现了InitializingBean接口，调用afterPropertiesSet方法； 
8. 如果Bean定义了init-method方法，则调用Bean的init-method方法； 
9.  调用BeanPostProcessor的postProcessAfterInitialization方法；当进行到这一步，Bean已经被准备就绪了，一直停留在应用的上下文中，直到被销毁； 
10. 如果应用的上下文被销毁了，如果Bean实现了DisposableBean接口，则调用destroy方法，如果Bean定义了destory-method声明了销毁方法也会被调用

**方式一：**由容器管理Bean的生命周期，我们可以通过自己指定bean的初始化方法和bean的销毁方法

```java
@Configuration 
public class MainConfig { 
    //指定了bean的生命周期的初始化方法和销毁方法.
    @Bean(initMethod = "init",destroyMethod = "destroy") 
    public Car car() { 
        return new Car(); 
    } 
}
```

- 单实例bean，容器启动的时候，bean的对象就创建了，而且容器销毁的时候，也会调用Bean的销毁方法 
- 多实例bean, 容器启动的时候，bean是不会被创建的而是在获取bean的时候被创建，而且bean的销毁不受IOC容器的管理

**方式二：**通过 InitializingBean和DisposableBean 的二个接口实现bean的初始化以及销毁方法 

```java
@Component 
public class Person implements InitializingBean,DisposableBean { 
    public Person() { 
        System.out.println("Person的构造方法"); 
    }
    @Override 
    public void destroy() throws Exception { 
        System.out.println("DisposableBean的destroy()方法 "); 
    }
    @Override 
    public void afterPropertiesSet() throws Exception { 
        System.out.println("InitializingBean的 afterPropertiesSet方法"); 
    } 
}
```

**方式三：**通过**JSR250**规范 提供的注解**@PostConstruct **和**@ProDestory**标注的方法

```java
@Component 
public class Book { 
    public Book() { 
        System.out.println("book 的构造方法"); 
    }
    @PostConstruct 
    public void init() { 
        System.out.println("book 的PostConstruct标志的方法"); 
    }
    @PreDestroy 
    public void destory() { 
        System.out.println("book 的PreDestory标注的方法"); 
    } 
}
```

**方式四：**通过Spring的BeanPostProcessor的 bean的后置处理器会拦截所有bean创建过程

- postProcessBeforeInitialization：在init方法之前调用 
- postProcessAfterInitialization：在init方法之后调用

```java
@Component 
public class MyBeanPostProcessor implements BeanPostProcessor { 
    @Override 
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException { 
        System.out.println("MyBeanPostProcessor...postProcessBeforeInitialization:"+beanName); 
        return bean; 
    }
    @Override 
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException { 
        System.out.println("MyBeanPostProcessor...postProcessAfterInitialization:"+beanName); 
        return bean; 
    } 
}
```

**BeanPostProcessor的执行时机**

```java
populateBean(beanName, mbd, instanceWrapper);
initializeBean{ 
    applyBeanPostProcessorsBeforeInitialization();
    invokeInitMethods{ 
        isInitializingBean.afterPropertiesSet 
        自定义的init方法 
    }
    applyBeanPostProcessorsAfterInitialization()
}
```



## @Autowired和@Resouce的区别

| 区别         | @Autowired                                                   | @Resource  |
| ------------ | ------------------------------------------------------------ | ---------- |
| 自动装配顺序 | 首先时按照类型进行装配，若在IOC容器中发现了多个相同类型的组件，那么就按照属性名称来进行装配 | 按名称装配 |
| 扩展功能     | 支持 @Primary 和 @Qualifier                                  | 无         |
|              |                                                              |            |



## 如何在自己的组件中使用spring ioc的底层组件

我们可以通过实现XXXAware接口来实现

```java
@Component 
public class TulingCompent implements ApplicationContextAware,BeanNameAware { 
    // 注入applicationContext
    private ApplicationContext applicationContext; 
    @Override 
    public void setBeanName(String name) { 
        System.out.println("current bean name is :【"+name+"】"); 
    }
    
    @Override 
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException { 
        this.applicationContext = applicationContext; 
    } 
}
```



## spring中使用了哪些设计模式

[spring设计模式总结](spring设计模式总结.md)



## Spring是如何解决循环依赖问题

[spring循环依赖问题](spring循环依赖问题.md)



## Spring中有哪些扩展接口及调用时机

BeanFactoryPostProcessor

BeanPostProcessor

ApplicationContextAware, BeanNameAware, BeanClassLoaderAware

InitializingBean,DisposableBean

init-method



## spring事件机制

[spring事件原理](spring事件原理.md)