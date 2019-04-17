要想了解Consul的实现原理，就得先理解Consul是用来做什么的

按照Consul的[官方文档](https://www.consul.io/intro/index.html)，它主要提供以下功能：

* 服务注册与发现
* 服务健康检查
* KV存储
* 安全的服务通信（_Secure Service Communication_）
* 多数据中心

根据这个介绍，看上去consul的作用很简单：不就是一个KV形式的分布式数据库嘛。

可问题就出在"分布式"这三个字上。我们都知道，分布式环境会有各种问题存在:

* 通信异常
* 网络分区
* 三态（成功，失败，超时）
* 节点故障

正是由于这些问题，带来了数据的不一致性，也随之带来了解决数据一致性问题的需求。这就发展出了CAP和BASE理论。

而每个分布式系统都需要解决自己的一致性问题，不能每做一个系统都做一套一致性方案吧。这时候，作为中间件形式的一致性服务便出现了，这就迎来了Consul，以及Zookeeper，Etcd等的诞生

说到这儿，就不得不提到大名鼎鼎CAP理论了。CAP告诉我们：

* 一致性\(Consistency\)
* 可用性\(Availability\)
* 分区容错性\(Partition tolerance\)

  一个分布式系统不可能同时满足这三个基本需求，最多只能满足两项  
  同时，在实际运用中，网络问题又是一个必定会出现的异常情况。因此分区容错性成为了一个必然需要面对和解决的问题  
  例如，Consul，Zookeeper满足了CP，而Euerka满足了AP

到这里，我们可以总结一下了:  
**Consul是一个分布式的数据中心，能做到数据一致性的保证。**  
现在你明白为什么Consul可以用来做服务注册和服务发现了吧：**如果没有一个一致性的保证机制，可能会出现一个服务注册后其他服务无法感知，或者发现了一个已经注销的服务的情况**

现在我们可以来理解一下consul的实现原理了

![](/assets/consul1.png)

可以看到Consul可以有多个数据中心，多个数据中心构成Consul集群。
在每个数据中心内，包含可以高达上千个的Consul client，以及3个或5个（官方推荐）的Consul sever，在Consul sever中又选举出一个leader。
因为每个数据中心的节点是一个raft集合。从上边这个图中可以看到，Consul client通过rpc的方式将请求转发到Consul server ，Consul server 再将请求转发到 server leader，server leader处理所有的请求，并将信息同步到其他的server中去。
当一个数据中心的server没有leader的时候，这时候请求会被转发到其他的数据中心的Consul server上，然后再转发到本数据中心的server leader上。

