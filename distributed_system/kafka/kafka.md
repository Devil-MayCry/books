---
title: Kafka概念介绍与原理剖析
date: "2019-04-24"
description: ""
url: /blog/kafka/
image: "/blog/kafka/title.png"
---
Kafka的一些知识汇总
<!--more-->

#### 一、概念
- Producers：
- Consumers：
- topic：
- broker：


{% alert info %}
除了上面几个概念，还有一个也非常重要，那就是 `zookeeper`。
{% endalert %}


1) `Producer` 端使用 `zookeeper` 用来发现 `broker` 列表，以及和 `Topic` 下每个 `partition leader` 建立 `socket` 连接并发送消息。
2) `Broker` 端使用 `zookeeper` 用来注册 `broker` 信息，已经监测 `partition leader` 存活性。
3) `Consumer` 端使用 `zookeeper` 用来注册 `consumer` 信息,其中包括 `consumer` 消费的 `partition` 列表等，同时也用来发现 `broker` 列表，并和 `partition leader` 建立 `socket` 连接，并获取消息。

#### 二、Kafka 数据流
看完「鸡蛋」，「篮子」，我们逐渐清晰了，可这他娘的 `zookeeper` 又出来搅合了，是不是有点懵比了？不要着急，看看下面这张图，也许你就了然于胸了。

![kafka data flow](kafka1.png)

`Kafka` 的总体[数据流](https://www.jianshu.com/p/d3e963ff8b70)是这样的：`Producer` 往 `Broker` 里面的指定 `Topic` 中写消息，`Consumers` 从 `Brokers` 里面拉去指定 `Topic` 的消息，然后进行业务处理。

图中有两个 `topic`，`topic 0` 有两个 `partition`，`topic 1` 有一个 `partition` ，三副本备份。可以看到`consumer gourp 1` 中的 `consumer 2` 没有分到 `partition` 处理，这是有可能出现的。

那么，`Kafka` 又是如何[生产数据](https://www.jianshu.com/p/d3e963ff8b70)的？

![Producer](kafka2.png)

如上图所示：
- 首先创建一条记录，记录中一定要指定对应的 `topic` 和 `value` ，`key` 和 `partition` 可选。 
- 先序列化，然后按照 `topic` 和 `partition` ，放进对应的发送队列中。
- `kafka produce` 都是批量请求，会积攒一批，然后一起发送，不是调 `send()` 就进行立刻进行网络发包。

如果 `partition` 没填，那么情况会是这样的：
1.填写了 `key`：按照 `key` 进行哈希，相同 `key` 去一个 `partition`。（如果扩展了 `partition` 的数量那么就不能保证了）

2.没填 `key`：`round-robin` 来选 `partition`。

{% alert info %}
这些要发往同一个 `partition` 的请求按照配置，攒一波，然后由一个单独的线程一次性发过去。
{% endalert %}
