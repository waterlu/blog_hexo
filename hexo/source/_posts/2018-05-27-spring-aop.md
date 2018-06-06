---
title: Spring-AOP
date: 2018-05-27 09:34:10
categories: Spring
tags: [Spring, AOP]
toc: true
description: 面向切面编程是Spring除了IoC之外的又一个重要特性，本文介绍在Spring中配置和使用切面编程的方法，包括XML文件配置和注解配置，以及AspectJ的一些基本原理。
comments: true
---

## Spring AOP

### AOP

什么是AOP，AOP=Aspect Oriented Program，翻译过来就是面向切面编程。AOP与OOP是同等重要的概念，OOP是面向对象编程，面向对象的特点是封装、继承和多态。OOP是一种思想，通过把现实世界中的业务抽象描述成对象来解决问题；每个对象对应不同的业务功能，对象设计以高内聚、低耦合为目标。AOP也是一种思想，这里Aspect 的意思是横切面，或者说是各个对象的横切面，所以AOP是跨对象的，AOP解决的是各个对象的共性问题。OOP设计与具体业务强相关，AOP则是提供非业务功能的统一处理。

> 打个不恰当的比喻，AOP就像是做肉夹馍，横着把馍切开，把肉和菜放进去。如果我们有很多馍，一个一个分别切开放肉是传统做法；AOP做法是找一把很长的刀，把馍排行队，一次横向把所有馍都切开，把肉一起放进去。

简单说，AOP目的是在OOP的基础上，把各个对象中与具体业务无关的共性功能提取出来，做统一处理，这样就避免了各个对象重复建设，也方便统一管理。例如：日志输出、异常处理、事务控制、权限控制、缓存处理等功能很多对象都需要使用，与具体业务无关，如果每个对象都自己实现这些功能显然是重复且低效的，把这些共性功能提取到一个切面中进行统一管理，这就是面向切面编程。所以，面向切面编程一般都是统一xxx处理。看到这，你也许会认为这不就是拦截器吗，对，我认为拦截器就是AOP编程思想的一种体现，但AOP并不限于拦截器。

### Spring AOP

那什么是Spring AOP呢？我认为，Spring AOP就是Spring框架提供的面向切面编程的功能。具体又可以分为两块：一个是Spring框架本身提供的很多功能就是基于AOP实现的；还有一个就是我们自己也可以使用Spring完成面向切面编程的功能。

AOP这个名词听上去可能有点陌生，但实际上我们在代码中早已经使用过了，`@Transational` 和`@Cacheable` 这两个常用的注解就是基于AOP实现的。

### 概念

在学习使用Spring AOP之前，我们先来了解一些概念，这些概念是枯燥乏味的，我尽量用通俗的语言来解释。

Aspect - 切面，可以理解为一个横切多个对象的**功能**，例如：访问日志输出，访问权限控制等；

PointCut - 切入点，可以理解为对象的哪个**方法**需要被切开放入切面功能，例如：所有以set开头的方法；

Advice - 通知，分为前置、后置、异常和环绕四种；

JoinPoint - 连接点，主动切入的切面对象和被切入对象的连接处？

Weaving- 织入，完成切入的**过程**被称为织入。

### 使用

上面的概念还是太抽象，我们通过具体的例子来学习AOP，看看如何自定义切面。

#### XML配置

看看xml中如何配置一个切面。

```xml
<beans>
  <aop:config>
    <aop:aspect id="aspectLog" ref="logAspect">
      <aop:pointcut id="public" expression="execution(* cn.lu.spring.aop.Human.*(..))" />
      <aop:before method="before" pointcut-ref="public" />
      <aop:after-returning method="afterReturn" pointcut-ref="public" />
      <aop:after method="after" pointcut-ref="public" />
      <aop:after-throwing method="afterThrow" pointcut-ref="public" />     
      <aop:around method="around" pointcut-ref="public" />
    </aop:aspect>
  </aop:config>
</beans>
```

- 首先，切面配置必须包含在`<aop:config>` 中； 
- 使用`<aop:aspect>` 来声明一个切面功能，一个`<aop:config>` 中可以声明多个`<aop:aspect>`  ；
- 一个切面`<aop:aspect>` 有且仅有一个切入点`<aop:pointcut>` ，切入点通过表达式限定需要切入的方法范围，本例为切入到Human类的所有方法中；切入点就是声明切入到哪个目标方法执行；
- 最后还需要定义Advice通知，就是确定切面方法执行和目标方法执行的关系，有五种：
  - `<aop:before>` 前置通知，切面方法在切入目标方法执行之前执行；
  - `<aop:after-returning>` 返回前通知，切面方法在切入目标方法执行完成返回接过前执行；
  - `<aop:after>` 后置通知，切面方法在切入目标方法执行之后执行；
  - `<aop:after-throwing>` 异常通知，切面方法在切入目标方法抛出异常以后执行；
  - `<aop:aroud>` 环绕通知，给了切面方法更大的自由，可以自定义；

