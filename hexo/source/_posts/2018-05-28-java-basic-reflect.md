---
title: Java基础-反射
date: 2018-05-28 08:50:00
categories: JAVA
tags: [JAVA, 反射]
toc: true
description: 反射是Java语言的一种特性，通过反射机制，应用程序可以在运行期动态加载类。常用的反射类：Class、Method、Filed和Constructor，常用方法：newInstance()和invoke()：
comments: false
---

## Java反射机制

### 什么是反射？

反射是Java语言的一种特性，通过反射机制，应用程序可以在运行期动态读取类信息，并访问类实例、类成员方法和类成员变量。

所谓反射，个人理解就是反向映射，反向自然是和正向相对的。所谓正向指在编译期就知道类的内部结构；所谓反向指在编译期不知道类的内部结构，在运行期动态加载类以后才知道类的内部结构是怎么样的。编译器不知道也无需知道实现类的内部细节，也就意味着解耦合。通常我们在应用程序中声明接口，运行时利用反射机制动态加载实现类，这样就实现了应用程序和具体实现类的解耦，当具体实现类的实现细节发生改变，甚至改变了实现类时，应用程序本身都不需要进行调整。

反射非常非常重要，反射是Spring IoC和AOP的基础，也可以说反射是Spring框架的基础。IoC容器需要使用反射API来创建Bean对象，AOP中需要通过反射加载生成的代理类。

#### 类对象

我们先来澄清一些概念：

- `public class User {}` 我们称之为一个类，User类；
- `private String name;` 我们称之为类的成员变量，name是User类的私有成员变量；
- `public String getName() {}` 我们称之为类的成员方法，getName()是User类的公有成员方法；
- `User user = new User();` 我们称之为创建了一个类实例对象，user对象是User类的一个实例对象。

```java
package org.test;
public class User {
  private String name;
  public String getName() {
    return name;
  }
}
```

```java
User user = new User();
```

我们知道user是User类的一个实例对象，那么什么是类对象呢？调用user对象的getClass()方法就返回一个类对象。注意：user是类实例对象，clazz是类对象，它们有什么区别呢？

- 首先，user是User类的实例化，user对象的堆内存空间是User类的结构，里面保存name等属性；clazz是Class类的实例化，clazz对象的堆内存空间是Class类的接口，里面保存类名、类加载器、属性、方法等信息；
- 每个类被JVM加载到内存后，都有对应的Class类对象，类对象的数据结构是相同，但属性值不同；每个类被JVM加载到内存后，也会生成类的实例对象，类的实例对象的数据结构是互不相同的；

```java
Class clazz = user.getClass();
```

我们有三种方法来获取一个类的类对象，最后一种就是通过反射获取到类对象，它是以后所有反射操作的基础。

```java
User user = new User();
Class c1 = User.class;
Class c2 = user.getClass();
Class c3 = Class.forName("org.test.User");
```

> TODO：以下挪到JVM-类加载器中

我们知道在JVM规范中，User类信息存储方法区（method area）上，我们常用的Hotspot虚拟机把User类信息存储在堆的永久代上。User类对象在堆上创建，User类对象信息自然存储在堆上。user变量是对User类对象的应用，user变量存储在栈上。

![](/images/java-basic-reflect-class-object.png)

user变量是一个引用，指向User类实例对象；User类实例对象的数据结构就是User类的数据结构；那么User类信息在永久代上具体是如何存储的呢？我们可以认为，永久代上存储的也是一个类对象，Class类的对象。

```java
package java.lang;
public final class Class<T> {
  private transient String name;
  private final ClassLoader classLoader;  
}
```

我们再来看一下`User user = new User();` 的执行过程：

- 编译User.java生成User.class文件；
- JVM使用类加载器读取User.class文件，生成一个Class对象，放入永久代堆；
- JVM根据Class对象中的信息创建一个User类的实例对象，放入堆中；
- JVM在当前栈中添加user变量，指向堆中User类实例对象。

