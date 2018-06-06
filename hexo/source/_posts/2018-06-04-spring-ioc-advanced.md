---
title: Spring-IoC进阶
date: 2018-06-04 10:34:20
categories: Spring
tags: [Spring, IoC]
toc: true
description: 本文讨论IoC的进阶使用，包括：ApplicationContextAware、BeanPostProcessor、BeanFactoryPostProcessor、FactoryBean等深度定制的用法。
comments: true
---

## IoC使用进阶

本文讨论Spring IoC高级一点的用法，主要是一些深入定制化操作。

### FactoryBean

我们之前的例子都是一个Bean对应一个对象，像下面这样

```xml
<beans>
  <bean id="wheel" class="cn.lu.spring.ioc.Wheel" />
</beans>
```

但是像Connection这样的对象，我们肯定希望是连接池模式，可是默认scope是`singleton` 只能使用一个连接，如果修改scop为 `prototype` 每次都创建一个新的连接，两者不能满足要求，那么如何创建对象池呢？

这个时候可以使用FactoryBean，通过FactoryBean的getOjbect()方法返回对象，进行对象池管理。

```java
package org.springframework.beans.factory;
public interface FactoryBean<T> {
  // 返回对象
  T getObject() throws Exception;
  // 返回对象类型
  Class<?> getObjectType();
  // 是否单例
  boolean isSingleton();
}
```

定义自己的Factory类

```java
public class WheelFactory implements FactoryBean<Wheel> {
  @Override
  public Wheel getObject() throws Exception {
    // 这里可以写自己的对象池逻辑
  }  
}
```

然后修改xml配置

```xml
<beans>
  <bean id="wheel" class="cn.lu.spring.ioc.WheelFactory" />
</beans>
```

因为WheelFactory实现了FactoryBean接口，所以IoC容器不会返回WheelFactory类的实例对象，而是调用其getObject()方法返回真正的对象。

上面是通过XML配置FactoryBean，如何通过Java配置呢？

```java
@Configuration
public class Config {
  @Bean
  public WheelFactory wheelFactory() {
    return new WheelFactory();
  }
}
```

定义@Bean时返回WheelFactory，使用是正常@Autowired即可，实际返回Wheel对象。

```java
@Autowired
private Wheel wheel;
```

### ApplicationContextAware

通常都是Spring自动帮我注入Bean对象，但有时我们也需要主动从IoC容器获取Bean对象，例如：查找声明了某个注解的Bean对象，这个时候就需要用到ApplicationContextAware了。实现ApplicationContextAware接口的setApplicationContext()方法就可以获得Spring的IoC容器（对应ApplicationContext类），我们看一下ApplicationContext类都提供了哪些方法：

```java
package org.springframework.context;
// 这里省略了其他基类
public interface ApplicationContext extends ListableBeanFactory { 
  String getId();
  String getApplicationName();
}
```

ApplicationContext类没啥干货，再看看基类ListableBeanFactory提供了哪些方法

```java
package org.springframework.beans.factory;
public interface ListableBeanFactory extends BeanFactory {
  // 检查容器中是否包含beanName这个Bean对象
  boolean containsBeanDefinition(String beanName);
  // 返回容器中定义的Bean的数量
  int getBeanDefinitionCount();
  // 返回容器中定义的所有Bean的名字
  String[] getBeanDefinitionNames();
  // 返回容器中所有type类型的Bean
  <T> Map<String, T> getBeansOfType(Class<T> type) throws BeansException;
  // 返回容器中所有含有annotationType注解的Bean
  Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) 
    throws BeansException;
}
```

最常用的应该是getBeansOfType()和getBeansWithAnnotation()，根据类型或者注解获取Bean对象。

下面以解析自定义注解@RocketMQProduer为例说明ApplicationContextAware的用法：

- 得到applicationContext后可以调用getBeansWithAnnotation()方法获取所有声明了RocketMQProducer注解的Bean对象。

```java
public class MQProducerAutoConfiguration implements ApplicationContextAware {
  protected ApplicationContext applicationContext;
  @Override
  public void setApplicationContext(ApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
  }
  @PostConstruct
  public void init() throws Exception {
    Map<String, Object> beans = null;
    beans = applicationContext.getBeansWithAnnotation(RocketMQProducer.class);
  }
}
```

