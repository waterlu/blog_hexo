---
title: RocketMQ(2) 部署与维护
date: 2018-04-22
categories: 消息队列
tags: [mq, rocketmq]
toc: true
description: RocketMQ由NameServer和BrokerServer组成，NameServer作为注册中心，提供服务发现；BrokerServer提供消息发布、存储和订阅服务。
comments: false
---

由于用到了事务消息功能，所以我使用的是3.1.4版本，这是一个非常老的版本，没有使用最新的版本。RocketMQ从4.0版本开始在Apche孵化，现在最新的版本是4.2.0，预计4.3.0版本可能会增加事务消息功能。3.x的最新稳定版本是3.5.8。

RocketMQ官方版本V3.0.4~3.1.4基于文件系统实现了事务消息，已开源；V3.1.5~4.2.0基于数据库实现事务消息，未开源。

## 部署

[这里](https://github.com/YunaiV/rocketmq-3.1.9/tree/release_3.1.4) 有V3.1.4版本的源码，是作为V3.1.9的一个fork的分支存在的，以下操作都以V3.1.4版本为例，最新版本可能会不一样。

###  打包

- 环境准备：安装rocketmq前需要先安装jdk，git和maven

- 编译打包

```shell
$ cd rocketmq-3.1.4
$ mvn -Dmaven.test.skip=true clean package install assembly:assembly -U
$ cd ./target/alibaba-rocketmq-3.1.4/alibaba-rocketmq
$ ls -l
drwxrwxr-x 2 lu lu  4096 Jan 20 16:34 benchmark
drwxrwxr-x 2 lu lu  4096 Jan 20 16:34 bin
drwxrwxr-x 5 lu lu  4096 Jan 20 16:34 conf
drwxrwxr-x 2 lu lu  4096 Jan 20 16:34 lib
-rw-rw-r-- 1 lu lu 10275 Jan 20 16:34 LICENSE.txt
drwxrwxr-x 2 lu lu  4096 Jan 20 16:34 test
```

### 启动

由于默认启动参数配置的内存比较大，我们一般会先调整一下内存参数，具体为修改apache-rocketmq/bin目录下的runserver.sh和runbroker.sh脚本，修改其中的JAVA_OPT，将Xms/Xmx/Xmn等内存参数调整到合适大小

![图1-RocketMQ部署图](/images/rocketmq_intro_architecture.png)

如上图所示，RocketMQ由NameServer和BrokerServer组成，其中NameServer用做注册中心，Broker启动后注册到NameServer，Producer和Consumer与NameServer通信，获取Broker的地址；Producer发送消息给Broker，Broker完成消息存储，并负责将消息投递给Consumer。

首先，启动NameServer，NameServer的服务端口为9876

```shell
$ cd target/alibaba-rocketmq-3.1.4/alibaba-rocketmq
$  nohup sh bin/mqnamesrv > /dev/null 2>&1 &
```

然后，启动BrokerServer，其中-n参数为NameServer的地址和端口，Broker的服务端口为10911和10912，10911供Producer和Consumer通信使用，10912供Broker集群通信使用

```shell
$ cd target/alibaba-rocketmq-3.1.4/alibaba-rocketmq
$ nohup sh bin/mqbroker -n localhost:9876 > /dev/null 2>&1 &
```

Broker启动时将自己的地址和端口注册到NameServer上，但存在多网卡时，Broker有可能获取不到正确的IP地址，最后导致Producer和Consumer连接不到Broker上，出现类似下面这样的错误。

```shell
connect to <172.17.0.1:10911> failed
```

这种情况下，我们需要在Broker启动时指定IP地址，具体方法为配置一个启动参数文件broker.properties，内容如下（可以从conf/2m-noslave/broker-a.properties复制并修改）：

```properties
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0
brokerIP=10.10.10.100
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
```

启动脚本如下，重点在于-c参数指定了配置文件，当前如果需要我们可以在broker.properties中做更多的个性化配置

```shell
$ nohup sh bin/mqbroker -n localhost:9876 -c conf/broker.properties > /dev/null 2>&1 &
```

### 关闭

关闭Broker服务

```shell
$ sh bin/mqshutdown broker
```

关闭NameServer服务

```shell
$ sh bin/mqshutdown namesrv
```



## 命令管理

RocketMQ提供了bin/mqadmin命令行对消息队列进行管理。

> -n设置NameServer地址和端口(从NameServer上获取Broker地址和端口)

- 查看主题列表
```shell
$ bin/mqadmin topicList -n localhost:9876
TBW102
```

- 创建主题

| 参数   | 说明                              |
| ---- | ------------------------------- |
| -n   | NameServer地址和端口                 |
| -b   | Broker地址和端口(Topic创建在这个Broker上面) |
| -c   | Cluster名称（Broker和Cluster二选一）    |
| -t   | Topic主题名称                       |
| -r   | 读队列数(默认8个)                      |
| -w   | 写队列数(默认8个)                      |

创建主题test


```shell
$ bin/mqadmin updateTopic -n localhost:9876 -b localhost:10911 -t test -r 4 -w 4
create topic to localhost:10911 success.
TopicConfig [topicName=test, readQueueNums=4, writeQueueNums=4, perm=RW-, topicFilterType=SINGLE_TAG]
$ bin/mqadmin topicList -n localhost:9876
TBW102
test
```

- 查看Topic信息

```shell
$ bin/mqadmin topicStats -n localhost:9876 -t test
```

- 查看集群信息

```shell
$ bin/mqadmin clusterList -n localhost:9876
#Cluster Name     #Broker Name	#BID  #Addr                  #Version
DefaultCluster    ubuntu        0     172.17.0.1:10911       V3_1_4
```

- 根据MessageID查询消息

```shell
$ bin/mqadmin queryMsgById -n localhost:9876 -i 
```

- 根据消息Key查询消息

```shell
$ bin/mqadmin queryMsgByKey -n localhost:9876 -t test -k 
```



## 控制台

虽然可以通过命令行查看RocketMQ状态，但是使用起来很不方便。

rocketmq-console项目封装了mqadmin指令，提供了web页面来展示rocketmq信息，所以通常我们都使用rocketmq-console来查看RocketMQ状态。

rocketmq-console属于[rocketmq-externals](https://github.com/apache/rocketmq-externals) 项目的一部分。

- 首先，获取源码

```shell
$ git clone https://github.com/apache/rocketmq-externals
```

- 然后，打包部署(rocketmq-console是spring boot项目)

```shell
$ cd rocketmq-externals/rocketmq-console
$ mvn clean package -Dmaven.test.skip=true
$ java -jar rocketmq-console-ng-1.0.0.jar --server.port=12581 --rocketmq.config.namesrvAddr=10.89.0.64:9876
```



> v3.1.4版本是没有提供控制台工具的，rocketmq-externals(包括rocketmq-console)是为rocketmq v4.0.0以上版本服务的。所以，理论上rocketmq-console和v3.1.4是不兼容的。不过，mqadmin的指令协议没有变化，所以基本使用是可以的。当然，升级到4.0.0以上版本就更没有问题了。
>
> 



