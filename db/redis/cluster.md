## Redis主从同步
Redis主从数据时异步同步，满足可用性，不满足一致性，只满足最终一致性

### 增量同步
redis的同步时指令流，主节点会将那些对自己的状态产生修改性影响的指令记录在本地内存buffer中，然后异步将buffer中的指令同步到从节点。因为内存buffer有限，不能将所有指令记录在内存中，所以采用一个定长的环形数组，满了会从头覆盖。
如果网络原因，导致增量同步延迟被覆盖。这时候就需要用到快照同步了。

### 快照同步
首先在主节点上进行一次bgsave，将当前内存的数据全部快照到磁盘文件中，然后再将快照文件的内容全部传送到从节点。
从节点接收到后，立即进行一次清空然后全量加载。
从节点刚加入集群时，必须先进行一次快照同步。再进行增量同步。

无盘同步：直接通过套接字将快照发送给从节点

## Sentinel
当故障发生时可以自动进行主从切换。
客户端来连接集群时，会首先连接sentinel，通过sentinel来查询主节点地址。当主节点故障时，客户端会重新向sentinel要地址，sentinel会将新的主节点地址给客户端。

## Codis
Redis集群化方案，客户端向codis发送指令时，负责将指令转发到后面的redis实例来执行，再将结果返回转给客户端。
codis将所有的key划分为1024个槽位。首先对传过来的key进行hash计算，然后取模得到key
的槽位。

因为槽位信息在内存中，所以codis集群实例直接还需要进行同步。可利用Zookeeper或者etcd进行同步。

redis扩容时，需要对codis槽位进行调整迁移。在迁移过程中，如果有新的请求，因为没法判断在新旧哪个槽位，所以强制迁移到新槽位。

## Cluster
官方的Redis集群化方案，去中心化。
当redis-cluster的客户端来连接集群时，会得到一份集群的槽位配置信息，这样客户端查找时可以直接定位。同时，可以缓存这份槽位信息。

当客户端向一个错误的节点发出指令后，会向客户端发出一个跳转指令。客户端收到后，要纠正本地的槽位表。
