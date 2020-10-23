---
layout:     post
title:      "Zookeeper"
date:       2019-12-22 19:00:00
author:     "SH"
header-img: "img/post_bg_headset.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Zookeeper
    - BBigData
---


# Zookeeper

## 简介

Zookeeper从设计模式角度来理解：是一个基于观察者设计模式设计的分布式服务管理框架，它负责存储和管理大家都关心的数据，然后接受观察者的注册，一旦这些数据的状态发生变化，zookeeper就将负责通知已经在zookeeper上注册的那些观察者做出相应的反应。

**zookeeper = 文件系统 + 通知机制。**（协调服务）

- zookeeper的核心是一个精简的文件系统，它提供一些简单的操作和一些额外的抽象操作，例如，排序和通知。

zookeeper数据模型的结构与Unix文件系统很类似，整体上可以看作是一棵树，每个节点称作一个ZNode（没有目录、文件区别）。每一个ZNode默认能够存储1MB的数据，每个ZNode都可以通过其路径唯一标识。

特点：

1）一个领导者（Leader），多个跟随者（Follower）组成的集群；

2）集群中只要有半数以上节点存活，集群就能正常服务；

3）全局数据一致：每个Server保存一份相同的数据副本，Client无论连接到哪个Server，数据都是一致的；

4）更新请求顺序进行，来自同一个Client的更新请求按其发送顺序依次执行；

5）数据更新原子性，一次数据更新要么成功，要么失败；

6）实时性，在一定时间范围内，Client能读到最新数据。

应用场景：HA（High Available）、Kafka、Hbase；统一命名服务、统一配置管理、统一集群管理、软负载均衡。

客户端命令：

| 命令基本语法       | 功能描述                                               |
| ------------------ | ------------------------------------------------------ |
| help               | 显示所有操作命令                                       |
| ls path [watch]    | 使用 ls 命令来查看当前znode中所包含的内容              |
| ls2 path   [watch] | 查看当前节点数据并能看到更新次数等数据                 |
| create             | 普通创建   -s  含有序列   -e  临时（重启或者超时消失） |
| get path   [watch] | 获得节点的值                                           |
| set                | 设置节点的具体值                                       |
| stat               | 查看节点状态                                           |
| delete             | 删除节点                                               |
| rmr                | 递归删除节点                                           |

## 内部原理

### 节点类型

- **有序、无序**：序号是全局顺序编号；
- **临时、永久**：客户端和服务器端断开连接后，节点是否删除；

四种节点类型：

1. 持久化目录节点：客户端与zookeeper断开连接后，该节点依旧存在；
2. 持久化顺序编号目录节点：客户端与zookeeper断开连接后，该节点依旧存在，zookeeper给该节点进行顺序编号；
3. 临时目录节点：客户端与zookeeper断开连接后，该节点被删除；
4. 临时顺序编号目录节点：客户端与zookeeper断开连接后，该节点被删除，zookeeper给该节点进行顺序编号。

### 节点状态

Stat结构体：

- **czxid**：创建节点的事务zxid，每次修改ZooKeeper状态都会收到一个zxid形式的时间戳，也就是ZooKeeper事务ID，事务ID是ZooKeeper中所有修改总的次序，每个修改都有唯一的zxid，如果zxid1小于zxid2，那么zxid1在zxid2之前发生；

- ctime：znode被创建的毫秒数(从1970年开始)；

- **mzxid**：znode最后更新的事务zxid；

- mtime：znode最后修改的毫秒数(从1970年开始)；

- pZxid**：znode最后更新的子节点zxid；

- cversion：znode子节点变化号，znode子节点修改次数；

- dataversion：znode数据变化号；

- aclVersion：znode访问控制列表的变化号；

- ephemeralOwner：如果是临时节点，这个是znode拥有者的session id，如果不是临时节点则是0；

- dataLength：znode的数据长度；

- numChildren：znode子节点数量。

### 监听器原理

```reStructuredText
-- new Zookeeper()
    -- new ClientCnxn()
    	-- new SendThread()
    	-- new EventThread()
```

- 在main线程中创建Zookeeper客户端，这时就会创建两个线程，一个负责网络连接通信（connect），一个负责监听（listener）；
- 通过connect线程将注册的监听事件发送给zook；
- 在zookeeper的注册监听器列表中将注册的监听事件添加到列表中；
- zookeeper监听到有数据或路径变化，就会将这个消息发送给listener线程；
- listener线程内部调用process()方法

![1577019725337](/img/Zookeeper/zookeeper_监听器原理.png)

常见的监听：

- 监听节点数据的变化，get path watch；
- 监听子节点增减的变化，ls path watch。

### ZAB协议

ZAB协议：ZooKeeper Atomic Broadcast，ZooKeeper 原子广播协议。

- 没有leader选leader（崩溃恢复）；
- 有leader就干活（增删读写）。

#### Paxos算法

Paxos算法一种基于消息传递且具有高度容错特性的一致性算法。

分布式系统中的节点通信存在两种模型：共享内存（Shared memory）和消息传递（Messages passing）。基于消息传递通信模型的分布式系统，不可避免的会发生以下错误：进程可能会慢、被杀死或者重启，消息可能会延迟、丢失、重复，在基础 Paxos 场景中，先不考虑可能出现消息篡改即拜占庭错误的情况。Paxos 算法解决的问题是在一个可能发生上述异常的分布式系统中如何就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性。

在一个Paxos系统中，首先将所有节点划分为Proposers、Acceptors、Learners（每个节点都可以身兼数职）；

一个完整的Paxos算法流程分为三个阶段：

