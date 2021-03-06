---
title: 《从Paxos到zookeeper分布式一致性原理与实践》学习笔记
date: "2019-03-30"
description: ""
image: "/blog/paxos_to_zookeeper/bitcoin.jpeg"
---
 阅读《从Paxos到zookeeper》的一些笔记记录.
 
 虽然主要是讲Zookeeper使用的，但前几章对分布式体系和历史的介绍很赞
<!--more-->

# 第一章：分布式架构

* ACID:事务需要满足的原子性，一致性，隔离性，持久性
* 分布式事务的CAP和BASE理论
* CAP:  一致性，可用性，分区容错性，只能同时满足两个。但针对分布式系统，P是必须满足的，所以实在C和A之间寻找平衡
* BASE:是基本可用，软状态，最终一致性的简写。

# 

# 第二章: 一致性协议

* 2PC（两阶段提交）和3PC（三阶段提交），Paxos:都是为了解决分布式系统一致性的问题
* 当一个事务操作需要跨越多个分布式节点的时候，为了保持事物的ACID特性，就需要引入一个称为协调者（coordinator）的组件来统一调度所有的分布式节点的执行逻辑。这些被调度的分布式节点则被称为参与者（Participant）

### 2PC：

阶段一：提交事务请求

1. 事务询问
2. 执行事务，并将uodo和redo信息记入事务日志中
3. 各参与者向协调者反馈1的相应

阶段二：执行事务提交

如果所有参与者都是yes

1. 发送提交请求
2. 参与者执行提交
3. 反馈提交结果
4. 完成事务

如果一个参与者反馈no，或者等待超时无反馈，进行中断

1. 发送回滚请求
2. 事务回滚
3. 反馈事务回滚结果
4. 中断事务

2PC优点：

1. 强一致性
2. 原理简单，实现方便

2PC缺点：

1. 同步阻塞
2. 协调者出现问题，整个系统停滞
3. 异常时会有数据不一致
4. 总的来说，没有容错机制，任意一个节点失败都会导致整个事务失败

### 3PC：

三阶段提交将”提交事务请求“过程一分为二，形成了由CanCommit,PreCommit, doCommit三个阶段组成的事务处理协议

阶段一：CanCommit

1. 事务询问
2. 各参与者向协调者反馈事务询问的相应

阶段二：PreCommit

当所有都为yes

1. 发送预提交请求（PreCommit）,进入Prepared
2. 参与者接收到预提交后，执行事务操作，并将uodo和redo记录到日志
3. 参与者向协调者反馈执行相应ack，等待最终指令

当没有收到所有yes时

1. 协调者向所有参与者发送abort
2. 中断事务

阶段三：doCommit

两种情况

执行提交：

1. 如果协调者接收到所有参与者的ack，它将从”预提交“转换到”提交“，向所有的参与者发送doCommit
2. 事务提交
3. 参与者完成提交后，向协调者发送ack
4. 协调者收到所有ack后，完成事务

中断事务：

1. 协调者向所有参与者发送abort请求
2. 参与者利用undo信息执行回滚操作
3. 反馈事务回滚结果，向协调者发送ack
4. 中断事务

在阶段三，如果协调者出现问题或者协调者与参与者网络出现问题，无法收到abort请求，参与者都会在等待超时后，继续进行事务提交

3PC优点：降低阻塞范围，在单点故障后继续达成一致

缺点:在参与者收到preCommit消息后，如果网络出现分区，协调-参与之间通信异常，参与者继续进行事务操作，带来数据不一致。

#### Paxos算法

拜占庭将军问题： 在异步系统和不可靠通道上来达到一致性状态是不可能的。所以在对一致性的研究过程中，都假设信道可靠。

Paxos算法的前提也是，消息可能有不可预知的延迟，重复，或丢失，但不会被篡改

Paxos的目标是：不论发生任何异常，保证最终有一个提案会被选定，进程也最终能获取到被选定的提案

Paxos算法的核心：过半数投票

