---
title: 数据库水平拆分
date: 2018-05-03 10:31:58
categories: 数据库
tags: [Database, MySQL]
toc: true
description: 数据库水平拆分。
comments: false
---

### 用户中心

用户中心是典型的1对1例子。

#### 水平分库

实现用户注册、登录、基本信息修改，主键user_id。

业务初期，单库单表就能满足需求。

随着数据量越来越大，需要对数据库进行水平切分，常见的方法有范围法和取模法。

范围法：前提user_id是整数，例如：0~1000万存储到db1，1000万~2000万存储到db2

- 好处：
  - 简单，可以迅速定位数据在哪个db上；
  - 扩容简单，对历史数据无影响；
- 坏处：
  - user_id必须是整数，且自增；

  - 数据量不均匀，前面的db满了才会新增db，新增db数据量小；

  - 访问量不均匀，一般新注册用户活跃度高，新db的负载比旧db负载高

哈希法：

对user_id取模，根据mod值分配db。如果user_id不是整数，可以先计算hashCode()，然后再取模。

- 好处：
  - 简单，计算哈希值就可以确定db位置；
  - 数据均匀，哈希值均匀分布可以保证数据均匀分布到各个db上；
  - 访问量均匀，理由同上；
- 坏处：
  - 扩容复杂，扩容以后可以引起数据迁移，如果平滑的进行数据迁移是一个问题

> 一致性哈希。

#### 问题解决

水平分库后带来的问题：

- 对于user_id的查询可以直接定位到db，但是对于非user_id的查询怎么办呢？



##### 非主键查询

例如，用户表根据user_id进行了分库，现在需要根据user_phone来查询。

前端需求：通过手机号码查询用户。

后台需求：根据渠道、注册时间、性别、地区等属性查询用户，查询条件比较复杂。

前端采用建立非主键属性到主键映射关系的方案，后台采用前后端分离方案。

建一张表，主键是user_phone，内容是user_id，先通过user_phone查询出user_id。这个表不分库。考虑性能，可以把这部分信息直接进redis，放到hash中。如果redis不命中，再查数据库。user_phone和user_id对应关系一旦建立不会改变，可以缓存。

如果需要查询的是user_login_name，而不是user_phone，使用一种算法来根据user_login_name计算user_id，这样就不用查询一次了。f(user_login_name)=user_id。这个前提是数据是我们自定义的。

函数计算比较复杂，也可以不用计算出user_id，只需要知道对应的db是哪个就行。

![](/images/database-partition-user-gene.jpg)

如上图所示，最后三个字节保存的是user_id在那个db上（mod 8），也就是说，新写入一条记录时，先生成user_id，然后通过user_id计算出在哪个db上，然后将这个信息放到user_login_name的最后。当根据用户登录名查询时，截取后三位就知道应该去哪个数据库查询了。

后台

![](/images/database-partition-user-fb.png)

切分库是为了提高访问速度，后台查询条件复杂，对响应时间要求不高，不用分库。从前端通过MQ同步数据即可。MQ同步数据机制建立以后，也可以同步到Hive等其他存储结构中。

### 帖子中心

1对多。





### 订单中心

以order表为例

order_id

user_id

product_id

首先，区分前端访问和后台访问。前端访问指用户的访问，用户下单后需要查询订单，查询条件比较简单，一般只能查询自己的订单，但是响应速度要求要快。后台访问指运营人员对订单的查询，一般查询条件比较复杂，但实时性要求低一些。

![img](/images/database-partition-order-fb.png)

如图所示，order-center提供前端服务，order-center对应的数据库进行了水平切分；order-back提供后台服务，这里显示的存储方案是ES或者Hive，其实也可以是MySQL，总之后台服务的存储数据结构不依赖于前端，可以根据实际业务需求进行选择。如果选择了MySQL，后台存储可以选择不分库，因为后台需求往往逻辑比较复杂，但是对响应性要求不高。

前后端数据同步可以通过开源的中间件来实现，更多的时候像图中一样，通过MQ来同步。MQ异步同步意味着数据是有延迟的。MQ要保证数据送到的可靠性和幂等性。

不只是订单服务，后台可以把所有前端服务产生的数据都汇总到一个库中，可以是原始数据直接进入，也可以是经过order-back处理后的数据进入。（看上去有点像数据仓库了。）

接下来看，前端的数据库存储如何做水平拆分。

这里有一个前提，前端用户查询的都是自己的订单，不能查询别人的订单。这样，我们就可以根据user_id来分库。例如：如果分为4个库，那么最简单的办法对user_id % 4，来确定订单数据落在那个schema上面。实际操作中，我们是通过主键order_id来做水平切分的，所以要保证order_id和user_id的切分一致。所以在生成订单时，可以让order_id的最后两位和user_id 的最后两位一致，或者最简单order_id=timestamp+user_id，这样取模就肯定和user_id一致了。



https://mp.weixin.qq.com/s/PCzRAZa9n4aJwHOX-kAhtA

https://blog.csdn.net/admin1973/article/details/74923283

https://blog.csdn.net/jiangzhexi/article/details/76794801

https://blog.csdn.net/admin1973/article/details/77713716



