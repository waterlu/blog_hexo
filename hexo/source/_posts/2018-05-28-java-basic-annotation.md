---
title: Java基础-注解
date: 2018-05-28 14:18:08
categories: JAVA
tags: [JAVA, 注解]
toc: true
description: 使用@interface定义注解，最常用的元注解是@Target和@Retention。Class、Method和Field类都提供了getAnnotation()方法返回注解信息。在Spring中可以调用容器类的getBeansWithAnnotation()方法得到声明了注解的Bean对象，或者实现BeanPostProcessor接口在Bean对象创建完成时进行解析和处理。
comments: false
---

## Java注解

注解自JDK1.5开始被引入，广泛用于应用程序代码中，尤其是Spring中大量使用注解，也促进了注解的流行。

什么是注解？形式上看有点像注释，但注解在程序运行期也可以起到作用，更多时起到标识的作用。从代码角度来看，注解是和类、接口并列的概念，但它不能独立存在，需要依附于类、方法或者属性。

**常见注解**

| 注解                               | 提供方        | 作用                     |
| -------------------------------- | ---------- | ---------------------- |
| @Override                        | JDK        | 子类的方法是覆盖父类的方法          |
| @Deprecated                      | JDK        | 过时的方法，编译器在编译时会给出警告     |
| @SuppressWarnings                | JDK        | 去除编译器的警告信息             |
| @Autowired                       | Spring     | 自动注入Bean               |
| @Component                       | Spring     | 声明Bean交给Spring的Ioc容器管理 |
| @Service/@Controller/@Repository | Spring     | @Component的细分          |
| @SpringBootApplication           | SpringBoot | 多个注解的组合                |

### 注解

就像`class` 定义类，`interface` 定义接口一样， `@interface` 用来定义注解。

我们先来看一下@Override的定义，其中@Target和@Retention是用来定义注解的注解，一般成为元注解。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

#### 元注解

四个常用的元注解是：@Target、@Retention、@Inherited和@Documented

**@Target**

注解的作用域，就是注解可以加在哪里，可以多选，形如`{ElementType.METHOD, ElementType.FIELD}` 。

- ElementType.TYPE - 注解加到类或者接口上
- ElementType.FIELD - 注解加到类的成员变量上
- ElementType.METHOD - 注解加到类的成员方法上
- ElementType.CONSTRUCTOR - 注解加到类的构造函数上
- ElementType.PARAMETER - 注解加到方法的参数上


**@Retention**

注解的生命周期，就是在什么时候起作用，或者说注解会被带到哪个阶段。

- RetentionPolicy.SOURCE - 只在源代码中起作用
- RetentionPolicy.CLASS - 保留到.class文件中
- RetentionPolicy.RUNTIME - 在运行期仍然起作用（通常我们都用这个）

**@Inherited**

添加@Inherited元注解说明这个注解可以被子类继承。

**@Documented**

添加@Documented元注解说明可以被JavaDoc文档化。

上述四个元注解中最常用的是@Target和@Retention。

#### 属性

