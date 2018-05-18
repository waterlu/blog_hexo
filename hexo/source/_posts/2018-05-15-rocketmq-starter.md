---
title: SpringBoot整合RocketMQ
date: 2018-05-15 09:13:56
categories: 消息队列
tags: [mq, rocketmq, springboot]
toc: true
description: 虽然RocketMQ已经在2018年提供官方starter实现了和Spring Boot集成，但目前提供的功能还太简单，距离实际应用还有差距。这里我给出自己的开源实现waterloo-starter-rocketmq。
comments: false
---

## 集成RocketMQ

### 基本实现

我们先来回顾一下前面提过的基本用法。

#### 生产者

```java
// 启动Producer，可共用
DefaultMQProducer producer = new DefaultMQProducer(producerGroupName);
producer.setNamesrvAddr(nameServerAddr);
producer.start();

// 发消息，根据返回状态判断发送是否成功
// 默认同步发送，重试次数和超时时间可在创建DefaultMQProducer时配置
Message message = new Message(topic, tag, msgKey, msgBody);
SendResult sendResult = producer.send(message);
```

#### 消费者

```java
// 创建消息者
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(consumerGroupName);
consumer.setNamesrvAddr(nameServerAddr);
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);

// 多个标签中间通过"||"分隔，例如："tag1||tag2||tag3"
// 注意：不可以多次调用subscribe()方法，效果为后者替换前者
// 例如：consumer.subscribe(topic, tag1); consumer.subscribe(topic, tag2); 只订阅了tag2
consumer.subscribe(topic, tags);

// 注册消息回调处理方法
consumer.registerMessageListener(new MessageListenerConcurrently() {
	@Override
	public ConsumeConcurrentlyStatus consumeMessage() {
		return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
	}
}

// 启动
consumer.start();
```

### 官方Starter

RocketMQ在2018年给出了官方集成Spring Boot实现 [rocketmq-spring-boot-starter](https://github.com/rocketmq/rocketmq-spring-boot-starter)

用法如下：

1. 引入pom

```xml
		<dependency>
			<groupId>org.rocketmq.spring.boot</groupId>
			<artifactId>rocketmq-spring-boot-starter</artifactId>
			<version>1.0.0.RELEASE</version>
		</dependency>
```

2. 配置

```properties
spring.rocketmq.name-server=192.168.75.159:9876
```

3. 加注解

```java
@EnableRocketMQ
public class ServiceApplication {
}
```

consumer

```java
@Slf4j
@Service
@RocketMQListener(topic = "topic-1")
public class MyConsumer1 {
     @RocketMQMessage(messageClass = String.class,tag = "tag-1")
     public void onMessage(String message) {
         log.info("received message: {}", message);
     }
}
```

从文档和代码都可以看出，官方starter还处于起步阶段，目前只实现了consumer的消息触发机制，还没有封装producer，配置也非常简单，没有幂等性等处理，所以还是等到它成熟以后再用吧。

### 我的Starter

我做的starter在[这里](https://github.com/waterlu/waterloo-starter-rocketmq) ，目前比官方的功能多一些，也提供Test例子代码。

Github上readme写的挺详细的，这里就不再复述了，欢迎使用、打star、提issue。

下面说一下主要思路和重点。

#### Producer

先说生产者，生产者相对简单一些。思路就是创建一个单例的DefaultMQProducer，在系统启动时start()，以后@Autowired注入以后就可以使用了，非常简单。剩下的就是配置DefaultMQProducer参数的问题了。

DefaultMQProducer本身是支持异步消息发送机制的，这里我只使用了同步发送消息机制，默认重试3次。

#### Consumer

消费者要比生产者复杂，一个AbstractMQProducer对应一个DefaultMQProducer，但是因为Tag的存在，可能多个AbstractMQConsumer对应一个DefaultMQPushConsumer，如果看代码，这里是比较绕的地方。

首先，官方提供了DefaultMQPushConsumer和DefaultMQPullConsumer。虽然从名字上看，应该一个是push，另外一个是pull，但实际上都是pull。我选择的是DefaultMQPushConsumer，用上去更像push。

细心的人可能注意到，消息回调方法consumeMessage()的参数是`List<MessageExt> list` ，但我们是一个一个消息处理的，这样没有问题吗？现在没有问题是因为默认`consumer.setConsumeMessageBatchMaxSize(1);` 也就是说这个list的默认长度是1，只有1个元素，如果我们调整了consumeMessageBatchMaxSize参数，那么实现逻辑就需要修改了。

还有，ConsumeFromWhere也是非常重要的一个参数，我现在固定使用CONSUME_FROM_MAX_OFFSET，也就是从最新数据开始消费；实际上还支持CONSUME_FROM_FIRST_OFFSET和CONSUME_FROM_TIMESTAMP，也就是从头消费或者从固定时间点开始消费，现在没有实现。

最后就是扫描注解是如何实现的，其实就一行代码

```java
Map<String, Object> beans = applicationContext.getBeansWithAnnotation(RocketMQConsumer.class);
```



