---
title: Java基础-动态代理
date: 2018-05-28 17:42:24
categories: JAVA
tags: [Java, 动态代理]
toc: true
description: 本文从静态代理到动态代理，由浅入深逐步分析引入动态代理的目的以及JDK动态代理的原理。本文详细分析了Proxy.newProxyInstance()方法实现原理，并对源码进行解析。
comments: true
---

## JDK动态代理

动态代理是AOP面向接口编程的基础。

### 静态代理

解释动态代理概念前我们先来看看什么是静态代理。

非常常见的代码，我们定义UserMapper接口和它的实现类UserMapperImpl，然后我们又定义了代理类UserMapperProxy，代理类同样实现UserMapper接口，但它把真正的工作交给UserMapperImpl去做。

```java
public interface UserMapper {
  void save(User user);
}
```

```java
public class UserMapperImpl implements UserMapper {
  @Override
  public void save(User user) {
    System.out.println("UserMapperImpl save()");
  }
}
```

```java
public class UserMapperProxy implements UserMapper {
  private UserMapperImpl userMapperImpl;
  public UserMapperProxy(UserMapperImpl userMapperImpl) {
    this.userMapperImpl = userMapperImpl;
  }
  @Override
  public void save(User user) {
    System.out.println("UserMapperProxy before save()");
    userMapperImpl.save(user);
    System.out.println("UserMapperProxy after save()");
  }
}
```

使用时的代码如下，这就是静态代理。UserMapperProxy作为UserMapperImpl的代理，接收外部请求，可以在实际处理逻辑前后或者抛出异常时增加自己的处理。UserMapperProxy实际上就起到了切入的作用，但它是静态了，需要硬编码。所谓动态代理，个人理解就是UserMapperProxy类不用硬编码了，我们可以在运行时动态生成UserMapperProxy类的代码。

```java
UserMapper userMapper = new UserMapperProxy(new UserMapperImpl());
userMapper.save(new User());
```

### 动态代理

动态代理有两种常见的实现方式：JDK实现和CGLib实现，本文只关注JDK实现方式。

下面看看基于JDK动态代理如何实现上面的效果。注意：JDK只能基于接口做动态代理。

首先，我们创建DynamicProxy类。

- DynamicProxy类的作用是生成代理对象；
- DynamicProxy类必须继承InvocationHandler接口，并实现invoke()方法；
- 核心代码是`Proxy.newProxyInstance()` ，这个方法创建了代理对象。

```java
public class DynamicProxy implements InvocationHandler {
  private Object target;
  public Object bind(Object target) {
    this.target = target;
    return Proxy.newProxyInstance(target.getClass().getClassLoader(), 
                                  target.getClass().getInterfaces(), this);
  }
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("proxy invoke begin");
    Object result = method.invoke(target, args);
    System.out.println("proxy invoke end");
    return result;
  }
}
```

下面看看如何使用

```java
DynamicProxy proxy = new DynamicProxy();
UserMapper userMapper = (UserMapper) proxy.bind(new UserMapperImpl());
userMapper.save(new User());
```

#### 为什么用动态代理

我们再来比对一下静态代理和动态代理的代码，看上去差不多啊，没看出来使用动态代理有什么好处啊，为什么要使用动态代理呢？

```java
// 静态代理
UserMapper userMapper = new UserMapperProxy(new UserMapperImpl());
userMapper.save(new User());
// 动态代理
UserMapper userMapper = (UserMapper) new DynamicProxy().bind(new UserMapperImpl());
userMapper.save(new User());
```

首先，假设我们在UserMapper中增加一个update()方法，那么UserMapperImpl肯定要实现这个新增的update()方法，这个跑不了。我们再来看看静态代理和动态代理都需要做出哪些改变：

- 静态代理：UserMapperProxy类由于实现了UserMapper接口，所以必须要修改UserMapperProxy类实现update()方法，在update()方法中调用UserMapperImpl并输出日志。
- 动态代理：DynamicProxy类不用修改就可以用。

```java
public interface UserMapper {
  void save(User user);
  int update(User user);
}
```

接下来，假设我们添加一个新的接口AccountMapper和它的实现类AccountMapperImpl，那么静态代理和动态代理又要做出哪些改变呢：

- 静态代理：UserMapperProxy不能用了，我们还得再定义一个AccountMapperProxy类；
- 动态代理：DynamicProxy类不用修改就可以用。

看到动态代理的好处了吧，这就是使用动态代理的原因。

#### 实现原理

