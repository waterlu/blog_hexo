---
title: Spring-IoC
date: 2018-05-22 11:34:34
categories: Spring
tags: [Spring, IoC]
toc: true
description: Spring与IOC相关的要点，如何从XML配置一步一步发展到JavaBean配置的。
comments: false
---

## IoC概述

IoC（Inversion of Control），顾名思义控制反转，什么是控制反转呢？这里的控制指对象的控制权，包括对象的创建和销毁等操作，控制反转就是对象的创建和销毁本来应该由应用程序自己控制的，现在应用程序把对象的控制权交给了Spring容器来管理，这就是控制反转。

> 个人理解：Spring容器成为Bean对象的代理或中介。

经常与Ioc一起出现的还有DI（Dependency Injection）依赖注入。依赖注入的意思是应用程序需要的对象是依赖于Spring容器注入的，不是自己创建的。看上去和IoC是一个意思，个人理解DI是具体实现IoC的一种方法，通过依赖Spring容器注入对象的方式实现了应用程序把对象的控制权转交给了Spring容器。

> 如果用租房来比喻，常规方式就是租户自己联系房东租房，IoC就是租户把找房子这件事代理给中介来完成，DI就是中介会帮租户找好房子，租户直接入住就好。（IoC还有可能是中介帮租户联系房东，租户一个一个看房，DI就是租户完全授权给中介，中介直接帮他定一个）。

### 面向接口编程

自己管理对象不是很好吗，为什么要使用IoC，把控制权交给Spring容器呢？

我理解，目的当然是解耦合。对象由Spring容器管理意味着应用程序不必依赖固定的Bean对象，就像租房只要找到满足条件的房子即可，不必非要指定一套房子，这就是解耦合。因此可以得出结论：面向接口编程是IoC的基础，如果应用程序声明注入是实体类，那么还是强耦合的；所以通常应用程序声明注入的都是接口，这才能发挥Spring IoC的作用。

例如：注入UserService接口后可以方便的切换实现类以实现不同的目标。

```java
public interface UserService {
}
@Component
public class UserServiceMemoryImpl implements UserService {
}
// @Component
public class UserServiceDatabaseImpl implements UserService {
}
```

应用程序使用时声明注入的是接口，具体实现可以通过打开和关闭@Component注解切换

```java
@Autowired
UserService userService;
```

注入Bean通常有两种方式：byType和byName，按照类型注入或者按照名字注入。按照类型注入是典型的面向接口编程思想，byType时一般class类型为接口，容器负责查找实现了接口的Bean对象。

## IoC使用

### Bean配置

既然Spring容器负责管理类对象，那么它就得知道哪些类对象需要被管理。随着Spring的发展，配置bean对象的方法也在变化，可以使用XML配置文件，也可以使用注解或者JavaConfig，下面我们就来看一下配置方式的进化过程。

#### XML配置

把需要创建的对象（Spring称之为Bean）定义在一个xml文件里面，每个bean指定ID和对应类。IoC容器读取这个xml文件，通过反射创建类对象，通过ID返回类对象。Bean里面可以通过property属性显式定义bean的依赖，也可以不定义，让Spring完成自动配置。

例如：配置/resources/spring-context.xml如下：定义了car和wheel两个bean，car依赖wheel

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
  <bean id="car" class="cn.lu.spring.ioc.Car" scope="singleton">
    <!--constructor-arg ref="wheel" /-->
    <property name="wheel" ref="wheel" />
  </bean>
  <bean id="wheel" class="cn.lu.spring.ioc.Wheel"/>