配置完成后，我们通过JUnitTest运行

```java
public class AOPWithXML {
  private ApplicationContext context;
  @Before
  public void before() {
    context = new ClassPathXmlApplicationContext("classpath:spring-aop.xml");
  }
  @Test
  public void testAop() {
    Human human = context.getBean(Human.class);
    human.go();
  }
}
```

虽然我们从容器中取出的是Human类的实例对象，调用的也是Human类方法，但输出了切面类的日志信息，这说明切入成功

```shell
LogAspect before
LogAspect around 1
Human go()
LogAspect around 2
LogAspect after
LogAspect afterReturn
```

Human类很简单，go方法什么也不做，只输出一行日志，我们看一下LogAspect类，就是`<aop:aspect>` 对应的切面类

```java
public class LogAspect {
    public void before(){
        System.out.println("LogAspect before");
    }
    public void afterReturn() {
        System.out.println("LogAspect afterReturn");
    }
    public void after() {
        System.out.println("LogAspect after");
    }
    public void afterThrow() {
        System.out.println("LogAspect afterThrow");
    }
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("LogAspect around 1");
        Object value = joinPoint.proceed();
        System.out.println("LogAspect around 2");
        return value;
    }
}
```

综上，我们可以根据业务需求使用`<aop:aspect>`配置切面类，具体通过`<aop:pointcut>` 指定切入点就是被切入的方法，然后通过`<aop:before>`和`<aop:after>`定义切面方法并指定执行的位置，最后在切面类中编写代码即可。例子中的切面类LogAspect实现了日志输出功能，延伸一点，使用`<aop:aroud>` 可以统计方法的执行时间。

#### 注解配置

同样的功能，我们再来看看用注解如何完成。

```java
@Component
@Aspect
public class LogAspect {
    @Pointcut("execution(* cn.lu.spring.aop.Human.*(..))")
    public void pointCut() {
    }
    @Before("pointCut()")
    public void before(){
        System.out.println("LogAspect before");
    }
    @AfterReturning("pointCut()")
    public void afterReturn() {
        System.out.println("LogAspect afterReturn");
    }
    @After("pointCut()")
    public void after() {
        System.out.println("LogAspect after");
    }
    @AfterThrowing("pointCut()")
    public void afterThrow() {
        System.out.println("LogAspect afterThrow");
    }
    @Around("pointCut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("LogAspect around 1");
        Object value = joinPoint.proceed();
        System.out.println("LogAspect around 2");
        return value;
    }  
}
```

以上代码相当于对xml文件做了一次翻译工作，很好理解，只是@PointCut这种写法有点奇怪。当然，如果你不嫌麻烦也可以不定义@PointCut，直接这样写：

```java
@Component
@Aspect
public class LogAspect {
    @Before("execution(* cn.lu.spring.aop.Human.*(..))")
    public void before(){
        System.out.println("LogAspect before");
    }
}
```

有一点需要注意，需要先开启AspectJ，下面用@EnableAspectJAutoProxy注解开启AspectJ：

```java
@Configuration
@EnableAspectJAutoProxy
public class AopConfig {
  @Bean
  public Human human() {
    return new Human();
  }
  @Bean
  public LogAspect logAspect() {
    return new LogAspect();
  }
}
```

### 原理

以上两种方法，无论xml配置还是@Aspect注解配置，都是用静态代理实现AOP。Spring使用AspectJ实现静态代理，所以上面的例子需要引入AsprectJ包。

```xml
<dependency>
  <groupId>org.aspectj</groupId>
  <artifactId>aspectjweaver</artifactId>
  <version>1.8.9</version>
</dependency>
```

否则使用XML配置会抛出如下异常：

```shell
NoClassDefFoundError: org/aspectj/weaver/reflect/ReflectionWorld
```

使用@Aspect注解就更不用说了，@Aspect注解在`org.aspectj.lang.annotation.*;` 包中定义。也就是说@Aspect不是Spring的注解，是AspectJ的注解。

Spring实现AOP有三种方式：

- AspectJ - 静态代理，编译期实现AOP；
- JDK动态代理 - 动态代理，运行期实现，基于接口实现的AOP；
- CGLib动态代理 - 动态代理，运行期实现，无接口时实现AOP。

由于Spring框架倡导基于接口编程，所以实际上JDK动态代理使用的最多。

#### AspectJ

