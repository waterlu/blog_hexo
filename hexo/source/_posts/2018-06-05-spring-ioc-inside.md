---
title: Spring-IoC原理
date: 2018-06-05 09:24:09
categories: Spring
tags: [Spring, IoC]
toc: true
description: 理解了Bean的生命周期就掌握了IoC的原理，源码分析是对Bean生命周期的代码化深入理解。
comments: true
---

## IoC原理

### Bean生命周期

首先，IoC容器从XML文件或者Java类中读取Bean对象的定义信息，封装成BeanDefinition，并注册到BeanDifinitionRegistry中。

然后，当IoC容器接收到getBean()方法请求时，触发Bean对象的实例化操作。当然，单例模式下如果Bean对象已经存在就直接返回。

在实例化Bean对象之前，回调BeanFactoryPostProcessor，可以对BeanDefinition中保存的信息进行修改，比如修改某些属性值。

![](/images/spring-ioc-inside-beanlife.png)



1. 通过反射机制实例化Bean对象；
2. 如果Bean对象实现了BeanNameAware接口，则回调setBeanName()方法，传入该bean的id；
3. 如果Bean对象实现了BeanFactoryAware接口，则回调setBeanFactory()方法，传入该Bean的BeanFactory；
4. 如果Bean对象实现了ApplicationContextAware接口，则回调setApplicationContext()方法，传入该Bean的ApplicationContext；
5. 如果有一个Bean实现了BeanPostProcessor接口，则回调postProcessBeforeInitialization()方法；
6. 如果Bean对象实现了InitializingBean接口，则回调afterPropertiesSet()方法；
7. 如果Bean对象配置了init-method方法，则会执行init-method配置的方法；
8. 如果有一个Bean实现了BeanPostProcessor接口，则回调postProcessAfterInitialization()方法；

以下为容器关闭后执行的方法，通常IoC容器一直运行，不会执行

1. 如果Bean对象实现了DisposableBean接口，则回调destroy()方法；
2. 如果Bean对象配置了destroy-method方法，则会执行destroy-method配置的方法。

总结一下：

- 首先回调BeanFactoryPostProcessor，此时可以修改BeanDefinition；
- 实例化Bean对象；
- 回调Bean对象的ApplicationContextAware，返回ApplicationContext；
- 回调BeanPostProcessor的postProcessBeforeInitialization()方法；
- 回调Bean对象的InitializingBean，做后初始化操作；
- 回调Bean对象的init-method方法，做后初始化操作；
- 回调BeanPostProcessor的postProcessAfterInitialization()方法，此时可以修改Bean对象；
- 返回Bean对象。

### BeanFactory和ApplicationContext

BeanFactory和ApplicationContext都可以认为是IoC容器，都提供了getBean()方法。BeanFactory是IoC最基础的接口，getBean()方法就是在BeanFactory接口中声明的。

```java
package org.springframework.beans.factory;
public interface BeanFactory {
  Object getBean(String name) throws BeansException;
}
```

ApplicationContext继承自BeanFactory，提供了更多功能。

```java
public interface HierarchicalBeanFactory extends BeanFactory {
}
```

```java
package org.springframework.context;
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, 
  HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
}
```

严格意义上BeanFactory是容器，ApplicationContext是应用上下文，由于几乎所有的应用场合都直接使用ApplicationContext而非底层的BeanFactory，所以一般我也把ApplicationContext成为IoC容器。

- 默认初始化所有的Singleton，也可以通过配置取消预初始化（lazy-init）；
- 继承MessageSource，因此支持国际化；
- 资源访问，比如访问URL和文件；
- 事件机制；
- 同时加载多个配置文件；
- 以声明式方式启动并创建Spring容器。

## IoC源码分析

> TODO

入口是AbstractApplicationContext的refresh()方法，我们从这里开始。

```java
package org.springframework.context.support;
public abstract class AbstractApplicationContext {
  public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
      // 创建BeanFactory
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Allows post-processing of the bean factory in context subclasses.
      postProcessBeanFactory(beanFactory);

      // Invoke factory processors registered as beans in the context.
      invokeBeanFactoryPostProcessors(beanFactory);

      // Register bean processors that intercept bean creation.
      registerBeanPostProcessors(beanFactory);

      // 实例化Bean
      finishBeanFactoryInitialization(beanFactory);
    }
  }
}
```

最重要的两个方法是obtainFreshBeanFactory()和finishBeanFactoryInitialization()，前者读取Bean相关配置信息，后者创建Bean实例。

```java
package org.springframework.context.support;
public abstract class AbstractApplicationContext {
  protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return this.beanFactory;
  }
}
```

```java
package org.springframework.context.support;
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
  @Override
  protected final void refreshBeanFactory() throws BeansException {
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    loadBeanDefinitions(beanFactory);
    synchronized (this.beanFactoryMonitor) {
      this.beanFactory = beanFactory;
    }
  }
}
```