</beans>
```

Bean对象如下：

```java
public class Car {
  private Wheel wheel;
  public void setWheel(Wheel wheel) {
    this.wheel = wheel;
  }
}
public class Wheel {
}
```

> 注意：这里Car类必须实现setWheel()方法。

使用如下：

- ApplicationContext类可以理解为IoC容器，ClassPathXmlApplicationContext类是从classpath目录读取XML文件来解析Bean的IoC容器实现；
- 调用容器类的getBean()方法就可以从容器中获取Bean对象了，传入的参数可以是ID，也可以是类名；
- Bean对象之间的依赖关系可以在XML中定义bean时通过property属性设置，相当于调用set方法，也可以使用constructor-arg设置，相当于在构造函数中传入依赖对象。

```java
@RunWith(BlockJUnit4ClassRunner.class)
public class IoCTest {
  private ApplicationContext context;
  @Before
  public void before() {
    context = new ClassPathXmlApplicationContext("classpath:spring-context.xml");
  }
  @Test
  public void testContext() {
    Car car = (Car)context.getBean("car");
    // Car car = context.getBean(Car.class);
  }
}
```

以上这种用法非常好理解，ApplicationContext类读取XML文件并构造Bean对象，通过getBean()返回对象，ApplicationContext类就是IoC容器。容器需要做的就是解析XML文件，通过反射构造对象。

> 以上方法实际上没有实现依赖注入，需要自己主动获取。

#### 注解配置

如上所述，我们需要把所有用到的bean对象都定义在xml文件中，xml文件将越来越庞大。Spring 2.5以后出现了注解方式，使用注解意味着将集中的bean配置（xml文件）分散到各个bean对象中。个人觉得没有两种用法没有好坏之分。

##### 注解

@Component注解起到XML文件中`<bean/>` 一样的作用，下面两种写法是等同的。

```java
@Component("car")
public class Car {}
```

```xml
<bean id="car" class="Car"/>
```

@Component注解的value属性等同于bean的id属性，@Component的value默认值为首字母小写的类名，一般可以省略。从@Component又衍生出@Controller、@Service和@Repository注解，用在不同层，本质是一样的。

@Autowired/@Resource注解起到XML文件中<bean.property>和<bean.constructor-arg>的作用，@Autowired注解可以修饰类的成员变量，也可以修饰类的构造函数和成员方法。

@Autowired和@Resource的区别：

- @Autowired默认通过bean的类型注入；
- @Resource默认通过bean的名字注入。

引入@Autowired注解后真正实现了依赖注入，当我们需要使用一个对象时，直接将其声明为@Autowired，容器负责帮我们构造这个对象的实例。@Autowired注解默认是按照类型构造的，也就是说容器负责查找实现类并通过反射进行构造，所以通常@Autowired注解修饰的是接口。

进一步思考一下，如果一个@Autowired注解修饰的接口有多个实现类，那么容器就不知道应该构造哪个类了，怎么办？这个时候我们可以增加@Qualifier注解来指定bean的名称。下面以UserService为例，有UserServiceMemoryImpl和UserServiceDatabaseImpl两个实现类，实现选择数据库实现，代码如下：

```java
@Autowired
@Qualifier("userServiceDatabaseImpl")
UserService userService;
```

> 因为@Component默认bean名称为首字母小写的类名，所以可以使用userServiceDatabaseImpl，如果UserServiceDatabaseImpl在@Component中自定义了bean名称，那么要使用自定义的。

##### 例子

引入注解后需要增加一个新的配置，就是到哪里去查找注解，所以使用注解后的xml配置文件如下：

```xml
<beans>
  <context:component-scan base-package="cn.lu.spring.ioc" />
</beans>
```

Bean对象如下：

```java
@Component
public class Car {
  @Autowired
  private Wheel wheel;
}
@Component
public class Wheel {
}
```

> 注意：这里Car类不用实现setWheel()方法就可以注入wheel，这和反射的实现有关。

使用如下：我们通过@Autowired注解直接注入Car对象，这才是真正的依赖注入。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"spring-context.xml"})
public class IoCTest {
  private Logger logger = LoggerFactory.getLogger(this.getClass());
  @Autowired
  private Car car;
  @Test
  public void testAnnotation() {
    logger.info(car.toString());
  }
}
```

注意：这里我们没有显式创建ApplicationContext，是 @Runwith(SpringJUnit4ClassRunner.class)和@ContextConfiguration起了相同的作用。

#### Java配置

从Spring 3.0开始提供了Java Config也叫做Java配置，Spring Boot建议使用Java配置替代XML配置。

Java配置引入了@Configuration、@Bean、@ImportResource和@Value等注解。

@Configuration注解就相当于一个xml文件，@Bean注解就相当于xml文件中的一个`<bean/>`  ，@ComponentScan注解相当于xml文件中的`<context:component-scan />` 。

例如：Java配置

```java
@Configuration
public class Config {
  @Bean
  public Wheel wheel() {
    return new Wheel();
  }
  @Bean
  public Car car(Wheel wheel) {
    Car car = new Car();
    car.setWheel(wheel);
    return car;
  }
}
```

等同于如下xml配置

```xml
<beans>
  <bean id="wheel" class="cn.lu.spring.ioc.Wheel"/>
  <bean id="car" class="cn.lu.spring.ioc.Car">
    <property name="wheel" ref="wheel" />
  </bean>
</beans>
```