AspectJ可以理解为Java的Aspect实现，也就是Java提供的一个实现AOP的包，那和Spring AOP有什么关系呢？就像OOP一样，AOP是一种思想或者编程方法，和具体开发语言和开发框架无关。C++和Java都可以实现面向对象编程，同样道理，AOP的实现方式也是多种多样的。Java有自己的AOP实现，JBoss有自己的AOP实现，Spring也有自己的AOP实现。Spring的AOP实现在静态的情况下，直接使用了AspectJ实现，就是这么简单。

>  因为Spring只是做了配置，真正实现在AspectJ中，所以我们必须在pom.xml中引入AspectJ的包。

AspectJ的实现方式是在编译期间自动生成Java字节码，也可以理解为一种Java编译器。编译时，AspectJ根据配置修改原始的.class文件，添加增强代码，形成新的.class文集。所谓增强代码就称为Advice，添加增强代码的位置就是JoinPoint，合并的过程就称为织入。

AspectJ的好处是非侵入，不需要像JDK动态代理一样必须要定义接口。

**AspectJ揭秘**

下面我们来看一看AspectJ都做了什么，打开Human.java文件。

```java
@Component
public class Human {
  public void go(){
    System.out.println("Human go()");
  }
}
```

在打开target/classes目录下的Human.class文件，IDEA自动帮我们反编译成java源文件

```java
@Component
public class Human {
    public Human() {
    }
    public void go() {
        try {
            try {
                LogAspect.aspectOf().before();
                System.out.println("Human go()");
                LogAspect.aspectOf().afterReturn();
            } catch (Throwable var3) {
                LogAspect.aspectOf().after();
                throw var3;
            }
            LogAspect.aspectOf().after();
        } catch (Throwable var4) {
            LogAspect.aspectOf().afterThrow();
            throw var4;
        }
    }
}
```

同样再看看target/classes目录下的LogAspect.class文件

```java
@Component
@Aspect
public class LogAspect {
  public LogAspect() {
  }
  @Before("pointCut()")
  public void before() {
    System.out.println("LogAspect before");
  }
  public static LogAspect aspectOf() {
    if(ajc$perSingletonInstance == null) {
      throw new NoAspectBoundException("cn.lu.spring.aop.LogAspect", ajc$initFailureCause);
    } else {
      return ajc$perSingletonInstance;
    }
  }
  public static boolean hasAspect() {
    return ajc$perSingletonInstance != null;
  }
  static {
    try {
      ajc$postClinit();
    } catch (Throwable var1) {
      ajc$initFailureCause = var1;
    }
  }
}
```

>  这里不分析AspectJ实现的细节，大家只要知道AspectJ在编译时修改.class文件添加功能增强代码即可。

当然，现在如果你打开Human.class文件就会发现根本没有AspectJ织入的增强代码。这是因为我提前做了AspectJ相关的配置，为了看到这些增强代码，你需要修改IDEA和项目的配置。

1. 首先，请确认IDEA安装了AspectJ相关的两个插件：AspectJ Support和Spring AOP/@AspectJ。
2. 然后，我们需要修改Java编译器。在Setting->Build->Comiler中找到Java Compiler，修改"Use compiler"的选项，将"Javac"修改为Ajc。从这里也可以印证AspectJ是在编译器上做的手脚。

在"Path to Ajc compiler"项目输入aspectjtools.jar的全路径名，可以去[这里](http://www.eclipse.org/downloads/download.php?file=/tools/aspectj/aspectj-1.8.9.jar) 下载。如果下载速度非常慢（我就是这样）也可以去[maven库](https://mvnrepository.com) 中下载。我这里使用的是1.8.9版本，下载到了maven库中，所以我的配置如下：

```shell
C:\Users\lu\.m2\repository\org\aspectj\aspectjtools\1.8.9\aspectjtools-1.8.9.jar
```

3. 接下来，修改项目配置。先在pom.xml中添加aspectjrt包依赖（我这里选用的都是1.8.9版本）。

```xml
<dependency>
  <groupId>org.aspectj</groupId>
  <artifactId>aspectjrt</artifactId>
  <version>1.8.9</version>
</dependency>
```

然后进入Proejct Structure->Modules->Dependencies，勾选org.aspectj:aspectjrt:1.8.9。

4. 最后，在Application的启动参数VM Options里面添加`-javaagent: ` ，指向aspectjweaver.jar，我的配置如下：

```shell
-javaagent:C:/Users/lu/.m2/repository/org/aspectj/aspectjweaver/1.8.9/aspectjweaver-1.8.9.jar
```

5. 好了，以上配置完成以后，重新编译运行就能在.class文件中看到AspectJ织入的增强代码了。

以上都是为了证明AspectJ是在编译期实现的静态AOP。

#### JDK动态代理

我专门有一篇文章介绍JDK动态代理，这里不再赘述，只关于Spring如何使用JDK动态代理。



#### CGLib动态代理



### 



https://www.colabug.com/2102191.html

http://sexycoding.iteye.com/blog/1062372









