---
title: 从短地址说起
date: 2018-05-07 08:44:41
categories: WEB
tags: [WEB, 短地址, 哈希, hashCode]
toc: true
description: 短地址是实际业务中常见的需求，通过转换缩短URL链接地址的长度。小型业务中，我们通常使用免费的第三方服务来实现短地址转换。如果需要自己实现，该怎么做呢，本文给出方案和思考。此外，最后还引申了页面跳转和进制转换问题。
comments: true
---

## 短地址

短地址（也叫Short URL）就是为了让一个很长的网站链接缩短为一个短的链接，短地址最初的出现是因为微博内有字数限制，现在短地址的应用领域越来越广泛。

小型业务中，我们通常使用免费的第三方服务来实现短地址转换。如果需要自己实现，该怎么做呢，本文给出方案和思考。单纯就短地址服务来说，的确不需要自己来实现，但仔细思考一下其实现原理和细节，对于我们理解哈希、分片等概念都有帮助。

### 需求

业务上有发营销短信的需求，短信内容中带有APP下载地址。由于短信字数有限制（100个左右），很可能一个链接地址长度就超过了总字数限制，所以要使用短地址。

例如：下面为客户端1.5.1安卓版本的下载地址，仅链接地址就超过了100个字符。

http://imtt.dd.qq.com/16891/EB45FB72BADD5F2C0EBF02C906AAFCD6.apk?fsname=com.zjinv.kingold_1.5.1_15.apk&csr=1bbd

> 进一步，还可以根据短地址生成二维码。

### 托管实现

实际工作中，我们可以通过短地址服务平台来实现。这些平台提供了将长地址转换为短地址的功能。

- 新浪: http://sina.lt/
  - sina.lt/t.cn/
- 百度: http://dwz.cn/
  - dwz.cn

例如：通过新浪提供的服务，我们把一篇博客的长地址转换为短地址，然后使用即可，新浪帮我们完成页面跳转。

 - 长地址：https://my.oschina.net/didispace/blog/1807876
 - 短地址：http://t.cn/Rukpf81

### 自己实现

以上是通过第三方服务实现短链接，下面来看看如何自己实现。

#### 哈希

首先，容易想到调用String类的hashCode()方法，将一个字符串转换为一个int值，然后再将10进制的int值转换为16进制字符串就实现了短地址。下面为例子代码：

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

我们已经实现了长地址到短地址的映射（当然，还需要存储起来供查询）。但是，这种做法是有问题的。因为有可能产生冲突，也就是说两个不一样的url地址，计算出来的hash值一样，导致它们的短地址一样。

我们把两个对象生成的hash值一样的情况成为哈希冲突，哈希冲突需要做专门的处理，请参考HashMap的做法。

##### hashCode()

下面我们先来看看hashCode()方法是如何实现的。

通常意义上，hashCode()是Object类的方法，返回的是对象在JVM堆上的内存地址，不同对象的堆内存地址不同，hash值就不同。也就是说，在一个JVM内我们能够保证不同对象的hashCode()不同，但是不同JVM之间就不能保证了。

```java
public class Object {
  public native int hashCode();  
}
```

但是，String类重载了Object类的hashCode()方法，没有使用内存地址，自定义了自己的实现，代码如下：

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

算法如下，从第一个字符开始，第i字符转换为int，然后乘以31的n-i次方，然后加和，计算哈希值。

```java
hashCode=s[0]*31^(n-1)+s[1]*31(n-2)+s[2]*31^(n-3)+......+31*s[n-1]+s[n]
```

ASCII码表如下，这里可以查看到每个字符对应的int值

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

这种算法有可能出现两个字符串的哈希值一样的情况，例如："hierarch"和"crinolines"的哈希值一样，"buzzards" 和 "righto"和哈希值也相同。

##### 为什么是31

为什么乘以31的n次方，不是2，不是32，不是101呢？

- 31是一个不大不小的质数，是作为 hashCode 乘子的优选质数之一。另外一些相近的质数，比如37、41、43等等，也都是不错的选择。选择质数作为因子是数学理论，可以保证：
  - 冲突率尽量低；
  - 分布率尽量广，计算出来的hash值分散广泛，自然冲突就少，本质和冲突率低是一样的。