使用时和前面基本一样，只需要把@ContextConfiguration配置的xml文件替换为class类

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {Config.class})
public class IoCTestWithJavaConfig {
}
```

#### 选择

现在我们有三种方法定义一个Bean，选择哪一个呢？

```xml
<bean id="car" class="cn.lu.spring.ioc.Car"/>
```

```java
// @Component
public class Car {}
```

```java
@Configuration
public class JavaConfiguratoin {
  @Bean
  public Car car() {
    return new Car();
  }    
}
```

习惯上，我们使用第二种方法@Component注解（实际更多使用@Controller、@Service等注解）来配置我们的业务Bean，XML不再使用，Java配置用来完成非业务Bean的配置。

> 个人理解：@Component注解是侵入式的，XML和Java配置是不需要侵入源代码的，这是他们的优势。

### SpringBoot

下面看看在SpringBoot中如何使用IoC管理Bean对象。

```java
@SpringBootApplication
public class ServiceDemoApplication {
}
```

```java
@Controller
public class UserController {
  @Autowired
  UserService userService;
}
```

```java
@Service
public class UserServiceImpl implements UserService {
}
```

上面的例子代码展示了最常见的用法：

- Spring容器扫描到@Service注解后创建UserServiceImpl类实例对象；
- Spring容器扫描到@Controller注解后创建UserController类实例对象；
- Spring容器扫描到@Autowired注解后将UserServiceImpl类实例对象注入到UserController类实例对象的userService属性中。

上面过程很好理解，但仔细一想你会发现代码中没有出现@ComponentScan注解，Spring容器怎么知道去哪里扫描注解呢？还有，容器类ApplicationContext是如何创建的呢？

#### @SpringBootApplication

问题的答案就在@SpringBootApplication注解中。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}
```

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {
}
```

@SpringBootApplication注解组合了@Configuration、@ComponentScan和@EnableAutoConfiguration三个最重要的注解，所以拥有了它们的功能。

我们知道一个@Configuration类文件对应一个XML配置文件，所以当Spring扫描到@Configuration注解后就会创建ApplicationContext容器类，并扫描@Component注解创建Bean对象放入容器类中。

@ComponentScan注解默认扫描当前包及其子目录下的源代码，所以Application类通常都是放在包的最外层，这样才可以保证web/service等目录下的注解可以被扫描到。

例如：新建应用在com.test.demo包下，引用的common项目在com.test.common包下，那么common项目中的@Component等注解是不生效的，因为Spring没有扫描它们。如果希望common包下的注解也生效，需要显式配置scanBasePackages，样例代码如下：

```java
@SpringBootApplication(scanBasePackages = {"com.test.demo", "com.test.common"})
public class DemoApplication {
}
```

> @EnableAutoConfiguration是SpringBoot特性，和IoC无关，这里不展开。

### 高级使用

前面介绍了IoC的最基本用法，帮助我们的应用程序创建类实例对象，下面看看其他用法。

#### 初始化

实际开发中，有时需要在Bean对象创建后和销毁前做一些操作，可以使用下面两个注解：

- @PostConstruct，在构造函数完成之后执行；
- @PreDestroy，在析构函数执行之前执行；

除了使用@PostConstruct注解，还可以实现InitializingBean接口和afterPropertiesSet()方法。

```java
@Controller
public class UserController implements InitializingBean {
  public UserController() {
    System.out.println("UserController()");
  }  
  @PostConstruct
  public void postConstruct() {
    System.out.println("UserController @PostConstruct");
  }
  @PreDestroy
  public void preDestroy() {
    System.out.println("UserController @PreDestroy");
  }
  @Override
  public void afterPropertiesSet() throws Exception {
    System.out.println("UserController afterPropertiesSet");
  }
}
```

上面代码的输出日志顺序为：

```shell
UserController()
UserController @PostConstruct
UserController afterPropertiesSet
UserController @PreDestroy
```

#### @Scope

常用的Scope有两种：Singleton和Prototyp，默认是Singleton：

- Singleton，一个容器只创建一个Bean实例，每次getBean()都返回相同的实例；
- Prototyp，每次getBean()都新创建一个Bean实例；
- Session，每个HTTP Session创建一个Bean实例。

> 注意：Singleton强调一个容器只有一个Bean实例，不同容器可以有不同Bean实例。

#### @Profile

@Profile注解为我们提供了在不同环境下创建不同Bean实例的能力。

一般用在配置类中，例如：生产环境MySQL和Redis需要使用集群配置，开发和测试环境使用单点配置就好了，这两种情况下的配置可能是不同的，这个时候@Profile就派上用场了。

```java
@Configuration
public class DatabaseConfig {
  @Bean
  @Profile("dev")
  public DataSource getDevDataSource() {
    return new DevDataSource();
  }
  @Bean
  @Profile("prod")
  public DataSource getProdDataSource() {
    return new ProdDataSource();
  }
}
```

```java
@Autowired
DataSource dataSource;
```

DataSource是接口，DevDataSource和ProdDataSource是具体实现，根据运行环境的不同返回不同的实现。运行环境通过配置application.properties切换。

```properties
spring.profiles.active=dev
```

#### @Value

使用@Value注解可以注入属性值，最常见的是从application.properties中读取配置。

```java
@Component
public class Book {
    @Value("${book.name}")
    private String bookName;
}
```

`@Value("${book.name}")` 表示读取配置文件中`book.name` 的值，application.properties配置如下：

```properties
book.name=Spring
```

不要忘了@Component注解，只有Spring容器管理的Bean才可以使用@Value注解。

使用@Value注解还可以读取其他Bean的属性 ，例如：以下代码从company这个Bean中读取name字段并赋值到Book的bookPub字段，和前面的区别是`$`换成了`#` 。