### BeanPostProcessor

所有Bean对象初始化前后都会回调BeanPostProcessor接口，实现BeanPostProcessor接口可以做很多定制化工作，例如：解析自定义注解。 注意：BeanPostProcessor和InitializingBean不同，InitializingBean只会在实现类Bean创建以后回调，BeanPostProcessor则是在所有Bean创建以后都会回调，所以实现BeanPostProcessor接口相当于可以遍历所有创建的Bean对象。

```java
package org.springframework.beans.factory.config;
public interface BeanPostProcessor {
  // Bean对象初始化之前执行
  Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
  // Bean对象初始化之后执行
  Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

> 特别注意：BeanPostProcessor接口的两个方法不能返回null，否则后面就走不下去了，千万注意。

执行顺序

```shell
Constructor
@PostConstruct
afterPropertiesSet
postProcessBeforeInitialization
postProcessAfterInitialization
```

下面仍然以解析自定义注解@RocketMQProduer为例说明BeanPostProcessor的用法：

```java
@Configuration
public class MyBeanPostProcessor implements BeanPostProcessor{
  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) 
    throws BeansException {
    return bean;
  }
  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) 
    throws BeansException {
    Class clazz = bean.getClass();
    // 判断Bean对象是否声明了RocketMQProducer注解
    if(!clazz.isAnnotationPresent(RocketMQProducer.class)) {
      return bean;
    }
    // 读取注解
    RocketMQProducer annotation = (RocketMQProducer)clazz.getAnnotation(RocketMQProducer.class);
    // 获取注解上的属性
    System.out.println(annotationTable.value());
    return bean;
  }
}
```

### BeanFactoryPostProcessor

BeanFactoryPostProcessor允许容器实例化Bean之前读取Bean的定义(配置元数据)，并可以修改它。典型的，`<context:property-placeholder>` 标签就是使用BeanFactoryPostProcessor的例子，读取Bean对象value中的`${db.username}` ，替换为从properties文件中读取到的真实值。`<context:property-placeholder>`  标签使用PropertyResourceConfigurer类实现。

```java
package org.springframework.beans.factory.config;
public abstract class PropertyResourceConfigurer implements BeanFactoryPostProcessor {
  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
    throws BeansException {
    try {
      Properties mergedProps = mergeProperties();
      // Convert the merged properties, if necessary.
      convertProperties(mergedProps);
      // Let the subclass process the properties.
      processProperties(beanFactory, mergedProps);
    }
    catch (IOException ex) {
      throw new BeanInitializationException("Could not load properties", ex);
    }
  }  
}
```

post()方法最重要是传入了ConfigurableListableBeanFactory参数，通过它可以读取到Bean的定义

```java
package org.springframework.beans.factory.config;
public interface ConfigurableListableBeanFactory extends ListableBeanFactory, 
		AutowireCapableBeanFactory, ConfigurableBeanFactory {
  // 读取BeanDefinition	          
  BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
}
```

### BeanDefinitionRegistryPostProcessor

这个接口给了我们在Spring容器构造完所有Bean对象后对其进行修改的可能，Mybatis的动态代理就是基于BeanDefinitionRegistryPostProcessor实现的。

下面通过一个例子说明它的用法。

我们定义@Car注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Car {
    String value() default "";
}
```

我们定义接口ICar和它的实现类Benz、Audi

```java
@Car("Benz")
public interface ICar {
  String getBrand();
}
public class Benz implements ICar {
  protected String brand = "Benz";
  @Override
  public String getBrand() {
    return brand;
  }
}
public class Audi implements ICar {
  protected String brand = "Audi";
  @Override
  public String getBrand() {
    return brand;
  }
}
```

我们希望注入ICar时，通过@Car注解的value属性来确定创建那个实体类对象

```java
@Autowired
ICar car;
```

由于`@Car("Benz")` ，所以这里car变量注入Benz类实例对象。

我们先来分析一下如何实现，然后再来看代码：