角色：Proposer， Acceptor,  Learner

#### 推导过程：

最终有一个提案被选定，当只有一个提案时也能选定-&gt;

推导一： p1 :一个Acceptor必须批准它收到的第一个提案

推导二：一个Acceptor必须能够批准不止一个提案

推导三：当一个具有Value值的提案被半数以上的Acceptor批准后，我们认为该Value被**选定**了，此时我们也认为该提案被**选定**了

由三可推导出-&gt;

推导四：允许多个提案被选定,但所有被选定的提案都用相同的value

推导-&gt;p2

如果编号M0，value为V0的提案被选定了，那么所有比编号M0高的，且被选定的提案，其Value的值也必须是V0

想要满足p2，可以通过满足p2a来实现：

p2a:

如果编号M0，value为V0的提案被选定了，那么所有比编号M0高的，且被Acceptor批准的提案，其Value的值也必须是V0

由于：一个提案可能会某个Acceptor未收到任何提案时就被选定，而当它收到一个编号更高的提案，并且必须通过（p1）时，就会与p2a矛盾，所以，必须对p2a进行强化：

p2b: 如果一个提案\[M0,V0\]被选定后，那之后的任何Proposer产生的编号更高的提案\(Mn&gt;M0\)，其value都为v0

所以，只要论证p2b成立即可

证明p2b过程（第二数学归纳法）：

证明p2b，可以通过证明以下结论满足：

**假设**编号在M0到Mn-1之间的提案，其value都是V0,   **证明**编号Mn的提案value也是v0

因为M0已经被选定了，意味着肯定存在一个由半数以上的Acceptor组成的集合C，C中的每个Acceptor都批准了该提案。

再结合假设，得知：

C中的每个Acceptor都批准了一个编号在M0到M1范围内的提案，并且每个编号在M0到Mn-1范围内的，被Acceptor批准的提案，其值都为V0

因为任何包含半数以上Acceptor的集合S都至少包含C中的一个成员，因此我们可以认为如果保持了下面p2c的不变性，那么编号Mn的提案的value也为V0

p2c: 对于任意的Mn和Vn，如果提案\[Mn,Vn\]被提出，那么肯定存在一个由半数以上的Acceptor组成的集合S，满足以下两个条件中的任意一个：

* S中不存在任何批准过编号小于Mn提案的Acceptor

#### 

#### 算法陈述

阶段一：

1. Proposer选择一个提案编号Mn，然后向Acceptor的某个超过半数的子集成员发送编号为Mn的prepare请求
2. 如果一个Acceptor收到一个编号为Mn的Prepare请求。且编号Mn大于该Acceptor已经响应的所有Prepare请求的编号，那么它就会将它已经批准过的最大编号的提案作为响应反馈给Proposer，同时该Acceptor会承诺不会再批准任何编号小于Mn的提案

阶段二：

1. 如果Proposer收到来自半数以上的Acceptor对于其发出的编号为Mn的Prepare请求的响应，那么它就会发送一个针对\[Mn,Vn\]提案的Accept请求给Acceptor。注意，Vn就是收到的响应中编号最大的提案的值，如果响应中不包含任何提案，那么它就是任意值。
2. 如果Acceptor收到这个针对\[Mn,Vn\]提案的Accept请求，只要该Acceptor尚未对编号大于Mn的Prepare请求做出响应，它就可以通过这个提案

#### 提案的获取

Learner获取一个已经被选定的提案的前提是，该提案已经被半数以上的Acceptor批准。让所有的Acceptor将它们对提案的批准情况，统一发送给一个特定的Learner（为避免单点故障，可以优化为一个集群\)。Learner之间通过消息通信相互告知。

#### 选取主Proposer保证算法活性

为了避免极端情况的”死循环“，必须选择一个主Proposer，规定只有主Proposer才能提出提案，这样只要主Proposer能够和半数以上的Acceptor通信，主Proposer提出一个编号更高的提案，就会被批准