动态代理到底都做了什么？简单说，就是动态生成了XxxProxy代理类，这应该也是动态代理这个名字的由来。从上面的例子我们可以看到，静态代理麻烦之处就在于每次接口发生改变，代理类就得随之改变，怎么解决这个问题呢？那就是代理类不是在编译期生成的，而是在运行期生成的，在运行期根据接口内容动态生成代理类，自然就能满足要求。所以，动态代理可以认为是一种动态生成Java字节码的技术。

添加JVM参数`-Dsun.misc.ProxyGenerator.saveGeneratedFiles=true` ，重新运行前面的例子，可以在项目根目录下的com/sun/proxy目录下找到自动生成的代理类$proxy?.class。

> 我这里是$Proxy2.class

Proxy2就是bind()时返回的代理类，所以userMapper.save()实际上调用的是Proxy2的save()方法，这个方法调用了InvocationHandler接口的invoke()方法（所以DynamicProxy一定要实现InvocationHandler接口），invoke()方法传递了三个参数，this就是$Proxy2类实例，m4是通过反射得到的UserMapper接口的save()方法，var1是透传的参数。

```java
public final class $Proxy2 extends Proxy implements UserMapper {
  private static Method m3;
  private static Method m4;
  static {
    m3 = Class.forName("UserMapper").getMethod("update", new Class[]{Class.forName("User")});
    m4 = Class.forName("UserMapper").getMethod("save", new Class[]{Class.forName("User")});
  }  
  public $Proxy2(InvocationHandler var1) throws  {
    super(var1);
  }  
  public final int update(User var1) throws  {
    return ((Integer)super.h.invoke(this, m3, new Object[]{var1})).intValue();
  }
  public final void save(User var1) throws  {
    super.h.invoke(this, m4, new Object[]{var1});
  }
}
```

```java
public class Proxy {
  protected InvocationHandler h;  
  protected Proxy(InvocationHandler h) {
    this.h = h;
  }  
}
```

> 个人理解：根据实现类的实例对象通过反射机制自动生成了实现了接口的代理类实例对象，这就是动态代理。

DynamicProxy类的bind()方法是整个动态代理实现的核心，我们来详细解读一下：

- 首先，bind()方法的入参和返回值都是一个对象：入参是实现类实例对象，返回值是代理类实例对象；
- target.getClass().getInterfaces()得到了入参对象实现的接口类信息（可以是多个）；
- Proxy.newProxyInstance()方法传入接口类信息和回调Handler，生成代理类实例对象并返回：
  - 自动生成$Proxy0类，继承Proxy类，实现传入的接口类；
  - 通过反射获取接口类的所有方法，在$Proxy0类中自动生成这些方法的实现；
  - $Proxy0类中保存了通过反射得到的接口类的方法；
  - $Proxy0类在接口类实现方法中调用Handler的invoke()方法，传入反射Method作为参数；
- DynamicProxy类的invoke()方法中，通过反射调用target对象的Method方法，虽然Method方法是通过对接口的反射得到的，由于接口是从target对象的类信息中提取的，所以target一定有这个方法。

**核心要点**

- 自动生成$Proxy0类并通过反射创建它的实例对象；
- 根据实现类提取接口，在代理类中实现这些接口，并在代理类中通过反射得到接口的成员方法；
- DynamicProxy类拥有target对象，invoke()回调传递了方法，就可以通过反射来执行了。

### 源码分析

#### Proxy

以上我们了解了JDK动态代理的原理，知道最核心的是Proxy.newProxyInstance()方法，下面就来看看它的源码。

```java
package java.lang.reflect;
public class Proxy implements java.io.Serializable {
  public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, 
                                        InvocationHandler h) {
    Class<?> cl = getProxyClass0(loader, intfs);
    final Constructor<?> cons = cl.getConstructor(constructorParams);
    cons.setAccessible(true);
    return cons.newInstance(new Object[]{h});
  }      
}
```

调用getProxyClass0()获取`$Proxy0`类信息，反射获取构造函数，以h为参数构造`$Proxy0`类的实例对象并返回。

```java
package java.lang.reflect;
public class Proxy implements java.io.Serializable {    
  private static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
    return proxyClassCache.get(loader, interfaces);
  }
}
```

```java
package java.lang.reflect;
final class WeakCache<K, P, V> {
  public V get(K key, P parameter) {
    // 这里很复杂，做了简化
    Factory factory = new Factory(key, parameter, subKey, valuesMap);
    V value = factory.get();
    return value;
  }
  private final class Factory implements Supplier<V> {
    @Override
    public synchronized V get() { 
      V value = Objects.requireNonNull(valueFactory.apply(key, parameter));
      return value;
    }
  }
}
```

getProxyClass0()方法优先从缓存中读取，如果读取不到使用factory创建，最终调用Proxy类的内部类ProxyClassFactory的apply()方法。

