---
title: Web短地址生成
date: 2018-05-07 08:44:41
categories: WEB
tags: [WEB, 短地址, 哈希, hashCode]
toc: true
description:
comments: false
---

## 短地址

### 需求

业务上有发营销短信的需求，短信内容中带有APP下载地址。由于短信字数有限制（100个左右），很可能一个链接地址长度就超过了总字数限制，所以要使用短地址。

例如：下面为客户端1.5.1安卓版本的下载地址，链接地址就超过了100个字符。

http://imtt.dd.qq.com/16891/EB45FB72BADD5F2C0EBF02C906AAFCD6.apk?fsname=com.zjinv.kingold_1.5.1_15.apk&csr=1bbd

> 进一步，还可以根据短地址生成二维码。

### 托管实现

实际工作中，我们可以通过短地址服务平台来实现。这些平台提供了将长地址转换为短地址的功能。

- 新浪: http://sina.lt/
  - sina.lt/t.cn/
- 百度: http://dwz.cn/
  - dwz.cn

例如：通过新浪，我们把一篇博客的长地址转换为短地址。

 - 长地址：https://my.oschina.net/didispace/blog/1807876
 - 短地址：http://t.cn/Rukpf81

### 自己实现

以上是通过第三方托管实现短链接，下面来看看如何自己实现。

#### 哈希

调用String类的hashCode()方法，可以将一个字符串转换为一个int值。然后再将10进制的int值转换为16进制字符串就实现了地址缩短。

```java
public class HashExample {
  public static void main(String[] args) {
    String url = "https://my.oschina.net/didispace/blog/1807876";
    int hashCode = url.hashCode();
    String code = String.format("%x", hashCode);
    System.out.println("http://t.cn/" + code);
  }
}
```

输出如下

```shell
http://t.cn/70fcf5cf
```

我们已经实现了长地址到短地址的映射，但是，这种做法是有问题的。因为有可能产生冲突，也就是说两个不一样的url地址，计算出来的hash值一样，导致它们的短地址一样。

##### hashCode()

下面我们来仔细分析一下hashCode()方法：

hashCode()是Object类的方法，返回的是对象在JVM堆上的内存地址，不同对象的堆内存地址不同，hash值就不同。也就是说，在一个JVM内我们能够保证不同对象的hashCode()不同，但是不同JVM之间就不能保证了。

```java
public class Object {
  public native int hashCode();  
}
```

String类重载了Object类的hashCode()方法，代码如下：

```java
public final class String{    
  public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
      char val[] = value;
      for (int i = 0; i < value.length; i++) {
        h = 31 * h + val[i];
      }
      hash = h;
    }
    return h;
  }
}
```

算法如下，从第一个字符开始，乘以31的n-1次方，然后加和，计算哈希值。

```java
hashCode=s[0]*31^(n-1)+s[1]*31(n-2)+s[2]*31^(n-3)+......+31*s[n-1]+s[n]
```

ASCII码表如下

| 字符   | 10进制 | 16进制 | 字符   | 10进制 | 16进制 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| A    | 65   | 0x41 | a    | 97   | 0x61 |
| B    | 66   | 0x42 | b    | 98   | 0x62 |
| Z    | 90   | 0x5A | z    | 122  | 0x7A |
| 0    | 48   | 0x30 | .    | 46   | 0x2E |
| 1    | 49   | 0x31 | /    | 47   | 0x2F |
| 9    | 57   | 0x39 | :    | 58   | 0x3A |

字符串"ab"的hashCode

```java
97*31+98=3105
```

字符串"ABC"的hashCode

```java
65*31*31+66*31+67=64578
```

整数的范围是[-2,147,483,648, 2,147,483,647] = [负2的31次方，2的31次方-1]。31的7次方就超过了整数的最大值。这种算法有可能出现两个字符串的哈希值一样的情况，例如："hierarch"和"crinolines"的哈希值都是-1732884796。"buzzards" 和 "righto"和哈希值也相同。

##### 为什么是31

- 31是一个不大不小的质数，是作为 hashCode 乘子的优选质数之一。另外一些相近的质数，比如37、41、43等等，也都是不错的选择。
  - 冲突率低
  - 分布率广
- 31可以被 JVM 优化，方便计算，`31 * i = (i << 5) - i`。

> 一定不能选取2的整数幂，例如：2，4，16，32，64等，因为乘以2的整数幂，相当于左移；左移时丢弃高位，地位补零，相当于丢失了信息，容易发生冲突。

#### 自增

既然哈希可能有冲突，不能使用，那么我们来看看另外一种思路，使用数据库自增ID的方法。

建一张表，主键是bigint自增ID，内容是长地址即可。短地址只需要将10进制bigint转换成62进制即可。

> 为什么62进制？A-Z(26)+a-z(26)+0-9(10)=62。

每收到一个地址转换请求，就往数据库表里面增加一条记录，建立对应关系。访问时将62进制短地址转换为ID就可以查询到长地址。

##### 改进

如果不考虑空间浪费，不同的短地址可以对应相同的长地址。下面看看改进方案。

方案一：每次查询一次数据库表，如果长地址存在，直接返回，如果长地址不存在，添加；效率太低。

方案二：查询数据库太慢，把长地址和短地址的对应关系放到redis中，占用空间太多。

方案三：还是用redis缓存，但是不缓存所有，根据时间LRU。

并发的问题：可以分片，例如：4个db，每个起始值+1，步长4。

##### 重定向

操作过程

- 用户访问短地址http://t.cn/Rukpf81
- t.cn服务器收到请求，查询到长地址https://my.oschina.net/didispace/blog/1807876
- t.cn服务器返回HTTP 302，在HTTP的location中设置长地址
- 浏览器重定向请求到长地址

注意：HTTP 301和302重定向的区别：

- 301永久重定向，SEO用新页面替换旧页面
- 302临时重定向，SEO认为是新旧是两个页面

##### ID发号器

ID发号器就是分布式ID生成器（snowflake）。

自增ID从1开始，很容易看到规律，生成的短地址非常短，高并发时需要数据库分片。使用ID发号器，在应用端生成有增长趋势的分布式ID也可以。