- 31可以被 JVM 优化，方便计算，`31 * i = (i << 5) - i`。

> 一定不能选取2的整数幂，例如：2，4，16，32，64等，因为乘以2的整数幂，相当于左移；左移时丢弃高位，地位补零，相当于丢失了信息，特别容易发生冲突。

#### 自增

既然哈希可能有冲突，不能使用，那么我们来看看另外一种思路，使用数据库自增ID的方法。

建一张表，主键是bigint自增ID，内容是长地址即可。短地址只需要将10进制bigint转换成62进制即可。

> 为什么62进制？A-Z(26)+a-z(26)+0-9(10)=62。

每收到一个地址转换请求，就往数据库表里面增加一条记录，建立对应关系。访问时将62进制短地址转换为ID就可以根据主键快速查询到长地址。

使用这个方法，每收到一个请求就往数据库中添加一条新的记录，长地址是可以重复的。多个短地址都指向同一个长地址，操作不会出错，但是占用的多余的存款空间。

##### 改进

下面看看如何改进，保证每个长地址对应唯一的短地址：

方案一：首先，最容易想到：每次插入前先查询一次数据库表，如果长地址存在，直接返回主键，如果长地址不存再添加；当数据库量大时，这么操作效率太低，首先排除。

方案二：再进一步，查询数据库太慢，我们可以把长地址和短地址的对应关系冗余存储到redis中，插入数据库前先查询redis；这种方法减小了数据库压力，但是redis中使用hash表存储所有对应关系，占用空间也很大；

方案三：继续优化，既然redis的问题是占用空间太大，那么我们是否可以不保存所有的对应关系，只保留最近的对应关系；使用LRU算法可以在保证一定命中率的前提下减少存储空间占用。（可以这么做的前提是我们认为有些地址热度较高，经常会被引用到）。

前面都只考虑了单机的情况，如果并发量巨大，一台数据库服务器处理不过来，那么就需要继续优化了：

方案一：数据库分片，例如：分4个db，每个起始值+1（1，2，3，4），步长4。

方案二：使用snowflake等分布式ID生成器生成ID。

##### 重定向

最后，还有一个页面重定向的问题，我们来看看操作过程：

- 用户访问短地址`http://t.cn/Rukpf81` 
- t.cn服务器收到请求，将`Rukpf81` 转换为主键`3038198013989` 查询到长地址`https://my.oschina.net/didispace/blog/1807876` 
- t.cn服务器返回HTTP 302，在HTTP的Location中设置长地址
- 浏览器重定向请求到长地址

注意：HTTP 301和302重定向的区别：

- 301永久重定向，SEO用新页面替换旧页面
- 302临时重定向，SEO认为是新旧是两个页面

### 进制转换

62进制转10进制的源码：

- 也可以设置因子factor=1，从最后一位开始计算，每次factor = factor*62；
- 代码算法参考了String类的hashCode()实现，因为每次都有value * 62，所以第一位实际上做了n-1次* 62操作，与乘以62的n-1次方结果是一样的。

```java
public final char[] array = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
		.toCharArray();
private long _62_to_10(String code) {
  long value = 0;
  for (int i=0; i<code.length(); i++) {
    value = value * 62 + _62_2_10(code.charAt(i));
  }
  return value;
}
private int _62_2_10(char c) {
  for (int i=0; i<array.length; i++) {
    if (c == array[i]) {
      return i;
    }
  }
  return 0;
}
```

计算结果

```shell
Rukpf81=3038198013989
```

10进制转62进制的源码：

- 每一次计算出来的余数是当前最低位的值，商用于下一次计算；
- 最后做字符串反转。

```java
static String _10_to_62(long value) {
  StringBuffer buffer = new StringBuffer();
  while(value > 0) {
    int v = (int)(value % 62);
    buffer.append(array[v]);
    value = value / 62;
  }
  return buffer.reverse().toString();
}
```

> 注意：`int v = (int)(value % 62);` 不能写成`int v = (int)value % 62;` ，精度截断后计算结果就错了。