```java
package org.springframework.beans.factory.support;
public class DefaultListableBeanFactory extends AbstractBeanFactory {
  private final Map<String, BeanDefinition> beanDefinitionMap = new 
    ConcurrentHashMap<String, BeanDefinition>(256);
}
```

以XML配置为例，AbstractApplicationContext类的子类AbstractXmlApplicationContext实现了loadBeanDefinitions()方法，完成加载和解析XML文件的操作，并将BeanDefinition信息放入到BeanFactory中。其中，BeanDefinition与XML文件中的一个`<bean />` 相对应，BeanFactory接口的默认实现类是DefaultListableBeanFactory，可以看到DefaultListableBeanFactory中使用ConcurrentHashMap来保存Bean信息。

这就完成了第一步工作，解析XML文件读取配置信息到BeanFactory的Map中保存。

接下来看看如何根据上面的信息实例化对象。

```java
package org.springframework.context.support;
public abstract class AbstractApplicationContext {
  protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);
    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();
    // Instantiate all remaining (non-lazy-init) singletons.
    beanFactory.preInstantiateSingletons();
  }
```

AbstractApplicationContext调用BeanFactory的方法来实例化对象。

```java
package org.springframework.beans.factory.support;
public class DefaultListableBeanFactory extends AbstractBeanFactory {	
  @Override
  public void preInstantiateSingletons() throws BeansException {
    List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);
    for (String beanName : beanNames) {
      getBean(beanName);
    }
  }
  @Override
  public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
  }
}
```

```java
package org.springframework.beans.factory.support;
public abstract class AbstractBeanFactory implements BeanFactory {
  
  protected <T> T doGetBean(final String name, final Class<T> requiredType, 
                            final Object[] args, boolean typeCheckOnly) {
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    if (mbd.isSingleton()) {
      sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
        @Override
        public Object getObject() throws BeansException {
          try {
            return createBean(beanName, mbd, args);
          }
          catch (BeansException ex) {
            destroySingleton(beanName);
            throw ex;
          }
        }
      });
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    } else if (mbd.isPrototype()) {
      // It's a prototype -> create a new instance.
      Object prototypeInstance = null;
      try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
      }
      finally {
        afterPrototypeCreation(beanName);
      }
      bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
    }
    return (T) bean;
  }
}
```

DefaultListableBeanFactory类的getBean()方法会调用AbstractBeanFactory类的doGetBean()方法，里面通过createBean()方法创建实例对象。

```java
package org.springframework.beans.factory.support;
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory {
  @Override
  protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) {
	Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    return beanInstance;
  }
  protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, 
                                final Object[] args) {
      BeanWrapper instanceWrapper = null;
    instanceWrapper = createBeanInstance(beanName, mbd, args);
    final Object bean = instanceWrapper.getWrappedInstance();
  }
  protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd,
                                           Object[] args) {
    
  }
  protected BeanWrapper instantiateBean(final String beanName, 
                                        final RootBeanDefinition mbd) {
      
  }
}
```

AbstractAutowireCapableBeanFactory类的createBean()方法



```java
package org.springframework.beans.factory.support;
public class SimpleInstantiationStrategy implements InstantiationStrategy {
  	@Override
	public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
      Constructor<?> ctor =	clazz.getDeclaredConstructor((Class[]) null);
      ctor.setAccessible(true);
      return ctor.newInstance(null);
    }
}
```





```java
package org.springframework.beans.factory.support;
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory {
  protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    applyPropertyValues(beanName, mbd, bw, pvs);
  }
  protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw,
                                     PropertyValues pvs) {
    MutablePropertyValues mpvs = null;
    List<PropertyValue> original;
    for (PropertyValue pv : original) {
      
    }
    bw.setPropertyValues(new MutablePropertyValues(deepCopy));
  }
}

```



```java
package org.springframework.beans;
public class BeanWrapperImpl implements BeanWrapper {
  private class BeanPropertyHandler extends PropertyHandler {
    private final PropertyDescriptor pd;
    @Override
    public void setValue(final Object object, Object valueToApply) throws Exception {
      final Method writeMethod = (this.pd instanceof GenericTypeAwarePropertyDescriptor ?
                                  ((GenericTypeAwarePropertyDescriptor) this.pd).getWriteMethodForActualAccess() :
                                  this.pd.getWriteMethod());
      if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers()) && !writeMethod.isAccessible()) {
        if (System.getSecurityManager() != null) {
          AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
              writeMethod.setAccessible(true);
              return null;
            }
          });
        }
        else {
          writeMethod.setAccessible(true);
        }
      }
      final Object value = valueToApply;
      if (System.getSecurityManager() != null) {
        try {
          AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
            @Override
            public Object run() throws Exception {
              writeMethod.invoke(object, value);
              return null;
            }
          }, acc);
        }
        catch (PrivilegedActionException ex) {
          throw ex.getException();
        }
      }
      else {
        writeMethod.invoke(getWrappedInstance(), value);
      }
    }      
  }
}
```



[bean作用域：理解Bean生命周期](https://blog.csdn.net/soonfly/article/details/69480058)