啰嗦这么多就是想说：这就是正向创建对象的过程，加载Class类信息的操作由JVM完成，对应用程序是透明的。那什么是反射呢？个人理解就是应用程序手动来完成原本JVM自动完成的那些操作。

下面看一个例子

```java
User user = new User();
Class c1 = User.class;
Class c2 = user.getClass();
Class c3 = Class.forName("org.test.User");
System.out.println(c1.hashCode());
System.out.println(c2.hashCode());
System.out.println(c3.hashCode());
```

运行后返回的hashCode是一样的，说明三个Class类对象是一个对象。也就是说：使用反射方法得到的类对象c3和JVM正常加载User类得到的c1和c2都指向同一个类对象。

#### 动态加载

还是上面的例子，c1和c2在编译时就需要加载User类，这称为静态加载。c3在编译时不需要加载User类，在运行时加载User类，这称为动态加载。从类加载的角度说，反射最重要的特性就是实现了运行时类的动态加载。

> TODO：以上挪到JVM-类加载器中

### 反射的使用

简单说，反射机制就是Java为应用程序提供了在程序运行期间获取类对象并对其进行操作的能力。

反射主要涉及到以下几个类：Class、Method、Field。

#### Class

得到类对象信息后我们就可以动态创建类实例对象了，常见有两种方法。

**Class.newInstance()**

- newInstance()相当于执行不带参数的默认构造函数
- User类可以不声明任何构造函数，也可以声明默认构造函数`public User() {}`，但是不能只声明带参数的构造函数；例如：如果声明`public User(String name)` ，那么newInstance()找不到合适的构造函数将抛出异常

```java
import org.test;
public class User {
  private String name;
}
```

```java
Class clazz = Class.forName("org.test.User");
User user = (User) clazz.newInstance();
```

**Constructor.newInstance()**

上面的方法是不能带参数的，如果需要执行带参数的构造函数，需要调用Constructor.newInstance()

- 先调用Class类的getConstructor()方法获取构造函数Constructor
- 然后再调用Constructor类的newInstance()方法并传入参数

```java
import org.test;
public class User {
  private String name;
  public User(String name) {
    this.name = name;
  }
}
```

```java
Class clazz = Class.forName("org.test.User");
Constructor constructor = clazz.getConstructor(String.class);
User user = (User) constructor.newInstance("test");
```

上面例子中如果User类构造函数声明为private，那么getConstructor()时将抛出NoSuchMethodException异常，怎么办？可以将getConstructor()替换为getDeclaredConstructor()，前者只能获取到public的构造函数，后者可以获取到User类声明的所有构造函数。

```java
Class clazz = Class.forName("org.test.User");
Constructor constructor = clazz.getDeclaredConstructor(String.class);
User user = (User) constructor.newInstance("test");
```

运行之后发现还不行，抛出IllegalAccessException异常，怎么办？通过getDeclaredConstructor()我们可以得到私有构造方法了，但是没有private方法的执行权限，需要调用setAccessible()方法设置权限。

```java
Class clazz = Class.forName("org.test.User");
Constructor constructor = clazz.getDeclaredConstructor(String.class);
constructor.setAccessible(true);
User user = (User) constructor.newInstance("test");
```

> getXXX()可以获得public的方法，包括继承的方法；getDeclaredXXX()可以获得所有方法，但是只能获得自己声明的方法，不能获得继承的方法。

#### Method

从Class中可以获取类的成员方法Method，调用Method类的invoke()方法可以执行成员方法。

```java
import org.test;
public class User {
  private String name;
  public String getName() {
    return name;
  }
  public void setName(String name) {
    this.name = name;
  }  
}
```

```java
Class clazz = Class.forName("org.test.User");
Constructor constructor = clazz.getDeclaredConstructor();
constructor.setAccessible(true);
User user = (User) constructor.newInstance();
Method method = clazz.getMethod("setName", String.class);
method.invoke(user, "test");
method = clazz.getMethod("getName", null);
String name = (String) method.invoke(user);
```

同样的，如果需要访问私有方法，可以将getMethod()替换为getDeclaredMethod()并在执行invoke()方法前setAccessible(true)。