```java
@Component
public class Book {
    @Value("#{company.name}")
    private String bookPub;
}
```

#### @ImportResource

使用@ImportResource注解可以直接导入xml配置文件，这是从xml配置到java配置过渡的最简单方案。

```java
@Configuration
@ImportResource(locations={"classpath:applicationContext.xml"})
public class XmlConfiguration {
}
```

#### ApplicationContextAware

有时候我们需要读取容器中的其他类实例对象，例如：自定义注解后，需要扫描注解来实现自定义逻辑。以下以RocketMQ自定义的@RocketMQProduer注解为例：

- 实现ApplicationContextAware接口的setApplicationContext()方法获得容器；
- 在构造函数完成后调用ApplicationContext的getBeansWithAnnotation()方法获得所有声明了@RocketMQProducer注解的类实例对象。

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

#### BeanPostProcessor



#### BeanFactoryPostProcessor



## IoC原理





### Tomcat应用的IoC容器

早期Spring经常和Tomcat一起使用，使用的是XML配置，但我们并没有像前面例子代码那样显式调用`new ClassPathXmlApplicationContext()`  ，那么Tomcat是如何创建Spring的IoC容器的呢？

答案就在webapp/WEB-INF/web.xml中，关键点就是ContextLoaderListener

```xml
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

Tomcat中使用的是ApplicationContext的另外一个子类WebApplicationContext，ContextLoaderListener是Tomcat的一个ServletContext，当它初始化的时候创建WebApplicationContext容器类。

```java
package org.springframework.web.context;
import javax.servlet.ServletContextListener;
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
  public void contextInitialized(ServletContextEvent event) {
    this.initWebApplicationContext(event.getServletContext());
  }  
  public void contextDestroyed(ServletContextEvent event) {
    this.closeWebApplicationContext(event.getServletContext());
    ContextCleanupListener.cleanupAttributes(event.getServletContext());
  }
}
```

当Servelt容器启动时触发contextInitialized()方法，它执行基类ContextLoader的方法来构造容器

```java
package org.springframework.web.context;
public class ContextLoader {
  private WebApplicationContext context;
  public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    if(this.context == null) {
      this.context = this.createWebApplicationContext(servletContext);
    }
    return this.context;
  }
}
```

这里可以看到最终调用createWebApplicationContext()方法返回WebApplicationContext类实例对象，这就是Tomcat中的IoC容器。后面的操作大家都了解了，WebApplicationContext容器负责管理Bean对象。

### SpringBoot应用的IoC容器

我们再来看看目前流行的SpringBoot是如何创建Spring的IoC容器的。

SpringBoot其实更好理解一点，我们从程序入口开始看

```java
@SpringBootApplication
public class SpringbootApplication {
  public static void main(String[] args) {
    SpringApplication.run(SpringbootApplication.class, args);
  }
}
```

入口是SpringApplication的run()方法

```java
package org.springframework.boot;
public class SpringApplication {
  public ConfigurableApplicationContext run(String... args) {
    ConfigurableApplicationContext context = null;
    context = this.createApplicationContext();
    this.refreshContext(context);
    return context;
  }
}
```

这里只截取了和IoC相关代码，可以看到run()方法中创建了context容器，并调用refreshContext()方法加载Bean对象。

```java
package org.springframework.boot;
public class SpringApplication {   
  private boolean webEnvironment;
  protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if(contextClass == null) {
      if (this.webEnvironment) {
        contextClass = Class.forName("AnnotationConfigEmbeddedWebApplicationContext"); 
      } else {
        contextClass = Class.forName("AnnotationConfigApplicationContext");
      }
    }
    return (ConfigurableApplicationContext)BeanUtils.instantiate(contextClass);
  }
}
```

> 类名太长，这里省略了包名。

由于我们大部分是web应用，所以使用的是AnnotationConfigEmbeddedWebApplicationContext，从名字上可以看出，这个ApplicationContext（容器）是基于注解创建的，支持嵌入式web应用。

### Spring IoC源码分析

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