_**（这段没看明白。如果是通用做法，那主Proposer是怎样和其他Proposer交换信息的？还是只是面对一些极端情况下的舍车保帅做法？）**_

# 第三章: Paxos的工程实践

### Chubby

chubby是一个面向松耦合分布式系统的锁服务。在众多应用场景中，最为典型的就是集群中服务器的Master选举

一个典型的Chubby集群，通常有5台服务器组成。这些副本服务器采用Paxos协议，通过投票的方式来选举产生一个获得过半投票的服务器作为Master。**chubby的所有工作，不管是写还是读，都是由master节点来完成。其他所有节点只做备份。客户端请求chubby时，通过DNS来找到master**

#### Paxos协议实现

集群中的每个服务器都维护着一份服务端数据库的副本，Master才能进行写操作，其他都是使用Paxos从master进行同步更新

\#个人理解：

这儿的同步和mysql的主从同步不同。

##### mysql：

从库生成两个线程，一个I/O线程，一个SQL线程；

i/o线程去请求主库 的binlog，并将得到的binlog日志写到relay log（中继日志） 文件中；

主库会生成一个 log dump 线程，用来给从库 i/o线程传binlog；

SQL 线程，会读取relay log文件中的日志，并解析成具体操作，来实现主从的操作一致，而最终数据一致；

##### chubby

采用paxos来实现同步。master进行”Promise-&gt;Accept“阶段，写入本地日志，然后广播COMMIT消息给其他副本节点，并写入日志。如果其他副本没有收到COMMIT,则询问其他副本进行查询。



集群中的某台机器在宕机重启后，为了恢复状态机的状态，最简单的方法就是将已经记录的所有事务日志重新执行一遍

为了提高整个集群的性能，一小部分事务日志也可以通过从其他正常运行的副本上复制来进行获取，因此不需要实时地进行事务日志的flush

# 第四章: Zookeeper的ZAB协议
## 协议介绍

Zookeeper是一个高可用的分布式协调服务，用来完成一系列诸如可靠地配置存储和运行时状态记录等分布式协调工作，能够很好地在高并发情况下完成分布式数据一致性处理。

Zookeeper中，依赖ZAB协议来实现分布式数据一致性。基于该协议，**Zookeeper实现了一种主备模式的系统架构来保持集群中各副本之间的数据一致性**。具体的，Zookeeper使用一个单一的主进程来接收并处理客户端的所有事务请求，并采用ZAB的原子广播协议，将服务器数据的状态变更以事务Proposal的形式广播到所有副本进程

当Leader服务器出现崩溃或者重启，亦或是集群中已经不存在过半的服务器与该Leader服务器保持正常通信时，那么在重新开始新一轮原子广播事务操作前，所有进程首先会使用崩溃协议来使彼此达到一个一致的状态

一个机器要成为新的Leader，必须获得过半进程的支持。产生新的Leader后再次进入消息广播模式。

 > 最重要的一点要记住：如果一个事务在在Leader服务器上被提交，那最终会被所有服务器都提交。

### 消息广播
ZAB协议的消息广播过程使用的是一个原子广播协议，类似于一个二阶段提交。但是移除了中断逻辑，意味着不需要所有Follower服务器反馈相应，只要过半就行。

在消息广播过程中，Leader服务器会为每个事务请求生成对应的Proposal来进行广播，并且在广播事务Proposal之前，Leader服务器会首先为这个事务Proposal分配一个全局单调递增的唯一ID（ZXID）。由于ZAB协议需要保证每一个消息严格的因果关系，因此必须将每一个事务Proposal按照ZXID的先后顺序来进行排序和处理。


Leader服务器会为每一个Follower服务器都各自分配一个单独的队列，然后将需要广播的事务Proposal依次放入这些队列中去，根据FIFO策略进行发送。Follower收到这个Proposal之后，首先以事务日志的方式写到本地磁盘中去，并且在成功写入后反馈给Leader一个ACK。Leader接收到半数ACK后，广播一个Commit消息给所有的Follower，通知其提交，同时Leader自身也完成提交。