```java
Method method = clazz.getDeclaredMethod("setName", String.class);
method.setAccessible(true);
method.invoke(user, "test");
```

#### Field

从Class中可以获取类的成员变量Field，调用Field类的get()和set()方法可以访问成员变量。通过Field即使没有提供get()和set()方法，我们也可以直接访问User类的私有成员变量name进行赋值。

```java
import org.test;
public class User {
  private String name;
}
```

```java
clazz = Class.forName("org.test.User");
Constructor constructor = clazz.getDeclaredConstructor();
constructor.setAccessible(true);
User user = (User) constructor.newInstance();
Field field = clazz.getDeclaredField("name");
field.setAccessible(true);
field.set(user, "test");
String name = (String) field.get(user);
```

#### PropertyDescriptor

由于通常我们使用的Bean对象都会定义get()和set()方法，Java为我们提供了PropertyDescriptor类来简化操作。

- `new PropertyDescriptor()` 时传入属性名称；
- `getReadMethod();` 返回get()方法； `getWriteMethod();` 返回set()方法。

```java
PropertyDescriptor descriptor = new PropertyDescriptor("name", clazz);
Method method = descriptor.getWriteMethod();
method.setAccessible(true);
method.invoke(user, "test");
String name = (String)descriptor.getReadMethod().invoke(user);
System.out.println(name);
```

### 反射包

最后，我们罗列一下反射相关的重要类和方法。

#### java.lang.Class

| 方法                        | 作用                   |
| ------------------------- | -------------------- |
| forName()                 | 动态加载类并返回类对象          |
| getName()                 | 获取类的名字               |
| getSuperclass()           | 获取基类                 |
| getInterfaces()           | 获取类实例对象实现的接口类        |
| getConstructors()         | 获取所有公有构造函数，包括父类的构造函数 |
| getMethods()              | 获取所有公有成员方法，包括父类的成员方法 |
| getFields()               | 获取所有公有成员变量，包括父类的成员变量 |
| getDeclaredConstructors() | 获取所有构造函数，不包括父类的构造函数  |
| getDeclaredMethods()      | 获取所有成员方法，不包括父类的成员方法  |
| getDeclaredFields()       | 获取所有成员变量，不包括父类的成员变量  |
| getConstructor()          | 获取指定公有构造函数，包括父类的构造函数 |
| getMethod()               | 获取指定公有成员方法，包括父类的成员方法 |
| getField()                | 获取指定公有成员变量，包括父类的成员变量 |
| getDeclaredConstructor()  | 获取指定构造函数，不包括父类的构造函数  |
| getDeclaredMethod()       | 获取指定成员方法，不包括父类的成员方法  |
| getDeclaredField()        | 获取指定成员变量，不包括父类的成员变量  |
| isAnnotationPresent()     | 类上是否声明了指定注解          |
| getAnnotations()          | 获取类上声明的所有注解          |
| getAnnotation()           | 获取类上声明的指定注解          |
|                           |                      |

#### java.lang.reflect.Constructor

| 方法                                 | 作用             |
| ---------------------------------- | -------------- |
| T newInstance(Object ... initargs) | 执行构造函数创建类的实例对象 |
|                                    |                |

#### java.lang.reflect.Method

| 方法                                       | 作用              |
| ---------------------------------------- | --------------- |
| Object invoke(Object obj, Object... args) | 执行方法，传入类实例对象和参数 |
|                                          |                 |

#### java.lang.reflect.Field

| 方法                                 | 作用               |
| ---------------------------------- | ---------------- |
| Object get(Object obj)             | 读取属性值，传入类实例对象    |
| void set(Object obj, Object value) | 设置属性值，传入类实例对象和新值 |
|                                    |                  |

#### java.lang.reflect.InvocationHandler

```java
public interface InvocationHandler {
  // 
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

#### java.lang.reflect.Proxy

```java
public class Proxy implements java.io.Serializable {
  // 
  public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,
                                        InvocationHandler h) {
  }     
}
```

