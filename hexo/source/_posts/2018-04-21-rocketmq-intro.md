---
title: RocketMQ(1) 介绍
date: 2018-04-21 14:00:00
updated: 2018-05-15 09:17:00
categories: 消息队列
tags: [mq, rocketmq]
toc: true
description: RocketMQ是阿里巴巴开源的一款高性能、高吞吐率的分布式消息中间件。产品基于高可用分布式集群技术，提供消息发布订阅、消息轨迹查询、定时（延时）消息、资源统计、监控报警等功能，是阿里巴巴双11使用的核心产品。2016年阿里巴巴正式宣布将RocketMQ捐赠给Apache软件基金会。
comments: true
---

## 简介
RocketMQ是阿里巴巴开源的一款高性能、高吞吐率的分布式消息中间件。产品基于高可用分布式集群技术，提供消息发布订阅、消息轨迹查询、定时（延时）消息、资源统计、监控报警等功能，是阿里巴巴双11使用的核心产品。2016年阿里巴巴正式宣布将 RocketMQ 捐赠给 Apache 软件基金会。

RocketMQ的前身叫MetaQ，MetaQ从3.0版本开始更名为RocketMQ。

官网地址http://rocketmq.apache.org/， 目前最新版本为4.2.0。

## 基本概念

以下内容来自阿里云对MQ项目的介绍。

### 应用场景

* 异步解耦，基于发布订阅模型，对分布式应用进行异步解耦，增加应用的水平扩展能力；
* 削峰填谷，大促等流量洪流突然来袭时，MQ 可以缓冲突发流量，避免下游订阅系统因突发流量崩溃；
* 日志监控，作为重要日志的监控通信管道，将应用日志监控对系统性能影响降到最低；
* 消息推送，为社交应用和物联网应用提供点对点推送，一对多广播式推送的能力；
* 金融报文，发送金融报文，实现金融准实时的报文传输，可靠安全；
* 电信信令，将电信信令封装成消息，传递到各个控制终端，实现准实时控制和信息传递。

> 上面是官方介绍，我来根据自己的理解解读一下：
>
> 异步解耦：对于实时性要求不高的操作，例如：用户注册送金币，将送金币的逻辑从用户注册的主逻辑中拆分处理，通过消息触发，这就是异步解耦。异步解耦可以提升主逻辑的响应速度，辅助逻辑稍有延迟。
>
> 削峰填谷：相当于排队，例如：突发大量用户抢购行为，不直接处理认购操作，而是将请求发送到MQ队列，再通过消费者消费掉。相当于把并发操作通过队列变成了串行操作，当然这里的串行不是一个一个执行，如果有M个消费者，类似于M个一起执行。或者说，就像银行柜员的操作，不管来多少人办理业务，就开3个窗口，大家拿号依次办理；不用队列就相当于人多了就得多雇柜员（峰），人少了就把柜员裁了（谷）。
>
> 日志监控：我理解就是ELK，一般用Kafka更多。
>
> 消息推送：我没用过，我感觉这个应该不实用，因为RocketMQ的consumer无论push还是pull模式都是主动拉去的。RabbitMQ做消息推送更合适，它提供了很多插件，比较方便。
>
> 金融报文和电信信令：没用过，不太理解，类似于事件驱动？

### 消息类型

#### 定时消息和延时消息

* 定时消息：Producer 将消息发送到 MQ 服务端，但并不期望这条消息立马投递，而是推迟到在当前时间点之后的某一个时间投递到 Consumer 进行消费。
* 延时消息：Producer 将消息发送到 MQ 服务端，但并不期望这条消息立马投递，而是延迟一定时间后才投递到 Consumer 进行消费。

定时/延时消息适用于如下一些场景：

