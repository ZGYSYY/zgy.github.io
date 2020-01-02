---
title: 什么是ZooKeeper
date: 2019-12-31 17:48:12
tags:
- zookeeper
- 分布式锁
- 服务注册与发现
categories:
- 后端
- Java
- Zookeeper
---

> 本文摘抄自： [ 程序员小灰——什么是ZooKeeper？](https://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653193977&idx=1&sn=12319f8cb81f55a40ac461bd0ad9d74e&chksm=8c99f423bbee7d35056ce7ec1b321f33aad15c309de6eba0086cb31a48b975ccb1d695d5a251&mpshare=1&scene=23&srcid=1009q5vbBfA0ssZhVnm61nEd&sharer_sharetime=1570673341100&sharer_shareid=f8be6084b82cc018b018588532bb3fe1#rd)

{% asset_img 640.webp %}

<!-- more -->

{% asset_img 640-1570673571208.webp %}

<center><b>—————  第二天  —————</b></center>

{% asset_img 640-1570673571211.webp %}

{% asset_img 640-1570673571216.webp %}

{% asset_img 0.jpg %}

{% asset_img 640-1570673571231.webp %}

{% asset_img 640-1570673571236.webp %}

{% asset_img 640-1570673571243.webp %}

{% asset_img 640-1570673571252.webp %}

{% asset_img 640-1570673571259.webp %}

{% asset_img 640-1570673571263.webp %}

{% asset_img 640-1570673571267.webp %}

<hr/>

{% asset_img 640-1570673571275.webp %}

{% asset_img 640-1570673571283.webp %}

{% asset_img 640-1570673571287.webp %}

{% asset_img 640-1570673571295.webp %}

{% asset_img 640-1570673628957.webp %}

{% asset_img 640-1570673628968.webp %}

{% asset_img 640-1570673628973.webp %}

# Zookeeper的数据模型

Zookeeper的数据模型是什么样子呢？它很像数据结构当中的树，也很像文件系统的目录。

{% asset_img 640-1570673692556.webp %}

树是由节点所组成，Zookeeper的数据存储也同样是基于节点，这种节点叫做**Znode**。

但是，不同于树的节点，Znode的引用方式是**路径引用**，类似于文件路径：

 /动物/仓鼠

 /植物/荷花

这样的层级结构，让每一个Znode节点拥有唯一的路径，就像命名空间一样对不同信息作出清晰的隔离。

{% asset_img 640-1570673692564.webp %}

{% asset_img 640-1570673692568.webp %}

{% asset_img 640-1570673692576.webp %}

**data**：Znode存储的数据信息。

**ACL**：记录Znode的访问权限，即哪些人或哪些IP可以访问本节点。

**stat**：包含Znode的各种元数据，比如事务ID、版本号、时间戳、大小等等。

**child**：当前节点的子节点引用，类似于二叉树的左孩子右孩子。

这里需要注意一点，Zookeeper是为读多写少的场景所设计。Znode并不是用来存储大规模业务数据，而是用于存储少量的状态和配置信息，**每个节点的数据最大不能超过1MB**。

{% asset_img 640-1570673692584.webp %}

{% asset_img 640-1570673692592.webp %}

# Zookeeper的基本操作和事件通知

Zookeeper包含了哪些基本操作呢？这里列举出比较常用的API：

**create**：创建节点

**delete**：删除节点

**exists**：判断节点是否存在

**getData**：获得一个节点的数据

**setData**：设置一个节点的数据

**getChildren**：获取节点下的所有子节点这其中，exists，getData，getChildren属于读操作。Zookeeper客户端在请求读操作的时候，可以选择是否设置**Watch**。

Watch是什么意思呢？

我们可以理解成是注册在特定Znode上的触发器。当这个Znode发生改变，也就是调用了create，delete，setData方法的时候，将会触发Znode上注册的对应事件，请求Watch的客户端会接收到**异步通知**。

具体交互过程如下：

1.客户端调用getData方法，watch参数是true。服务端接到请求，返回节点数据，并且在对应的哈希表里插入被Watch的Znode路径，以及Watcher列表。

{% asset_img 640-1570673692596.webp %}

2.当被Watch的Znode已删除，服务端会查找哈希表，找到该Znode对应的所有Watcher，异步通知客户端，并且删除哈希表中对应的Key-Value。

{% asset_img 640-1570673692604.webp %}

# Zookeeper的一致性

{% asset_img 640-1570673692613.webp %}

{% asset_img 640-1570673692617.webp %}

Zookeeper的集群长成什么样呢？就像下图这样：

{% asset_img 640-1570673692620.webp %}

Zookeeper Service集群是一主多从结构。

在更新数据时，首先更新到主节点（这里的节点是指服务器，不是Znode），再同步到从节点。

在读取数据时，直接读取任意从节点。

为了保证主从节点的数据一致性，Zookeeper采用了**ZAB协议**，这种协议非常类似于一致性算法**Paxos**和**Raft**。

{% asset_img 640-1570673692628.webp %}

{% asset_img 640-1570673692636.webp %}

在学习ZAB之前，我们需要首先了解ZAB协议所定义的三种节点状态：

**Looking** ：选举状态。

**Following** ：Follower节点（从节点）所处的状态。

**Leading** ：Leader节点（主节点）所处状态。

我们还需要知道**最大ZXID**的概念：

最大ZXID也就是节点本地的最新事务编号，包含**epoch**和计数两部分。epoch是纪元的意思，相当于Raft算法选主时候的term。

假如Zookeeper当前的主节点挂掉了，集群会进行**崩溃恢复**。ZAB的崩溃恢复分成三个阶段：

**1.Leader election**

选举阶段，此时集群中的节点处于Looking状态。它们会各自向其他节点发起投票，投票当中包含自己的服务器ID和最新事务ID（ZXID）。

{% asset_img 640-1570673692644.webp %}

接下来，节点会用自身的ZXID和从其他节点接收到的ZXID做比较，如果发现别人家的ZXID比自己大，也就是数据比自己新，那么就重新发起投票，投票给目前已知最大的ZXID所属节点。

{% asset_img 640-1570673692651.webp %}

每次投票后，服务器都会统计投票数量，判断是否有某个节点得到半数以上的投票。如果存在这样的节点，该节点将会成为准Leader，状态变为Leading。其他节点的状态变为Following。

{% asset_img 640-1570673692659.webp %}

这就相当于，一群武林高手经过激烈的竞争，选出了武林盟主。

**2.Discovery**

发现阶段，用于在从节点中发现最新的ZXID和事务日志。或许有人会问：既然Leader被选为主节点，已经是集群里数据最新的了，为什么还要从节点中寻找最新事务呢？

这是为了防止某些意外情况，比如因网络原因在上一阶段产生多个Leader的情况。

所以这一阶段，Leader集思广益，接收所有Follower发来各自的最新epoch值。Leader从中选出最大的epoch，基于此值加1，生成新的epoch分发给各个Follower。

各个Follower收到全新的epoch后，返回ACK给Leader，带上各自最大的ZXID和历史事务日志。Leader选出最大的ZXID，并更新自身历史日志。

**3.Synchronization**

同步阶段，把Leader刚才收集得到的最新历史事务日志，同步给集群中所有的Follower。只有当半数Follower同步成功，这个准Leader才能成为正式的Leader。

自此，故障恢复正式完成。

{% asset_img 640-1570673692668.webp %}

{% asset_img 640-1570673692672.webp %}

什么是**Broadcast**呢？简单来说，就是Zookeeper常规情况下更新数据的时候，由Leader广播到所有的Follower。其过程如下：

1.客户端发出写入数据请求给任意Follower。

2.Follower把写入数据请求转发给Leader。

3.Leader采用二阶段提交方式，先发送Propose广播给Follower。

4.Follower接到Propose消息，写入日志成功后，返回ACK消息给Leader。

5.Leader接到半数以上ACK消息，返回成功给客户端，并且广播Commit请求给Follower。

{% asset_img 640-1570673692680.webp %}

Zab协议既不是强一致性，也不是弱一致性，而是处于两者之间的**单调一致性**。它依靠事务ID和版本号，保证了数据的更新和读取是有序的。

# Zookeeper的应用

{% asset_img 640-1570673740840.webp %}

{% asset_img 640-1570673740844.webp %}

**1.分布式锁**

这是雅虎研究员设计Zookeeper的初衷。利用Zookeeper的临时顺序节点，可以轻松实现分布式锁。

**2.服务注册和发现**

利用Znode和Watcher，可以实现分布式服务的注册和发现。最著名的应用就是阿里的分布式RPC框架Dubbo。

**3.共享配置和状态信息**

Redis的分布式解决方案Codis，就利用了Zookeeper来存放数据路由表和 codis-proxy 节点的元信息。同时 codis-config 发起的命令都会通过 ZooKeeper 同步到各个存活的 codis-proxy。

此外，Kafka、HBase、Hadoop，也都依靠Zookeeper同步节点信息，实现高可用。

{% asset_img 640-1570673740848.webp %}

{% asset_img 640-1570673740856.webp %}

**几点补充：**

1.ZAB协议相对比较复杂，小灰对此也只是浅层次的理解，有兴趣的小伙伴们可以去官方社区进行进一步学习。

**2.本漫画纯属娱乐，还请大家尽量珍惜当下的工作，切勿模仿小灰的行为哦。**