### 崩溃恢复
- ZAB协议需要确保那些已经在Leader服务器上**提交**的事务最终被所有服务器都提交。
- ZAB协议需要确保丢弃那些只在Leader服务器上**提出**的事务

针对这个要求，如果让Leader选举算法能够保证新选举出来的Leader服务器拥有集群中所有机器最高编号（ZXID最大）的事务Proposal，那个就可以保证这个新选举出来的Leader一定具有所有已经提交的提案。同时也省去Leader服务器检查Proposal的提交和丢弃工作的这一步操作了。

### 数据同步

完成Leader选举后，在正式工作（接收客户端事务请求，提出新提案）之前，Leader服务器会首先确认事务日志中的所有Proposal是否都已经被集群中过半的机器提交了，即是否完成数据同步。

Leader将那些没有被各Follower服务器同步的事务以Proposal消息的形式逐个发送给Follower服务器，并在每一个Proposal消息后面紧接着再发送一个Commit消息。等Follower已经将所有未同步的Proposal成功同步后，Leader服务器会将该Follower服务器加入到真正的可用Follower列表中

接下来看看ZAB如何处理那些需要被丢弃的事务Proposal的。
当选举产生一个新的Leader服务器，就会从这个Leader服务器上取出其本地日志中最大事务Proposal的ZXID，并解析出epoch值，进行加一，作为新的epoch，代表新的周期。
所以，当一个包含了上一个Leader周期中尚未提交过的事务Proposal的服务器启动时，其肯定无法成为Leader。因为该集群机器中一定包含了更高epoch事务Proposal，因此这台机器Proposal肯定不是最高的。
当这个机器加入集群后，以Follower的角色连上Leader，Leader会根据自己最后被提交的Proposal来和Follower比对，要求进行回退，到一个已经被半数机器提交的最新Proposal

### 算法描述
#### 阶段一：发现
- Follower F 将自己最后接受的事务Proposal的epoch值CEPOCH(Fp)发送给准Leader L
- 当接收到过半F 的epoch消息后，L会生成NewEpoch(e')（最大epoch+1）消息给这些过半F
- F收到来自L的消息后，如果自己的CEPOCH(Fp)小于e'，则将epoch赋值为e'，同时向这个准Leader L 反馈ack。这个消息中，包含了该F的epoch，以及该F的历史Proposal集合
- 当L接收到过半Ack之后，L就会从中选取一个F，并使用其作为初始化事务集合Ie'

#### 阶段二：同步
- L会将e'和Ie'以NewLeader(e',Ie')消息形式发送给所有集合中的Follower
- F收到消息后，如果发现自己的CEPOCH(Fp)不等于e'，则直接进入下一轮循环，因为还在上一轮，无法参加同步。如果相等，Follower就会执行事务应用操作。最后会反馈Leader，表明自己已经接受并处理Ie'中的事务。
- 当Leader接受来自半数F的反馈消息后，会向所有F发送commit
- F收到commit后，依次处理并提交Ie'中未处理的事务

#### 阶段三：广播
- Leader L接收到客户端新的事务请求后，会生成对应的Proposal，并依据ZXID向所有Follower发送提案
- F 根据消息接收的先后次序来处理这些来自L的事务，并加入完成序列当中，然后反馈给Leader
- L 收到半数F的Ack后，会发送commit给全部F，要求进行事务提交
- 当F收到commit后，开始进行提交



## ZAB 与Paxos协议区别

ZAB协议额外添加了一个同步阶段，能够有效保证Leader在新的周期中提出的Proposal之前，所有的进程已经完成了对之前所有Proposal的提交

 总的来说，ZAB协议和Paxos算法的本质区别是，两者的设计目标不一样。ZAB协议主要用于构建一个高可用的分布式**数据主备系统**（保证完整性），而Paxos算法则是用于构建一个分布式的**一致性状态机系统**（保证一致性）


