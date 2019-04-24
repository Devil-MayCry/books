---
title: Kafka 持久化原理
date: "2019-04-24"
description: ""
url: /blog/kafka/
image: "/blog/kafka/title.png"
---
Kafka持久化原理介绍
<!--more-->

#### 1. 如何保证宕机的时候数据不丢失
如果要想理解这个acks参数的含义，首先就得搞明白kafka的高可用架构原理。


比如下面的图里就是表明了对于每一个Topic，我们都可以设置他包含几个Partition，每个Partition负责存储这个Topic一部分的数据。


然后Kafka的Broker集群中，每台机器上都存储了一些Partition，也就存放了Topic的一部分数据，这样就实现了Topic的数据分布式存储在一个Broker集群上。