```java
package java.lang.reflect;
public class Proxy implements java.io.Serializable {  
  private static final class ProxyClassFactory {
    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
      // 示意
      String proxyName = "com.sun.proxy.$Proxy0";      
      byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, interfaces, 
                                                                accessFlags);
      return defineClass0(loader, proxyName, proxyClassFile, 0, proxyClassFile.length);
    }
  }
}
```

最后调用ProxyGenerator.generateProxyClass()方法来动态创建代理类。代理类的名字是在Proxy中确定的，具体如何生成代码在sun的package下。

#### ProxyGenerator

ProxyGenerator类在rt.jar中，但不在`java.*` 包下，在`sun.*` 包下。generateProxyClass()静态方法创建ProxyGenerator对象并调用它的generateClassFile()方法。

> JDK中默认不包括ProxyGenerator类的源码，以下内容为摘抄并整理。

```java
package sun.misc;
public class ProxyGenerator {
  private String className;
  private Class<?>[] interfaces;
  private int accessFlags;
  private byte[] generateClassFile() {
    // 组装ProxyMethod对象
    // 为代理类生成toString, hashCode, equals等方法
    addProxyMethod(hashCodeMethod, Object.class);
    addProxyMethod(equalsMethod, Object.class);
    addProxyMethod(toStringMethod, Object.class);
    // 遍历每一个接口的每一个方法, 并且为其生成ProxyMethod对象
    for (int i = 0; i < interfaces.length; i++) {
      Method[] methods = interfaces[i].getMethods();
      for (int j = 0; j < methods.length; j++) {
        addProxyMethod(methods[j], interfaces[i]);
      }
    }

    try {
      // 添加构造函数
      // 例如: public $Proxy2(InvocationHandler var1) throws { super(var1); }
      methods.add(generateConstructor());
      // 遍历前面生成的代理方法
      for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
        for (ProxyMethod pm : sigmethods) {
          // 添加代理类的静态字段
          // 例如: private static Method m2;
          fields.add(new FieldInfo(pm.methodFieldName, "Ljava/lang/reflect/Method;", 
                                   ACC_PRIVATE | ACC_STATIC));
          // 添加代理类的代理方法
          // 真正实现每一个方法体
          methods.add(pm.generateMethod());
        }
      }
      // 添加代理类的静态字段初始化方法
      // 例如：m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      methods.add(generateStaticInitializer());
    } catch (IOException e) {
      throw new InternalError("unexpected I/O Exception");
    }

    // 写入文件流
    ByteArrayOutputStream bout = new ByteArrayOutputStream();
    DataOutputStream dout = new DataOutputStream(bout);
    try {
      //1.写入魔数
      dout.writeInt(0xCAFEBABE);
      //2.写入次版本号
      dout.writeShort(CLASSFILE_MINOR_VERSION);
      //3.写入主版本号
      dout.writeShort(CLASSFILE_MAJOR_VERSION);
      //4.写入常量池
      cp.write(dout);
      //5.写入访问修饰符
      dout.writeShort(ACC_PUBLIC | ACC_FINAL | ACC_SUPER);
      //6.写入类索引
      dout.writeShort(cp.getClass(dotToSlash(className)));
      //7.写入父类索引, 生成的代理类都继承自Proxy
      dout.writeShort(cp.getClass(superclassName));
      //8.写入接口计数值
      dout.writeShort(interfaces.length);
      //9.写入接口集合
      for (int i = 0; i < interfaces.length; i++) {
        dout.writeShort(cp.getClass(dotToSlash(interfaces[i].getName())));
      }
      //10.写入字段计数值
      dout.writeShort(fields.size());
      //11.写入字段集合 
      for (FieldInfo f : fields) {
        f.write(dout);
      }
      //12.写入方法计数值
      dout.writeShort(methods.size());
      //13.写入方法集合
      for (MethodInfo m : methods) {
        m.write(dout);
      }
      //14.写入属性计数值, 代理类class文件没有属性所以为0
      dout.writeShort(0);
    } catch (IOException e) {
      throw new InternalError("unexpected I/O Exception");
    }
    //转换成二进制数组输出
    return bout.toByteArray();
  }  
}
```

首先从方法名称generateClassFile()就可以知道这是直接生成.class文件，不是生成.java文件，所以这个方法的任务是按照.class文件格式写入数据流。

- addProxyMethod()把要生成的方法加入到proxyMethods中；
- fields是需要写入到.class文件中的成员变量；
- methods是需要写入到.class文件中的方法。

## CGLIB动态代理

JDK动态代理必须基于接口，如果没有接口就不能使用JDK动态代理。

CGLIB是另外一种动态代理实现，不依赖于接口。