* 消息生产和消费有时间窗口要求：比如在电商交易中超时未支付关闭订单的场景，在订单创建时会发送一条 MQ 延时消息，这条消息将会在30分钟以后投递给消费者，消费者收到此消息后需要判断对应的订单是否已完成支付。如支付未完成，则关闭订单，如已完成支付则忽略。
* 通过消息触发一些定时任务：比如在某一固定时间点向用户发送提醒消息。

> 我们在提现操作中使用了延时消息，因为提现操作不能立即返回真正是否成功，需要后续主动查询提现结果。我们在提现请求成功后，过一段时间开始启动查询操作，用的就是延时消息。

#### 顺序消息

顺序消息是MQ提供的一种按照顺序进行发布和消费的消息类型。顺序消息由两个部分组成：顺序发布和顺序消费。顺序消息类型分为两种：全局顺序和分区顺序。

- 全局顺序消息

MQ全局顺序消息适用于以下场景：性能要求不高，所有的消息严格按照 FIFO 原则进行消息发布和消费的场景。

![图1-全局顺序消息](/images/rocketmq_intro_sequence_message_global.png)

- 分区顺序消息

MQ 分区顺序消息适用于如下场景：
性能要求高，以 sharding key 作为分区字段，在同一个区块中严格的按照 FIFO 原则进行消息发布和消费的场景。

举例说明：
【例一】用户注册需要发送发验证码，以用户 ID 作为 sharding key， 那么同一个用户发送的消息都会按照先后顺序来发布和订阅。
【例二】电商的订单创建，以订单 ID 作为 sharding key，那么同一个订单相关的创建订单消息、订单支付消息、订单退款消息、订单物流消息都会按照先后顺序来发布和订阅。

阿里巴巴集团内部电商系统均使用此种分区顺序消息，既保证业务的顺序，同时又能保证业务的高性能。

![图2-分区顺序消息](/images/rocketmq_intro_sequence_message_section.png)

#### 事务消息

- 事务消息：MQ 提供类似XA的分布事务功能，通过 MQ 事务消息能达到分布式事务的最终一致；
- 半消息：暂不能投递的消息，发送方已经将消息成功发送到了 MQ 服务端，但是服务端未收到生产者对该消息的二次确认，此时该消息被标记成“暂不能投递”状态，处于该种状态下的消息即半消息；
- 消息回查：由于网络闪断、生产者应用重启等原因，导致某条事务消息的二次确认丢失，MQ 服务端通过扫描发现某条消息长期处于“半消息”时，需要主动向消息生产者询问该消息的最终状态（Commit 或是 Rollback），该过程即消息回查。

![图3-事务消息](/images/rocketmq_intro_transaction_message.png)

如上图所示，事务消息的处理过程如下：

- 发送方向 MQ 服务端发送消息；
- MQ Server 将消息持久化成功之后，向发送方 ACK 确认消息已经发送成功，此时消息为半消息；
- 发送方开始执行本地事务逻辑；
- 发送方根据本地事务执行结果向 MQ Server 提交二次确认（Commit 或是 Rollback），MQ Server 收到 Commit 状态则将半消息标记为可投递，订阅方最终将收到该消息；MQ Server 收到 Rollback 状态则删除半消息，订阅方将不会接受该消息；
- 在断网或者是应用重启的特殊情况下，上述步骤4提交的二次确认最终未到达 MQ Server，经过固定时间后 MQ Server 将对该消息发起消息回查；
- 发送方收到消息回查后，需要检查对应消息的本地事务执行的最终结果；
- 发送方根据检查得到的本地事务的最终状态再次提交二次确认，MQ Server 仍按照步骤4对半消息进行操作。

事务消息完成本地事务后，可在返回如下三种状态：

- TransactionStatus.CommitTransaction 提交事务，允许订阅方消费该消息；
- TransactionStatus.RollbackTransaction 回滚事务，消息将被丢弃不允许消费；
- TransactionStatus.Unknow 暂时无法判断状态，期待固定时间以后 MQ Server 向发送方进行消息回查。

> 事务消息是我们选择RocketMQ作为消息队列的最重要原因，如果没有事务消息Kafka用的更多。