- **Prepare阶段：**
  - Proposer向Acceptors发出Prepare请求Promise（承诺）；
  - Acceptors针对收到的Prepare请求进行Promise承诺；
- **Accept阶段：**
  - Proposer收到多数Acceptors承诺的Promise后，向Acceptors发出Propose请求；
  - Acceptors针对收到的Propose请求进行Accept处理；
- **Learn阶段：**
  - Proposer将形成的决议发送给所有Learners；

流程中的每条消息描述如下：

1. **Prepare:** Proposer生成全局唯一且递增的Proposal ID (可使用时间戳加Server ID)，向所有Acceptors发送Prepare请求，这里无需携带提案内容，只携带Proposal ID即可。

2. **Promise:** Acceptors收到Prepare请求后，做出“**两个承诺，一个应答**”。

两个承诺：

​	a. 不再接受Proposal ID小于等于（注意：这里是<= ）当前请求的Prepare请求。

​	b. 不再接受Proposal ID小于（注意：这里是< ）当前请求的Propose请求。

一个应答：

​	c. 不违背以前做出的承诺下，回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值。

3. **Propose:** Proposer 收到多数Acceptors的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。如果所有应答的提案Value均为空值，则可以自己随意决定提案Value。然后携带当前Proposal ID，向所有Acceptors发送Propose请求。

4. **Accept:** Acceptor收到Propose请求后，在不违背自己之前做出的承诺下，接受并持久化当前Proposal ID和提案Value。

5. **Learn:** Proposer收到多数Acceptors的Accept后，决议形成，将形成的决议发送给所有Learners。

Paxos算法缺陷：在网络复杂的情况下，一个应用Paxos算法的分布式系统，可能很久无法收敛，甚至陷入活锁的情况。

![1577021201059](/img/Zookeeper/paxos算法.png)

![1577021099781](/img/Zookeeper/paxos算法_活锁.png)

系统中有一个以上的Proposer，多个Proposers相互争夺Acceptors，会造成迟迟无法达成一致的情况。针对这种情况，一种改进的Paxos算法被提出：从系统中选出一个节点作为Leader，只有Leader能够发起提案。这样，一次Paxos流程中只有一个Proposer，不会出现活锁的情况，此时只会出现例子中第一种情况。

#### 选举机制

1）半数机制：集群中半数以上机器存活，集群可用。所以Zookeeper适合安装奇数台服务器。

2）Zookeeper虽然在配置文件中并没有指定Master和Slave。但是，Zookeeper工作时，是有一个节点为Leader，其他则为Follower，Leader是通过内部的选举机制临时产生的。

假设有五台服务器组成的Zookeeper集群，它们的id从1-5，同时它们都是最新启动的，也就是没有历史数据，在存放数据量这一点上，都是一样的。假设这些服务器依序启动：

1. 服务器1启动，发起一次选举。服务器1投自己一票。此时服务器1票数一票，不够半数以上（3票），选举无法完成，服务器1状态保持为**LOOKING**；

2. 服务器2启动，再发起一次选举。服务器1和2分别投自己一票并交换选票信息：此时服务器1发现服务器2的ID比自己目前投票推举的（服务器1）大，更改选票为推举服务器2。此时服务器1票数0票，服务器2票数2票，没有半数以上结果，选举无法完成，服务器1，2状态保持**LOOKING**；

3. 服务器3启动，发起一次选举。此时服务器1和2都会更改选票为服务器3。此次投票结果：服务器1为0票，服务器2为0票，服务器3为3票。此时服务器3的票数已经超过半数，服务器3当选Leader。服务器1，2更改状态为**FOLLOWING**，服务器3更改状态为**LEADING**；

4. 服务器4启动，发起一次选举。此时服务器1，2，3已经不是LOOKING状态，不会更改选票信息。交换选票信息结果：服务器3为3票，服务器4为1票。此时服务器4服从多数，更改选票信息为服务器3，并更改状态为**FOLLOWING**；

5. 服务器5启动，同4一样当小弟。

#### 写数据流程

![1577022020162](/img/Zookeeper/zookeeper_写数据流程.png)

#### Zab 与 Paxos的区别

Paxos 的思想在很多分布式组件中都可以看到，Zab 协议可以认为是基于 Paxos 算法实现的，先来看下两者之间的联系：

- 都存在一个 Leader 进程的角色，负责协调多个 Follower 进程的运行；
- 都应用 Quroum 机制，Leader 进程都会等待超过半数的 Follower 做出正确的反馈后，才会将一个提案进行提交；
- 在 Zab 协议中，Zxid 中通过 epoch 来代表当前 Leader 周期，在 Paxos 算法中，同样存在这样一个标识，叫做 Ballot Number；

两者之间的区别是，**Paxos是理论，Zab是实践**，Paxos是论文性质的，目的是设计一种通用的分布式一致性算法，而 Zab 协议应用在 ZooKeeper 中，是一个特别设计的崩溃可恢复的原子消息广播算法。

Zab 协议增加了崩溃恢复的功能，当 Leader 服务器不可用，或者已经半数以上节点失去联系时，ZooKeeper 会进入恢复模式选举新的 Leader 服务器，使集群达到一个一致的状态。

### 其他

- [腾讯技术工程-浅谈 CAP 和 Paxos 共识算法](https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649745393&idx=1&sn=90f64ea82007b201cf58d1b7e24d28d3)
- [腾讯技术工程-ZooKeeper 源码和实践揭秘](https://mp.weixin.qq.com/s?__biz=MjM5ODYwMjI2MA==&mid=2649745966&idx=1&sn=50a6b9892783b9509c02ac0db0f4167e)

