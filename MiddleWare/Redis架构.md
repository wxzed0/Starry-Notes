# Redis架构

[TOC]



## 主从模式

主从模式最大的优点是**部署简单**，最少**两个节点便可以构成主从模式**，并且可以通过**读写分离避免读和写同时不可用**。不过，一旦 Master 节点出现故障，主从节点就**无法自动切换**所以，主从模式一般**适合业务发展初期，并发量低，运维成本低**的情况



**主从复制原理：**

 ①通过从服务器发送到PSYNC命令给主服务器

 ②如果是首次连接，触发一次**全量复制**。此时主节点会启动一个后台线程，生成 RDB 快照文件

 ③主节点会将这个 RDB 发送给从节点，slave 会先写入本地磁盘，再从本地磁盘加载到内存中

 ④master会将此过程中的写命令写入缓存，从节点**实时同步**这些数据

 ⑤如果网络断开了连接，自动重连后主节点通过命令传播**增量复制**给从节点部分缺少的数据

**缺点**

 所有的slave节点数据的复制和同步都由master节点来处理，会照成master节点压力太大，使用主从从结构来解决，redis4.0中引入psync2 解决了slave重启后仍然可以增量同步。



## 哨兵模式

 由一个或多个sentinel实例组成sentinel集群可以监视一个或多个主服务器和多个从服务器。**哨兵模式适合读请求远多于写请求的业务场景，比如在秒杀系统**中用来缓存活动信息。 如果写请求较多，当集群 Slave 节点数量多了后，Master 节点同步数据的压力会非常大。

![image-20201220231241725](images/68747470733a2f2f747661312e73696e61696d672e636e2f6c617267652f303038314b636b776c7931676c757136766c76676c6a33306e773065303736662e6a7067)

当主服务器进入下线状态时，sentinel可以将该主服务器下的某一从服务器升级为主服务器继续提供服务，从而保证redis的高可用性。

**检测主观下线状态**

 Sentinel每秒一次向所有与它建立了命令连接的实例(主服务器、从服务器和其他Sentinel)发送PING命 令

 实例在down-after-milliseconds毫秒内返回无效回复Sentinel就会认为该实例主观下线(**SDown**)

**检查客观下线状态**

 当一个Sentinel将一个主服务器判断为主观下线后 ，Sentinel会向监控这个主服务器的所有其他Sentinel发送查询主机状态的命令

 如果达到Sentinel配置中的quorum数量的Sentinel实例都判断主服务器为主观下线，则该主服务器就会被判定为客观下线(**ODown**)。

**选举Leader Sentinel**

 当一个主服务器被判定为客观下线后，监视这个主服务器的所有Sentinel会通过选举算法(raft)，选出一个Leader Sentinel去执行**failover(故障转移)**操作。



**Raft算法**

 Raft协议是用来解决分布式系统一致性问题的协议。 Raft协议描述的节点共有三种状态:Leader, Follower, Candidate。 Raft协议将时间切分为一个个的Term(任期)，可以认为是一种“逻辑时间”。 选举流程: 

①Raft采用心跳机制触发Leader选举系统启动后，全部节点初始化为Follower，term为0

 ②节点如果收到了RequestVote或者AppendEntries，就会保持自己的Follower身份

 ③节点如果一段时间内没收到AppendEntries消息，在该节点的超时时间内还没发现Leader，Follower就会转换成Candidate，自己开始竞选Leader。 一旦转化为Candidate，该节点立即开始下面几件事情: --增加自己的term，启动一个新的定时器 --给自己投一票，向所有其他节点发送RequestVote，并等待其他节点的回复。

 ④如果在计时器超时前，节点收到多数节点的同意投票，就转换成Leader。同时通过 AppendEntries，向其他节点发送通知。

 ⑤每个节点在一个term内只能投一票，采取先到先得的策略，Candidate投自己， Follower会投给第一个收到RequestVote的节点。

 ⑥Raft协议的定时器采取随机超时时间（选举的关键），先转为Candidate的节点会先发起投票，从而获得多数票。



**主服务器的选择**

 当选举出Leader Sentinel后，Leader Sentinel会根据以下规则去从服务器中选择出新的主服务器。

1. 过滤掉主观、客观下线的节点
2. 选择配置slave-priority最高的节点，如果有则返回没有就继续选择
3. 选择出复制偏移量最大的系节点，因为复制偏移量越大则数据复制的越完整
4. 选择run_id最小的节点，因为run_id越小说明重启次数越少



**故障转移**

 当Leader Sentinel完成新的主服务器选择后，Leader Sentinel会对下线的主服务器执行故障转移操作，主要有三个步骤:

 1、它会将失效 Master 的其中一个 Slave 升级为新的 Master , 并让失效 Master 的其他 Slave 改为复制新的 Master ;

 2、当客户端试图连接失效的 Master 时，集群会向客户端返回新 Master 的地址，使得集群当前状态只有一个Master。

 3、Master 和 Slave 服务器切换后， Master 的 redis.conf 、 Slave 的 redis.conf 和 sentinel.conf 的配置文件的内容都会发生相应的改变，即 Master 主服务器的 redis.conf配置文件中会多一行 replicaof 的配置， sentinel.conf 的监控目标会随之调换。

![image](images/sentinel.jpg)

## 集群模式

集群通过分片实现数据共享，并提供复制和故障转移功能。

 为了避免单一节点负载过高导致不稳定，集群模式采用**一致性哈希算法或者哈希槽的方法**将 Key 分布到各个节点上。其中，每个 Master 节点后跟若干个 Slave 节点，用于**出现故障时做主备切换**，客户端可以**连接任意 Master 节点**，集群内部会按照**不同 key 将请求转发到不同的 Master** 节点

 集群模式是如何实现高可用的呢？集群内部节点之间会**互相定时探测**对方是否存活，如果多数节点判断某个节点挂了，则会将其踢出集群，然后从 **Slave** 节点中选举出一个节点**替补**挂掉的 Master 节点。**整个原理基本和哨兵模式一致**

 虽然集群模式避免了 Master 单节点的问题，但**集群内同步数据时会占用一定的带宽**。所以，只有在**写操作比较多的情况下人们才使用集群模式**，其他大多数情况，使用**哨兵模式**都能满足需求



