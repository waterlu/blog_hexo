---
title: Spring-IoC
date: 2018-05-22 11:34:34
categories: Spring
tags: [Spring, IoC]
toc: true
description: IoC是Spring最重要的组织部分，面向接口编程是IoC的基础，本文介绍IoC的基本概念以及如何使用XML文件、注解和Java代码配置Bean对象。
comments: true
---

## IoC概述

IoC（Inversion of Control），顾名思义控制反转，什么是控制反转呢？这里的控制指对象的控制权，包括对象的创建和销毁等操作，控制反转就是对象的创建和销毁本来应该由应用程序自己控制的，现在应用程序把对象的控制权交给了Spring容器来管理，这就是控制反转。

> 个人理解：Spring容器成为Bean对象的代理或中介。

经常与Ioc一起出现的还有DI（Dependency Injection）依赖注入。依赖注入的意思是应用程序需要的对象是依赖于Spring的IoC容器注入的，不是自己创建的。看上去和IoC是一个意思，个人理解DI是具体实现IoC的一种方法，通过依赖IoC容器注入对象的方式实现了应用程序把对象的控制权转交给了IoC容器。除了DI依赖注入实现IoC以外，还可以DL（Dependency  Lookup）主动从IoC容器中读取Bean对象。

> 如果用租房来比喻，常规方式就是租户自己找房子，IoC就是租户把找房子这件事交给给中介来完成，DI就是中介找好房子把信息Push给租户，DL就是租户主动从中介Pull房子信息。

### 面向接口编程

自己管理对象不是很好吗，为什么要使用IoC，把控制权交给IoC容器呢？

我理解，目的当然是解耦合。对象由IoC容器管理意味着应用程序不必依赖固定的Bean对象，就像租房只要找到满足条件的房子即可，不必非要指定某一套房子，这就是解耦合。因此可以得出结论：面向接口编程是IoC的基础，如果应用程序声明注入是实体类，那么还是强耦合的；所以通常应用程序声明注入的Bean对象都是接口，这样才能更好发挥IoC的作用。

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

应用程序使用时声明注入的是接口，具体实现类可以通过打开和关闭@Component注解切换，引入方无感知。

```java
@Autowired
UserService userService;
```

注入Bean通常有两种方式：byType和byName，按照类型注入或者按照名字注入。按照类型注入是典型的面向接口编程思想，byType时一般class类型为接口，容器负责查找实现了接口的Bean对象。

## IoC使用

### 配置Bean对象

既然我们想把管理对象的工作交给IoC容器，那么它就得知道哪些对象需要被管理。随着Spring的发展，告诉IoC容器需要管理哪些Bean对象的方式也在变化，可以使用XML配置文件，也可以使用注解或者JavaConfig。

#### XML配置

把需要创建的Bean对象定义在一个xml文件里面，每个bean指定ID和对应类。IoC容器读取这个xml文件，通过反射创建Bean对象，通过ID返回Bean对象。同样，Bean对象之间的依赖关系也通过xml文件来配置。

##### bean标签

具体通过bean标签来配置一个Bean对象，通过class属性反射创建，通过id属性引用Bean对象；

- <property ref="" /> - 对应Bean对象的属性，形成依赖关系，调用set()方法设置；
- <property value="" /> - 也可以通过value属性直接赋值；
- <constructor-arg ref="" /> - 对应Bean对象构造函数的参数，形成依赖关系，通过构造函数设置；
- ref - ref就是Reference，引用，值为Bean的id；

例如：配置/resources/spring-context.xml如下：定义了car和wheel两个bean，car依赖wheel

```xml
<beans>
  <bean id="car" class="cn.lu.spring.ioc.Car">
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
- 调用容器类的getBean()方法就可以从容器中获取Bean对象了，传入的参数可以是ID，也可以是类名。

```java
@RunWith(BlockJUnit4ClassRunner.class)
public class IoCTest {
  private ApplicationContext context;
  @Before
  public void before() {
    context = new ClassPathXmlApplicationContext("classpath:spring-bean.xml");
  }
  @Test
  public void testContext() {
    Car car = context.getBean(Car.class);
  }
}
```

以上这种用法非常好理解，ApplicationContext类读取XML文件并构造Bean对象，通过getBean()返回对象，ApplicationContext类就是IoC容器。容器需要做的就是解析XML文件，通过反射构造对象。

> 以上方法实际上没有实现DI依赖注入，需要自己主动获取，这种实现方式也被称作DL依赖查找。

##### scope

Bean对象具有scope属性，常用的有两种：Singleton和Prototyp，默认是Singleton：

- Singleton，一个容器只创建一个Bean实例，每次getBean()都返回相同的实例；
- Prototyp，每次getBean()都新创建一个Bean实例；
- Session，每个HTTP Session创建一个Bean实例。

> 注意：Singleton强调一个容器只有一个Bean实例，不同容器可以有不同Bean实例。

##### import标签

如上所述，xml配置是非常好理解的。但是，我们需要把所有用到的bean对象都定义在xml文件中，而且还要定义bean对象之间复杂的依赖关系，xml文件将越来越庞大。实战中，会根据类型不同创建多个xml文件，并通过`<import>`标签引入。

例如：

```xml