注解可以拥有属性，虽然下面@Component注解中的value()写法看上去像是一个方法，但实际上它是一个属性，value()里面不允许有参数。注解可以拥有多个属性，可以通过`default` 给属性设置默认值。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Component {
   String value() default "";
}
```

#### 使用

以@Component注解为例，@Target声明为ElementType.TYPE，说明注解的作用域在类上。属性可以在()里面根据属性名进行设置，如果不设置使用default值。

```java
@Component(value="user")
public class User {    
}
```

注意：value是一个特殊属性，value是注解的默认属性，如果只有一个属性并且是value，那么可以写成这样

```java
@Component("user")
public class User {    
}
```

这里没有指定属性名，就使用默认的属性名value。

### 自定义注解

相信大家对于使用JDK和Spring提供的常见注解都不陌生，下面看看如何自定义注解并使用。

下面以实体类对象为例，我们定义两个注解@Table和@Column。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Table {
    String name() default "";
}
```

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
    String name() default "";
}
```

我们定义@Table注解作用域在类上，@Column注解作用域在类的成员变量上，都有一个name属性。我们再来定义一个User实体类，与user表对应，这里省略了get/set方法。

```java
@Table(name = "user")
public class User {
    @Column(name = "user_id")
    private Long userId;
    @Column(name = "user_name")
    private String userName;
    @Column(name = "user_mobile")
    private String userMobile;
}
```

### 反射读取注解示例

下面我们看一下自定义的注解如何起作用。我们通过自定义注解自动拼接SQL查询语句，如果字段值不为null就作为查询条件。

```java
protected String query(Object object) throws IllegalAccessException {
  StringBuffer buffer = new StringBuffer();
  Class clazz = object.getClass();
  // 判断object对象的类是否定义了@Table注解
  if(!clazz.isAnnotationPresent(Table.class)) {
    return null;
  }
  // 读取@Table注解信息
  Table annotationTable = (Table)clazz.getAnnotation(Table.class);
  buffer.append("select * from ");
  buffer.append(annotationTable.name());
  buffer.append(" where 1=1");
  // 反射读取类的全部成员变量
  Field[] fieldList = clazz.getDeclaredFields();    
  for (int i=0; i<fieldList.length; i++) {
    Field field = fieldList[i];
    // 设置可访问非公有成员变量
    field.setAccessible(true);
    Object value = field.get(object);
    // 跳过未赋值的成员变量
    if (null == value) {
      continue;
    }
    // 检查成员变量是否声明了@Column注解
    if (!field.isAnnotationPresent(Column.class)) {
      continue;
    }
    // 读取@Column注解信息
    Column annotationColumn = (Column)field.getAnnotation(Column.class);
    buffer.append(" and ");
    buffer.append(annotationColumn.name());
    buffer.append("=");
    if (value instanceof String) {
      buffer.append("'");
      buffer.append(value);
      buffer.append("'");
    } else {
      buffer.append(value);
    }
  }
  // 返回拼接的SQL查询语句
  return buffer.toString();
}
```

我们做如下测试

```java
User user = new User();
user.setUserId(1001L);
user.setUserName("test");
System.out.println(query(user));
```

输出结果为

```shell
select * from user where 1=1 and user_id=1001 and user_name='test'
```

### Spring读取注解示例

实战中我们往往使用Spring的IoC容器来管理类对象，我们再来看看如何在Spring中如何读取到注解信息。

首先，User类需要增加@Component注解，加入到Spring的IoC容器中进行管理

```java
@Table(name = "user")
@Component("user")
public class User {
    @Column(name = "user_id")
    private Long userId;
    @Column(name = "user_name")
    private String userName;
    @Column(name = "user_mobile")
    private String userMobile;
}
```

#### ApplicationContextAware

第一种方法，实现ApplicationContextAware接口。

```java
@Configuration
public class MybatisAutoConfiguration implements ApplicationContextAware {
  protected ApplicationContext context;
  @Override
  public void setApplicationContext(ApplicationContext context) throws BeansException {
    this.context = context;
  }
  @PostConstruct
  public void init() throws Exception {
    Map<String, Object> beans = context.getBeansWithAnnotation(Table.class);
    if (beans.size() > 0) {
      for (Map.Entry<String, Object> entry : beans.entrySet()) {
        String beanName = entry.getKey();
        Object object = entry.getValue();
        Table annotationTable = context.findAnnotationOnBean(beanName, Table.class);
        String tableName = annotationTable.name();
        System.out.println("tableName=" + tableName);
      }
    }
  }
```

核心是调用容器context的getBeansWithAnnotation()方法获取所有声明了@Table注解的Bean对象，然后再调用context的findAnnotationOnBean()方法读取@Table注解信息。

#### BeanPostProcessor

第二种方法，实现BeanPostProcessor接口。

上面的方法虽然也能实现，但最正统的应该是实现BeanPostProcessor接口，它保证在每个Bean创建完成后给我们一个定制化处理的机会。

> TODO：使用ApplicationContextAware需要保证执行@PostConstruct时所有对象都已经创建完毕，否则就可能出现遗漏的情况。实战过程中还没有出现过遗漏的情况，原因不清楚。

```java
@Configuration
public class BeanPostConfiguration implements BeanPostProcessor {
  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) 
    throws BeansException {
    return bean;
  }

  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) 
    throws BeansException {
    Class clazz = bean.getClass();
    if(!clazz.isAnnotationPresent(Table.class)) {
      return bean;
    }
    Table annotationTable = (Table)clazz.getAnnotation(Table.class);
    System.out.println(annotationTable.name());
    return bean;
  }
}
```

在IoC容器实例化每一个Bean对象以后都会调用postProcessAfterInitialization()方法，所以在这里增加对注解的处理是合适的。引申开来，我们甚至可以进行定制化修改，不返回bean对象，返回其他定制化对象。



[Spring探秘|妙用BeanPostProcessor](https://www.jianshu.com/p/1417eefd2ab1)