### 集群消费和广播消费

#### 集群消费

- 消息只被消费者组中的一个实例消费；

- 这是我们最常见的模式，集群消费可以很方便的横向拓展，提升处理能力。

  ![图4-集群消费](/images/rocketmq_intro_message_consume_group.png)

#### 广播消费

- 消费者组中的每一个实例都可以消费到消息。

![图5-广播消费](/images/rocketmq_intro_message_consume_broadcast.png)

### 消息过滤

Tag，即消息标签、消息类型，用来区分某个 MQ 的 Topic 下的消息分类。MQ 允许消费者按照 Tag 对消息进行过滤，确保消费者最终只消费到他关心的消息类型。

以下图电商交易场景为例，从客户下单到收到商品这一过程会生产一系列消息，比如订单创建消息（order）、支付消息（pay）、物流消息（logistics）。这些消息会发送到 Topic 为 Trade_Topic 的队列中，被各个不同的系统所接收，比如支付系统、物流系统、交易成功率分析系统、实时计算系统等。其中，物流系统只需接收物流类型的消息（logistics），而实时计算系统需要接收所有和交易相关（order、pay、logistics）的消息。

![图6-消息标签](/images/rocketmq_intro_message_tag.png)

说明：针对消息归类，您可以选择创建多个 Topic， 或者在同一个 Topic 下创建多个 Tag。但通常情况下，不同的 Topic 之间的消息没有必然的联系，而 Tag 则用来区分同一个 Topic 下相互关联的消息，比如全集和子集的关系，流程先后的关系。

### 消息重试

- 当消息不能送达时，MQ 默认允许每条消息最多重试 16 次，每次重试的间隔越来越长，从10 秒、30秒、1分钟，直到2小时；
- 如果消息重试 16 次后仍然失败，消息将不再投递；
- 也就是说，某条消息在一直消费失败的前提下，将会在接下来的 4 小时 46 分钟之内进行 16 次重试，超过这个时间范围消息将不再重试投递。

### 消费幂等

*发送时消息重复（消息 Message ID 不同）*

MQ Producer 发送消息场景下，消息已成功发送到服务端并完成持久化，此时网络闪断或者客户端宕机导致服务端应答给客户端失败。如果此时 MQ Producer 意识到消息发送失败并尝试再次发送消息，MQ 消费者后续会收到两条内容相同但是 Message ID 不同的消息。


*投递时消息重复（消息 Message ID 相同）*

MQ Consumer 消费消息场景下，消息已投递到消费者并完成业务处理，当客户端给服务端反馈应答的时候网络闪断。为了保证消息至少被消费一次，MQ 服务端将在网络恢复后再次尝试投递之前已被处理过的消息，MQ 消费者后续会收到两条内容相同并且 Message ID 也相同的消息。

真正安全的幂等处理，不建议以 Message ID 作为处理依据。最好的方式是以业务唯一标识作为幂等处理的关键依据，而业务的唯一标识可以通过消息 Key 进行设置。

> RocketMQ没有提供消息幂等实现，需要自己做。即使提供了，安全起见，我认为自己也要做幂等判断，尤其是敏感型消息。

### 消息发送方式

- 可靠同步发送，同步发送是指消息发送方发出数据后，会在收到接收方发回响应之后才发下一个数据包的通讯方式；
- 可靠异步发送，异步发送是指发送方发出数据后，不等接收方发回响应，接着发送下个数据包的通讯方式。MQ 的异步发送，需要用户实现异步发送回调接口（SendCallback），在执行消息的异步发送时，应用不需要等待服务器响应即可直接返回，通过回调接口接收服务器响应，并对服务器的响应结果进行处理；
- 单向（Oneway）发送，单向（Oneway）发送特点为只负责发送消息，不等待服务器回应且没有回调函数触发，即只发送请求不等待应答。此方式发送消息的过程耗时非常短，一般在微秒级别。


