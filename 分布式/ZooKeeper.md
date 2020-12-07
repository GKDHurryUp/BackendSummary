# Zookeeper

核心：一致，有头，数据树
特点：服务器为单数，为了自动选举不出现单数。整个集群里剩余机器超过集群的一半，可以正常运行，否则就挂掉。
所有客户端的访问会到达leader处，由leader进行转发

## zookeeper是颗数据树 ##
根节点为/
根节点下有znode，它是服务器+data

## 应用场景 ##
1. 集群的配置一致	
	依靠数据一致性
2. HA 高可用
	主备动态切换
3. naming service
4. pub/hub 发布订阅
4. load balance 负载均衡
5. 分布式锁 

## Zookeeper节点

zookeeper 中节点叫znode存储结构上跟文件系统类似，以树级结构进行存储。不同之外在于znode没有目录的概念，不能执行类似cd之类的命令。znode结构包含如下

| 节点类型              | 描述                           |
| :-------------------- | :----------------------------- |
| PERSISTENT            | 持久节点                       |
| PERSISTENT_SEQUENTIAL | 持久序号节点                   |
| EPHEMERAL             | 临时节点(不可在拥有子节点)     |
| EPHEMERAL_SEQUENTIAL  | 临时序号节点(不可在拥有子节点) |

## ZAB协议 ##

ZAB 协议是为分布式协调服务ZooKeeper专门设计的一种支持崩溃恢复的一致性协议。

1. leader
2. 过半机制
3. 2PC 两阶段提交
4. 同步信息

## 领导者选举机制 ##
1. 投票
2. 投给自己
3. 交流比较（zxid）
4. 投票箱
5. 统计

## 领导者选举发生的时间点 ##
1. 集群启动
2. Leader挂掉
3. Follower挂掉后，Leader发现已经没有过半的Folloer跟随自己了（不能对外提供服务了）

## 两阶段提交 ##
1. 记录日志，leader把日志发送给各个follower，各个follower进行持久化，回复ack

## 领导者选举算法中为什么要使用HashMap存储zxid？为什么不用List？ ##
为了方便更新，使用HashMap，把node节点作为key，把zxid作为value，方便值的更新。如果使用List就需要把它的

## 快速领导者选举算法 ##

![](https://cdn.nlark.com/yuque/0/2019/png/365147/1560666915378-6f10ffe0-3792-4bc1-afad-12323a3fbd7c.png)
1. 个人能力：通过事务id(zxid)表示数据的新旧，一个节点最新的zxid越大则该节点的数据越新，也就代表次节点能力越强
2. 改票：首先认为自己的数据是最新的，会先投自己一票，并且把这张选票（peerEpoch，zxid，sid）发送给其他服务器，同时也会接收到其他服务器的选票，首先比较peerEpoch，其次比较zxid，最后比较sid判断是否需要改票
3. 投票箱：Zookeeper集群并不会单独去维护一个投票箱，而是在每个节点内存里利用一个数组来作为投票箱。节点会将自己的选票以及从其他服务器接收到的选票放在这个投票箱中。
4. 领导者：每个节点进行选票PK，将自己的选票修改为投给数据最新的节点，每个节点都可以查看自己投票箱的选票，一旦集群中超过一半的节点都认为某一个节点上数据最新，则该节点就是领导者

### QuorumCnxManage ###
QuorumCnxManager就是传输层实现，QuorumCnxManager中几个重要的属性：
> •ConcurrentHashMap<Long, ArrayBlockingQueue<ByteBuffer>> **queueSendMap**
• ConcurrentHashMap<Long, SendWorker> **senderWorkerMap**
• ArrayBlockingQueue<Message> **recvQueue**
• QuorumCnxManager.Listener

传输层的每个zkServer需要发送选票信息给其他服务器，这些选票信息来至应用层，在传输层中将会按服务器id分组保存在queueSendMap中。

传输层的每个zkServer需要发送选票信息给其他服务器，SendWorker就是封装了Socket的发送器，而senderWorkerMap就是用来记录其他服务器id以及对应的SendWorker的。

传输层的每个zkServer将接收其他服务器发送的选票信息，这些选票会保存在recvQueue中，以提供给应用层使用

QuorumCnxManager.Listener负责开启socket监听

![](https://cdn.nlark.com/yuque/0/2019/png/365147/1560671013696-ec9a62d5-dcc4-469b-8a43-280215576cb2.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_10%2Ctext_6bKB54-t5a2m6Zmi5ZGo55Gc%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)


服务器有四个状态

	public enum ServerState {
	    LOOKING,
	    FOLLOWING,
	    LEADING,
	    OBSERVING
	}

## 服务器之间连接问题 ##
在集群启动时，一台服务器需要去连另外一台服务器，从而建立Socket用来进行选票传输，Socket是双向的。ZooKeeper在实现时做了限制，只允许服务器ID较大者去连服务器ID较小者，小ID服务器去连大ID服务器会被拒绝

	if (sid < self.getId()) {
		closeSocket(sock); // 关闭这条socket
		connectOne(sid);   // 由本服务器去连对方服务器
	} else {
	    // 继续建立连接
	}

# 脑裂问题 #
## 什么是脑裂？  ##
脑裂(split-brain)就是“大脑分裂”，通常会出现在集群环境中，比如ElasticSearch、Zookeeper集群，它们有一个大脑，比如ElasticSearch集群中有Master节点，Zookeeper集群中有Leader节点

## 脑裂场景 ##
对于一个集群，想要提高这个集群的可用性，通常会采用多机房部署，比如现在有一个由6台zkServer所组成的一个集群，部署在了两个机房：
![](https://cdn.nlark.com/yuque/0/2019/png/365147/1563867147007-e5000b66-fbe7-4958-89c7-11800de04f7c.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_10%2Ctext_6bKB54-t5a2m6Zmi5ZGo55Gc%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

正常情况下，此集群只会有一个Leader，那么如果机房之间的网络断了之后，两个机房内的zkServer还是可以相互通信的，如果**不考虑过半机制**，那么就会出现每个机房内部都将选出一个Leader。
![](https://cdn.nlark.com/yuque/0/2019/png/365147/1563867309583-b3c9d494-d91e-41f0-bb1f-310354cc14c4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_10%2Ctext_6bKB54-t5a2m6Zmi5ZGo55Gc%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

## 过半机制 ##
在领导者选举的过程中，如果某台zkServer获得了超过半数的选票，则此zkServer就可以成为Leader了。

```java
public class QuorumMaj implements QuorumVerifier {
    private static final Logger LOG = LoggerFactory.getLogger(QuorumMaj.class);
    
    int half;
    
    // n表示集群中zkServer的个数（准确的说是参与者的个数，参与者不包括观察者节点）
    public QuorumMaj(int n){
        this.half = n/2;
    }

    // 验证是否符合过半机制
    public boolean containsQuorum(Set<Long> set){
        // half是在构造方法里赋值的
        // set.size()表示某台zkServer获得的票数
        return (set.size() > half);
    }
}
```

## 过半机制中为什么是大于，而不是大于等于呢？ ##