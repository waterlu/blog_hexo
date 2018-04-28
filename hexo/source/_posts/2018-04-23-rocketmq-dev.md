---
title: RocketMQ(3) 开发
date: 2018-04-23
updated: 2018-04-24
categories: 消息队列
tags: [mq, rocketmq]
toc: true
description: 本文首先展示最基本的生产者和消费者的用法，接下来详细描述事务消息的发送和回查的写法，最后是和spring boot的集成。
comments: false
---

### 编码

#### 配置

在pom.xml引入rocketmq-client模块和rocketmq-common模块，选择合适的版本，这里我用的是3.1.4版本

https://github.com/apache/rocketmq-externals/tree/master/rocketmq-spring-boot-starter

```xml
<!-- rocket mq -->
<dependency>
	<groupId>com.alibaba.rocketmq</groupId>
	<artifactId>rocketmq-client</artifactId>
	<version>3.1.4</version>
</dependency>
<dependency>
	<groupId>com.alibaba.rocketmq</groupId>
	<artifactId>rocketmq-common</artifactId>
	<version>3.1.4</version>
</dependency>
```

#### 发送消息

- 发送消息比较简单，首先创建消息生产者，指定组；然后启动生产者；
- 当需要发消息时调用生产者的send()方法即可，消息对象需要指定主题、标签、主键、消息体内容等信息；通过send()方法的返回值同步判断是否发送成功；
- 发送失败将会自动重试，重试次数和超时时间可以在创建生产者时进行设置。

```java
package cn.waterlu.test.rocketmq;
  
import com.alibaba.rocketmq.client.producer.DefaultMQProducer;
import com.alibaba.rocketmq.client.producer.SendResult;
import com.alibaba.rocketmq.common.message.Message;

public class TestRocketMq {
    
	@Test
  	public void testProducer() {
      	// 生产者组的名称
      	String groupName = "group_producer";
      	// NameServer地址
      	String nameServer = "10.10.10.163:9876";
      	// 如果发送失败，重试次数
      	int retryTimes = 3;
      	// 发送超时时间（毫秒）
      	long timeout = 10000; 

      	// 创建Producer并进行配置
      	DefaultMQProducer producer = new DefaultMQProducer(groupName);
      	producer.setNamesrvAddr(nameServer);
      	producer.setRetryTimesWhenSendFailed(retryTimes);
      	producer.setSendMsgTimeout(timeout);
      
      	// 启动Producer，可复用
      	producer.start();
      
      	// 创建消息
      	// topic String 消息主题
      	// tag String 消息标签（可空）
      	// key String 消息主键
      	// body byte [] 消息体
      	Message message = new Message(topic, tag, key, body);
      	
      	// 发送消息
      	SendResult sendResult = producer.send(message);
      	SendStatus status = sendResult.getSendStatus();
      	
      	// 判断发送是否成果
        if (status.equals(SendStatus.SEND_OK)) {
        	logger.info("发送成功");
        } else {
        	logger.warn("发送失败");
        }      	
    }
}
```

#### 接收消息

- 消费消息与生产消息类似，需要首先创建消费者，设置参数，最后启动消费者消费消息；
- 消费者和生产者一样需要指定NamerServer地址和消费组名称；
- 消费者启动前需要指定订阅的主题和标签，进行消息过滤；
- 消费者需要注册收到消息后的处理方法；
- 消费者分为Pull和Push两种模式，其本质都是拉去消息；
- Push模式把轮询过程封装了，对用户来说，感觉消息是被推送过来的；
- Pull模式用户需要自己拉起消息。

```java
package cn.waterlu.test.rocketmq;

import com.alibaba.rocketmq.client.consumer.DefaultMQPushConsumer;
import com.alibaba.rocketmq.client.consumer.ConsumeFromWhere;


public class TestRocketMq {
    
	@Test
  	public void testConsumer() {
		// 消费组的名字
		String consumerGroupName = "group_consumer";
		// NameServer地址
		String nameServer = "10.10.10.163:9876";      
      
		// 创建消息者
		DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(consumerGroupName);
		consumer.setNamesrvAddr(nameServer);
      	
		// 三个选项，区别在新订阅组第一次启动时的行为不同，以后都是继续上一次的位置进行消费
		// CONSUME_FROM_LAST_OFFSET 新订阅组第一次启动从队列的最后开始消费
		// CONSUME_FROM_FIRST_OFFSET 新订阅组第一次启动从队列的头开始消费
		// CONSUME_FROM_TIMESTAMP 新订阅组第一次启动从指定时间点开始消费
		consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
      	
		// 指定订阅的主题和标签，主题和标签都是String
		// 多个标签中间通过"||"分隔，例如："pay||order||clear"
		consumer.subscribe(topic, tags);
      	
		// 注册消息处理的回掉方法
		consumer.registerMessageListener(new MessageListenerConcurrently() {
			@Override
			public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, 
					ConsumeConcurrentlyContext consumeConcurrentlyContext) {
				// consumeMessageBatchMaxSize默认值为1，所以List里面只有一个元素
				MessageExt messageExt = list.get(0);
				// 主题
				String topic = messageExt.getTopic();
				// 标签
				String tag = messageExt.getTags();
				// 消息ID，RocketMQ自动生成
				String messageID = messageExt.getMsgId();
				// 消息主键，业务自己指定
				String messageKey = messageExt.getKeys();
				// 消息内容
				byte[] messageBody = messageExt.getBody();
				
				// CONSUME_SUCCESS 表示消息消费成功
				// RECONSUME_LATER 表示消息消费失败，RocketMQ过一段时间后会重新投递消息
				return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
				//return ConsumeConcurrentlyStatus.RECONSUME_LATER;
			}
		}

		// 启动消费组
		consumer.start();
	}
}

```

