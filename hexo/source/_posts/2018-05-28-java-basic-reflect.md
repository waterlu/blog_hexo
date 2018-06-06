---
title: Java基础-反射
date: 2018-05-28 08:50:00
categories: JAVA
tags: [Java, 反射]
toc: true
description: 反射是Java语言的一种特性，通过反射机制，应用程序可以在运行期动态加载类信息。常用的类有Class、Method、Filed、Constructor和Proxy，常用的方法有newInstance()和invoke()。
comments: true
---

## Java反射机制

### 什么是反射？

反射是Java语言的一种特性，通过反射机制，应用程序可以在运行期动态读取类信息，并访问类实例、类成员方法和类成员变量。

所谓反射，个人理解就是反向映射，反向自然是和正向相对的。所谓正向指在编译期就知道类的内部结构；所谓反向指在编译期不知道类的内部结构，在运行期动态加载类以后才知道类的内部结构是怎么样的。编译器不知道也无需知道实现类的内部细节，也就意味着解耦合。通常我们在应用程序中声明接口，运行时利用反射机制动态加载实现类，这样就实现了应用程序和具体实现类的解耦，当具体实现类的实现细节发生改变，甚至改变了实现类时，应用程序本身都不需要进行调整。

反射非常重要，反射是Spring IoC和AOP的基础，也可以说反射是Spring框架的基础，无反射不框架。

#### 类对象

我们先来澄清一些概念：

- `public class User {}` 我们称之为一个类，User 表示User类；
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
```java
Class clazz = user.getClass();
```

我们知道user是User类的一个实例对象，那么什么是类对象呢？调用user对象的getClass()方法就返回一个类对象。上面的代码中：user是类实例对象，clazz是类对象，它们有什么区别呢？

- 首先，user是User类的实例化，user对象的堆内存空间是User类的结构，里面保存name等属性；clazz是Class类的实例化，clazz对象的堆内存空间是Class类的接口，里面保存类名、类加载器、属性、方法等信息；
- 每个类被JVM加载到内存后，都有对应的Class类对象，类对象的数据结构是相同，但属性值不同；每个类被JVM加载到内存后，也会生成类的实例对象，类的实例对象的数据结构是互不相同的；

我们有三种方法来获取一个类的类对象，最后一种就是通过反射获取到类对象，它是以后所有反射操作的基础。

```java
User user = new User();
Class c1 = User.class;
Class c2 = user.getClass();
Class c3 = Class.forName("org.test.User");
```

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

```java
package java.lang;
public final class Class<T> {
  // 动态加载类并返回类对象
  public static Class<?> forName(String className) throws ClassNotFoundException {}
  // 获取类的名字
  public String getName() {}
  // 获取基类
  public native Class<? super T> getSuperclass();
  // 获取实现的接口类
  public Class<?>[] getInterfaces() {}
  // 获取所有公有构造函数，包括父类的构造函数
  public Constructor<?>[] getConstructors() throws SecurityException {}
  // 获取所有构造函数，不包括父类的构造函数
  public Constructor<?>[] getDeclaredConstructors() throws SecurityException {}
  // 获取指定公有构造函数，包括父类的构造函数
  public Constructor<T> getConstructor(Class<?>... parameterTypes) {}
  // 获取指定构造函数，不包括父类的构造函数
  public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes) {}
  // 获取所有公有成员方法，包括父类的成员方法
  public Method[] getMethods() throws SecurityException {}
  // 获取所有成员方法，不包括父类的成员方法
  public Method[] getDeclaredMethods() throws SecurityException {}
  // 获取指定公有成员方法，包括父类的成员方法
  public Method getMethod(String name, Class<?>... parameterTypes) {}
  // 获取指定成员方法，不包括父类的成员方法
  public Method getDeclaredMethod(String name, Class<?>... parameterTypes) {}
  // 获取所有公有成员变量，包括父类的成员变量
  public Field[] getFields() throws SecurityException {}
  // 获取所有成员变量，不包括父类的成员变量
  public Field[] getDeclaredFields() throws SecurityException {}
  // 获取指定公有成员变量，包括父类的成员变量
  public Field getField(String name) {}
  // 获取指定成员变量，不包括父类的成员变量
  public Field getDeclaredField(String name) {}
  // 类上是否声明了指定注解
  public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {}
  // 获取类上声明的所有注解
  public Annotation[] getAnnotations() {}
  // 获取类上声明的指定注解
  public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {}
}
```

#### java.lang.reflect.Constructor

```java
package java.lang.reflect;
public final class Constructor<T> extends Executable {
  // 执行构造函数创建类的实例对象
  public T newInstance(Object ... initargs) {}
}
```

#### java.lang.reflect.Method

```java
package java.lang.reflect;
public final class Method extends Executable {
  // 执行方法，传入类实例对象和参数
  public Object invoke(Object obj, Object... args) {}
}
```

#### java.lang.reflect.Field

```java
package java.lang.reflect;
class Field extends AccessibleObject implements Member {
  // 读取属性值，传入类实例对象
  public Object get(Object obj) {}
  // 设置属性值，传入类实例对象和新值
  public void set(Object obj, Object value) {}
}
```

#### java.lang.reflect.InvocationHandler

```java
public interface InvocationHandler {
  // 执行代理方法的回调入口
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

#### java.lang.reflect.Proxy

```java
public class Proxy implements java.io.Serializable {
  // 生成动态代理类实例对象
  public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,
                                        InvocationHandler h) {}     
}
```