```

##### 读取配置文件

类似配置数据源时，用户名和密码等信息我们一般不会直接写在xml文件里面，而是放在单独的properties文件中，使用`<content:property-placeholder />` 标签可以读取配置文件。

例如：

```xml
<beans> 
  <context:property-placeholder location="classpath:db.properties" />
  <bean id="database" class="cn.lu.spring.ioc.placeholder.Database">
    <property name="driver" value="${db.driver}" />
    <property name="url" value="${db.url}" />
    <property name="username" value="${db.username}" />
    <property name="password" value="${db.password}" />
  </bean>
</beans>   
```

> 注意：`${db.username}` 如果换成 `${username}` 将读取到本地用户。也就是说是由内置变量的，所以通常在properties文件中属性都要加前缀，避免冲突。

##### 自动装配

前面我们使用`<property>` 属性手工配置了Car对象和Wheel对象的依赖关系，此外，还可以使用`autowire` 属性自动装配。自动装配意味着IoC自动帮我找到Wheel对象，并赋值到Car对象的wheel属性。

```xml
<beans>
  <bean id="car" class="cn.lu.spring.ioc.Car" autowire="byType">
  <bean id="wheel" class="cn.lu.spring.ioc.Wheel"/>
</beans>
```

`autowire` 属性有三个值：

- no - 默认值，默认不自动装配；
- byName - 通过Bean的id自动装配，这里我们把`id="wheel"` 修改了`id="wheel2"` 装配将会失败；
- byType - 通过Bean的class自动装配，最常用。

byType自动装配时，如果没有根据类型找到对象，或者找到多个对象，都会抛出异常。



#### 注解配置

Spring 2.5以后出现了注解方式，使用注解意味着将集中的bean配置（xml文件）分散到各个bean对象中。个人觉得没有两种用法没有好坏之分。

##### @Component注解

@Component注解起到XML文件中`<bean/>` 标签一样的作用，下面两种写法是等同的。

```java
@Component("car")
public class Car {}
```

```xml
<bean id="car" class="Car"/>
```

@Component注解的value属性等同于bean的id属性，@Component的value默认值为首字母小写的类名，一般可以省略。从@Component又衍生出@Controller、@Service和@Repository注解，本质是一样的。

##### @Autowired注解

@Autowired/@Resource注解起到XML文件中`autowird` 属性的作用，@Autowired注解可以修饰类的成员变量，也可以修饰类的构造函数和成员方法。

@Autowired和@Resource的区别：

- @Autowired默认通过bean的类型注入，等价于`autowire="byType"` ；
- @Resource默认通过bean的名字注入，等价于`autowire="byName"`。

##### 依赖注入

引入自动装配和@Autowired注解后真正实现了依赖注入，当我们需要使用一个对象时，直接将其声明为@Autowired，容器负责帮我们构造这个对象的实例。@Autowired注解默认是按照类型构造的，也就是说容器负责查找实现类并通过反射进行构造，所以通常@Autowired注解修饰的是接口。

进一步思考一下，如果一个@Autowired注解修饰的接口有多个实现类，那么容器就不知道应该构造哪个类了，怎么办？这个时候我们可以增加@Qualifier注解来指定bean的名称。下面以UserService为例，有UserServiceMemoryImpl和UserServiceDatabaseImpl两个实现类，实现选择数据库实现，代码如下：

```java
@Autowired
@Qualifier("userServiceDatabaseImpl")
UserService userService;
```

> 因为@Component默认bean名称为首字母小写的类名，所以可以使用userServiceDatabaseImpl，如果UserServiceDatabaseImpl在@Component中自定义了bean名称，那么要使用自定义的。

##### component-scan

虽然加入@Component注解以后我们不需要在xml文件中定义<bean/>了，但是我们还是需要告诉IoC容器去哪里扫描@Component注解，通过` <context:component-scan>` 标签实现。

```xml
<beans>
  <context:component-scan base-package="cn.lu.spring.ioc.bean" />
