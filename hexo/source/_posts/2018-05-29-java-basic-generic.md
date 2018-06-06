---
title: Java基础-泛型
date: 2018-05-29 16:11:12
categories: JAVA
tags: [Java, 泛型]
toc: true
description: 泛型是参数化类型，也就是类型成为了变量。泛型只在编译期有效，运行期实际上泛型是不存在的，这称为类型擦除。本文给出了实战中使用泛型Class对象和读取泛型真实类型的例子代码。
comments: true
---

## Java泛型

### 泛型

泛型简单理解就是类型可以是变量，原本需要声明一个固定的类，现在变成了变量，这就是泛型。

泛型的好处是在编译期就可以进行类型检查，避免出现错误。

常见的List类就声明了泛型，如果声明`List<String> list;` 那么list中就只能添加`String`对象，否则编译时会报错。

```java
public interface List<E> extends Collection<E> {
}
```

这里E可以使用任何字母，都没有关系，都表示一个待确定的类。一般习惯上经常使用的泛型字母有E、T、K、V。

泛型除了可以放在类上，还可以放在方法上，既可以声明参数，也可以声明返回值。

```java
public interface Map<K,V> {
  V get(Object key);
  V put(K key, V value);
}
```

泛型除了声明方法外，还可以直接放入代码中代替类声明

```java
for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
  K key = e.getKey();
  V value = e.getValue();
  putVal(hash(key), key, value, false, evict);
}
```

### 限定子类

以上声明的泛型可以被替换成任何类，如果需要限定类型范围，可以使用extends和super关键字

假设继承关系： Man -> Human -> Object

```java
// 可以使用Man或者Human，从Human类开始向下找，Human类和它的子类都可以
public class Test<? extends Human> {
}
```

```java
// 可以使用Human或者Object，从Human类开始向上找，Human类和它的父类都可以
public class Test<? super Human> {
}
```

除了在声明类和方法时使用extends和super关键字，也可以在引用时使用

```java
List<? extends Integer> list1 = new ArrayList<>();
List<? super Integer> list2 = new ArrayList<>();
```

### 类型擦除

类型擦除是Java泛型中比较绕的一点，类型擦除的意思是：泛型只在编译期有效，运行期实际上泛型是不存在的。

```java
public class Test<T> {
  T data;
}
```

在运行期间等同于

```java
public class Test<Object> {
  Object data;
}
```

也就是说，泛型的类型检查只在编译期起作用，在运行期实际上已经没有用了。这么做的原因是为了兼容，JDK1.5以后才引入泛型。

我们来看一个例子，运行结果输入`[1, test]` 。虽然list声明为只接收Integer元素，但是当我们通过反射获取到list的add()方法后，是可以添加字符串进去的。这个例子充分说明：泛型的类型检查只在编译期有效，运行期是没有类型检查的。

```java
List<Integer> list = new ArrayList<>();
list.add(1);
Class<? extends List> clazz = list.getClass();
Method addMethod = clazz.getMethod("add", Object.class);
addMethod.invoke(list, "test");
System.out.println(list);
```

> 运行期类型检查被去掉了，就叫做类型擦除。

由于类型擦除问题的存在，Java不允许创建泛型数组。

```java
List<Integer> [] listArray = new List<Integer>[2]; // Generic Array Creation
```

上面这样的写法是无法编译通过的。

为什么？因为Java对于数组[]有自己执着的要求，必须知道数组[]里面数据的类型。虽然现在声明了`<Integer>` ，但是由于运行期间类型会被擦除掉，所以Java实际上不知道也不能保证里面的数据类型，所以就不让创建。当然，把泛型去掉后就可以创建了，像下面这样写是可以编译通过的。

```java
List<Integer> [] listArray = new List[2];
```

### 实战

#### 泛型Class对象

例如：在JSON解析字符串时，有时需要从外部传入Class对象，一般写法如下：

```java
public <T> T string2Object(String jsonString, Class<T> clazz) {
  return JSONObject.parseObject(jsonString, clazz);
}
public <T> List<T> string2ObjectList(String jsonString, Class<T> clazz) {
  return JSONObject.parseArray(jsonString, clazz);
}
```

#### 读取真实类型

有时需要在代码中读取泛型的真实类型，一般写法如下：

```java
public abstract class BaseController<T extends BaseEntity, P extends ParamDTO> {
  protected T paramToEntity(P param) {
    String jsonString = JSON.toJSONString(param);
    return (T) JSON.parseObject(jsonString, getGenericType(0));
  }  
  protected Type getGenericType(int index) {
    // 读取泛型参数
    Type superType = this.getClass().getGenericSuperclass();
    if (superType instanceof ParameterizedType) {
      return ((ParameterizedType) superType).getActualTypeArguments()[index];
    } else {
      throw new RuntimeException("Unknown entity class type");
    }
  }
}
```

上面是一个真实的例子，BaseController类声明了两个泛型，T基础实体类基类BaseEntity，P继承实体类基类ParamDTO；paramToEntity()方法将P类实例对象通过JSON转换成T类实例对象。JSON.parseObject()方法需要传入实体类的真实类型T，这里写T或者T.class都是不行的。这个时候就需要通过反射读取真实的T类型。

```java
public class UserController extends BaseController<User, UserDTO> {    
}
```

运行时执行getGenericType(0)返回User.class。getActualTypeArguments()方法返回泛型参数，index是泛型参数的顺序，如果只有一个泛型参数写0就行。



