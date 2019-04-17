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

首先我们来了解一下关键术语：

* Agent——agent是一直运行在Consul集群中每个成员上的守护进程。通过运行 consul agent 来启动。agent可以运行在client或者server模式。指定节点作为client或者server是非常简单的，除非有其他agent实例。所有的agent都能运行DNS或者HTTP接口，并负责运行时检查和保持服务同步。
* Client——一个Client是一个转发所有RPC到server的代理。这个client是相对无状态的。client唯一执行的后台活动是加入LAN gossip池。这有一个最低的资源开销并且仅消耗少量的网络带宽。
* Server——一个server是一个有一组扩展功能的代理，这些功能包括参与Raft选举，维护集群状态，响应RPC查询，与其他数据中心交互WAN gossip和转发查询给leader或者远程数据中心。
* DataCenter——虽然数据中心的定义是显而易见的，但是有一些细微的细节必须考虑。例如，在EC2中，多个可用区域被认为组成一个数据中心？我们定义数据中心为一个私有的，低延迟和高带宽的一个网络环境。这不包括访问公共网络，但是对于我们而言，同一个EC2中的多个可用区域可以被认为是一个数据中心的一部分。
* Consensus——在我们的文档中，我们使用Consensus来表明就leader选举和事务的顺序达成一致。由于这些事务都被应用到有限状态机上，Consensus暗示复制状态机的一致性。
* Gossip——Consul建立在Serf的基础之上，它提供了一个用于多播目的的完整的gossip协议。Serf提供成员关系，故障检测和事件广播。更多的信息在gossip文档中描述。这足以知道gossip使用基于UDP的随机的点到点通信。
* RPC——远程过程调用。这是一个允许client请求server的请求/响应机制。

* LAN Gossip——它包含所有位于同一个局域网或者数据中心的所有节点。

* WAN Gossip——它只包含Server。这些server主要分布在不同的数据中心并且通常通过因特网或者广域网通信。

可以看到Consul可以有多个数据中心，多个数据中心构成Consul集群。数据中心间通过WAN GOSSIP在Internet上交互报文。也就是说，Consul多个数据中心之间基于WAN来做同步

  
在每个数据中心内，包含可以高达上千个的Consul client，以及3个或5个（官方推荐）的Consul sever，在Consul sever中又选举出一个leader。server和client之间，还有一条LAN GOSSIP通信，这是用于当LAN内部发生了拓扑变化时，存活的节点们能够及时感知，比如server节点down掉后，client就会触发将对应server节点从可用列表中剥离出去

当然，server与server之间，client与client之间，client与server之间，在同一个datacenter中的所有consul agent会组成一个LAN网络（当然它们之间也可以按照区域划分segment），当LAN网中有任何角色变动，或者有用户自定义的event产生的时候，其他节点就会感知到，并触发对应的预置操作。

所有的server节点共同组成了一个集群，他们之间运行raft协议，通过共识仲裁选举出leader。Consul client通过rpc的方式将请求转发到Consul server ，Consul server 再将请求转发到 server leader，server leader处理所有的请求，并将信息同步到其他的server中去。所有的业务数据都通过leader写入到集群中做持久化，当有半数以上的节点存储了该数据后，server集群才会返回ACK，从而保障了数据的强一致性。当然，server数量大了之后，也会影响写数据的效率。所有的follower会跟随leader的脚步，保障其有最新的数据副本。

当一个数据中心的server没有leader的时候，请求会被转发到其他的数据中心的Consul server上，然后再转发到本数据中心的server leader上

### Consul-server 中 Leader的选举

Consul中只有server节点会参与Raft算法并且作为peer set中的一部分。所有的client节点会转发请求道server节点。这个设计的部分原因是，随着更多的成员被添加到对等集，quorum的大小也会增加。这将带来性能问题，因为您可能需要等待数百台机器对条目达成一致，而不是等待少数机器。

当启动时，单个Consul server的模式为bootstrap。这个模式允许节点选举自身作为leader。一旦leader选举成功，其他服务器可以以保持一致性和安全性的方式添加到对等集。最终，一旦初始的部分server被加入，bootstrap模式就会被disable。

由于所有的server都处于对等集合中，它们都知道当前的leader是谁。当RPC到到一个非leader的server上时，请求会被转发到leader。如果PRC是一个查询类型，即只读的请求，leader会根据FSM的当前状态生成结果。如果RPC是一个转换类型，即它修改了状态，leader会生成一个新的日志条目并使用Raft算法对它进行应用。一旦日志条目提交并且应用于FSM了，转换请求就完成了。

由于Raft的备份特性，它的性能对网络延迟是很敏感的。出于这个原因，每一个datacenter会选举独立的leader并且维护自己的不相交的对等集合。数据被datacenter进行分区，所以每个leader只负责处理它自己datacenter的数据。当请求被一个远程datacenter接收时，会被转发到正确的leader。这种设计允许低延迟事务和高可用性，而不牺牲一致性。