</beans>
```

实际上在AutowiredAnnotationBeanPostProcessor中处理自动装配，但是要求开发者在xml文件中自己定义这个类的bean对象显然太low了，所以Spring为我们提供了`<context:annotation-config>` 标签来加载类似功能的类，由于` <context:component-scan>`包含了`<context:annotation-config>` 的功能，所以可以省略。

##### 例子

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
@ContextConfiguration(locations = {"spring-bean.xml"})
public class IoCTest {
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

总结，到目前为止我们的用法是：

- 在xml文件中定义component-scan，指向bean对象的包名；
- 在类中使用@Component和@Autowired注解。

从Spring 3.0开始提供了Java Config也叫做Java配置，Spring Boot建议使用Java配置替代XML配置。

Java配置引入了@Configuration、@Bean、@ImportResource和@Value等注解。

##### @Configuration注解

@Configuration注解就相当于一个xml文件，@Bean注解就相当于xml文件中的一个`<bean/>`  ，

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

##### @ComponentScan注解

@ComponentScan注解相当于xml文件中的`<context:component-scan />` 。

所以我们可以把xml文件替换为Java类

```java
@Configuration
@ComponentScan("cn.lu.spring.ioc.bean")
public class JavaConfig {
}
```

使用时和前面基本一样，只需要把@ContextConfiguration配置的xml文件替换为class类

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {JavaConfig.class})
public class IoCTestWithJavaConfig {
}
```

#### 选择

现在我们有三种方法定义一个Bean，选择哪一个呢？

XML配置

```xml
<bean id="car" class="cn.lu.spring.ioc.Car"/>
```

注解配置

```java
@Component
public class Car {}
```

Java配置

```java
@Configuration
public class JavaConfiguratoin {
  @Bean
  public Car car() {
    return new Car();
  }    
}
```

习惯上，我们使用第二种方法@Component注解（实际更多使用@Controller、@Service等注解）来配置我们的业务Bean，XML不再使用，Java配置用来完成非业务Bean（例如：数据源）的配置。

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

#### @ConfigurationProperties

@PropertySource注解起到和`<context:property-placeholder>` 标签一样的作用，读取proerties配置文件，并映射到Bean对象中。

```java
@Component
@PropertySource("classpath:db.properties")
public class DatabaseInfo {
  @Value("${db.driver}")
  private String driver;
  @Value("${db.url}")  
  private String url;
  @Value("${db.usernae}")
  private String userame;
  @Value("${db.password}")
  private String password;
}
```

@PropertySource注解和@Value注解配合使用，@Value可以设置默认值，例如：password默认值为`******`

```java
@Value("${db.password:******}")
private String password;
```

Spring Boot 提供了@ConfigurationProperties注解，读取配置文件更方便，可以不加@Value注解。prefix属性定义前缀名，连接符转换为驼峰，例如：`db.driver-class-name` 对应driverClassName。

```java
@ConfigurationProperties(prefix = "db")
public class DataSourceProperties {
  private String driver;
  private String url;
  private String username;
  private String password;
}
```

为了生效还必须添加@EnableConfigurationProperties注解。

```java
@Configuration
@EnableConfigurationProperties(DataSourceProperties.class)
public class Config {
}
```

### 扩展使用

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

DataSource是接口，DevDataSource和ProdDataSource是具体实现，根据运行环境的不同返回不同的实现。运行环境通过配置application.properties切换。

```properties
spring.profiles.active=dev
```

#### @ImportResource

使用@ImportResource注解可以直接导入xml配置文件，这是从xml配置到java配置过渡的最简单方案。

```java
@Configuration
@ImportResource(locations={"classpath:applicationContext.xml"})
public class XmlConfiguration {
}
```

#### @Import

和@ImportResource注解类似，使用@Import注解可以导入其他配置类

```java
@Configuration
@ImportResource({DatabaseConfig.class, RedisConfig.class})
public class AppConfiguration {
}
```

## IoC容器

### JUnit

我们的测试用例中使用ClassPathXmlApplicationContext做IoC容器，需要手工创建，并传入xml文件名。

### Tomcat

Spring和Tomcat集成后，使用的是WebApplicationContext，通过ContextLoaderListener创建。

ContextLoaderListener配置在webapp/WEB-INF/web.xml中：

```xml
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

ContextLoaderListener是Tomcat的一个ServletContext，当它初始化的时候创建WebApplicationContext容器类。

```java
package org.springframework.web.context;
import javax.servlet.ServletContextListener;
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
  public void contextInitialized(ServletContextEvent event) {
    // Context初始化时创建IoC容器
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

### SpringBoot

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





[Spring探秘|妙用BeanPostProcessor](https://www.jianshu.com/p/1417eefd2ab1)

[bean作用域：理解Bean生命周期](https://blog.csdn.net/soonfly/article/details/69480058)