- 我们在Spring容器完成Bean创建后，扫描@Car注解，手动将正确的Bean对象加入到容器中；
- 加入到容器中的Bean对象的类型是ICar，实体类对象是Benz或者Audi；
- 这里我们用FactoryBean来返回ICar对象。

先看如何扫描注解

```java
public class MyBeanDefinitionRegistry implements BeanDefinitionRegistryPostProcessor {
  private static final String RESOURCE_PATTERN = "/**/*.class";
  private ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
  CarFactoryBean carFactoryBean = new CarFactoryBean();	
  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) 
    throws BeansException {
    // 去哪个目录扫描注解
    String basePackageName = "cn.lu.spring.ioc.post.registry";
    String basePackage = ClassUtils.convertClassNameToResourcePath(basePackageName);
    try {
      String pattern = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX + 
        basePackage + RESOURCE_PATTERN;
      // 读取资源文件（.class文件）
      Resource[] resources = resolver.getResources(pattern);      
      MetadataReaderFactory readerFactory = new CachingMetadataReaderFactory(resolver);
      for (Resource resource : resources) {
        // 遍历包下的每一个.class文件
        if (resource.isReadable()) {
          // 解析.class文件
          MetadataReader reader = readerFactory.getMetadataReader(resource);          
          for (String annotationType : reader.getAnnotationMetadata().getAnnotationTypes()) {
            // 查找@Car注解
            if (!annotationType.equalsIgnoreCase(Car.class.getName())) {
              continue;
            }
            // 构造Bean定义BeanDefinition
            ScannedGenericBeanDefinition definition = new ScannedGenericBeanDefinition(reader);
            // beanName是ICar，Bean对象的类型是Interface，实体是FactoryBean
            String shortClassName = ClassUtils.getShortName(definition.getBeanClassName());
            String beanName = Introspector.decapitalize(shortClassName);
            // 定义<bean>的class为CarFactoryBean
            definition.setBeanClass(carFactoryBean.getClass());
            // 读取@Car注解的所有属性
            Map<String, Object> properties = reader.getAnnotationMetadata()
              .getAnnotationAttributes(annotationType);
            // 根据@Car注解的value属性为CarFactoryBean添加brand属性
            definition.getPropertyValues().add("brand", properties.get("value"));
            definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
            BeanDefinitionHolder holder = new BeanDefinitionHolder(definition,beanName);
            // 将我们手动创建的Bean加入到容器中
            BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
          }
        }
      }
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}
```

几个要点：

- postProcessBeanDefinitionRegistry()是入口
- 使用PathMatchingResourcePatternResolver读取.class文件
- 使用MetadataReader解析.class文件
- 创建BeanDefinition就是手动定义一个Bean的信息
- setBeanClass()为FactoryBean把接口改为了实体类，如果不修改Spring会尝试创建ICar对象，会报错
- 最后registerBeanDefinition()把手工定义的Bean加入到容器中

FactoryBean就比较简单了

```java
public class CarFactoryBean implements FactoryBean<Car> {
  protected String brand;
  @Override
  public Car getObject() throws Exception {
    if (brand.equalsIgnoreCase("Benz")) {
      return new Benz();
    } else if (brand.equalsIgnoreCase("Audi")) {
      return new Audi();
    } else {
      return new Car("Unknown");
    }
  }
}
```

### PostProcessor对比

- InitializingBean，每个对象实例化完成后执行该对象的afterPropertiesSet()方法，每个对象执行自己的方法，互不影响，无参数传入；一般我们可以在这个时间点做初始化操作。
- BeanPostProcessor，所有对象初始化前后都会执行同一个postProcessAfterInitialization()方法，每个对象执行的都是同一个对象的方法，出入参数为Bean对象本身；一般我们可以在这个时间点对Bean的特殊属性做特殊处理，例如自定义注解。通过修改返回值，实际上可以返回另外一个Bean对象。
- BeanFactoryPostProcessor，每个对象实例化前执行，可以读取到bean的配置元数据，并进行修改；
- BeanDefinitionRegistryPostProcessor，在IoC容器完成Bean初始化之后执行，我们可以在此时加入其他Bean对象到IoC容器中。

### 参考

[Spring探秘|妙用BeanPostProcessor](https://www.jianshu.com/p/1417eefd2ab1)

