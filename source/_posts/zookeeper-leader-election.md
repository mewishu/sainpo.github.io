---
title: zookeeper选举机制及源码解析
date: 2019-08-15 19:54:10
categories: 
- 分布式一致性
- zookeeper
tags:
- zookeeper
- Leader选举
---

## Zookeeper选举概述

### 时机

Leader选举是保证分布式数据一致性的关键所在。当Zookeeper集群中的一台服务器出现以下两种情况之一时，需要进入Leader选举: 

　　(1) 服务器初始化启动。

　　(2) 服务器运行期间无法和Leader保持连接。

在Zookeeper运行期间，Leader与非Leader服务器各司其职，即便当有非Leader服务器宕机或新加入，此时也不会影响Leader，但是一旦Leader服务器挂了，那么整个集群将暂停对外服务，进入新一轮Leader选举，其过程和启动时期的Leader选举过程基本一致。假设正在运行的有Server1、Server2、Server3三台服务器，当前Leader是Server2，若某一时刻Leader挂了，此时便开始Leader选举。

### 过程

若进行Leader选举，则至少需要两台机器，这里选取3台机器组成的服务器集群为例。在集群初始化阶段，当有一台服务器Server1启动时，其单独无法进行和完成Leader选举，当第二台服务器Server2启动时，此时两台机器可以相互通信，每台机器都试图找到Leader，于是进入Leader选举过程。Leader 选举过程，本质就是广播`优先级消息`的过程，选出**数据最新的服务节点**，选出**优先级最高的服务节点**，基本步骤：

1. **每个Server发出一个投票**。Server1和Server2都会将自己作为Leader服务器来进行投票，广播自己的优先级标识 `(sid，zxid)`。例如服务器启动时，此时Server1的投票为(1, 0)，Server2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。

2. **接受来自各个服务器的投票**。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器。

3. **处理投票**。服务器节点收到其他广播消息后，跟自己的优先级对比，自己优先级低，则变更当前节点投票的优先级`(sid，zxid)` ，并广播变更后的结果，优先级PK规则: 

   1. 优先比较 `zxid` （事务 ID），其次比较`sid`（服务器ID）
   2. `sid` (服务器 ID) 是节点配置文件中设定的

   对于Server1而言，它的投票是(1, 0)，接收Server2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时Server2的myid最大，于是更新自己的投票为(2, 0)，然后重新投票，对于Server2而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。

4. **统计投票**。当任意一个服务器节点收到的投票数，超过了`法定数量`(quorum)亦即集群规模大小的半数以上，则升级为 Leader，并广播结果。对于Server1、Server2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader。

5. **改变服务器状态**。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。



## Leader选举算法分析

在3.4.0后的Zookeeper的版本只保留了TCP版本的FastLeaderElection选举算法。当一台机器进入Leader选举时，当前集群可能会处于以下两种状态

- 集群中已经存在Leader。
- 集群中不存在Leader。

对于集群中已经存在Leader而言，此种情况一般都是某台机器启动得较晚，在其启动之前，集群已经在正常工作，对这种情况，该机器试图去选举Leader时，会被告知当前服务器的Leader信息，对于该机器而言，仅仅需要和Leader机器建立起连接，并进行状态同步即可。而在集群中不存在Leader情况下则会相对复杂，其步骤如下。

### 第一次投票

无论哪种导致进行Leader选举，集群的所有机器都处于试图选举出一个Leader的状态，即LOOKING状态，LOOKING机器会向所有其他机器发送消息，该消息称为投票。投票中包含了(sid，zxid) 形式来标识一次投票信息。假定Zookeeper由5台机器组成，`sid`分别为1、2、3、4、5，`zxid`分别为9、9、9、8、8，并且此时`sid`为2的机器是Leader机器，某一时刻，1、2所在机器出现故障，因此集群开始进行Leader选举。在第一次投票时，每台机器都会将自己作为投票对象，于是`sid`为3、4、5的机器投票情况分别为(3, 9)，(4, 8)， (5, 8)。

### 变更投票

每台机器发出投票后，也会收到其他机器的投票，每台机器会根据一定规则来处理收到的其他机器的投票，并以此来决定是否需要变更自己的投票，这个规则也是整个Leader选举算法的核心所在，其中术语描述如下: 

- **vote_sid**：接收到的投票中所推举Leader服务器的`sid`。
- **vote_zxid**：接收到的投票中所推举Leader服务器的`zxid`。
- **self_sid**：当前服务器自己的`sid`。
- **self_zxid**：当前服务器自己的`zxid`。

每次对收到的投票的处理，都是对(vote_sid, vote_zxid)和(self_sid, self_zxid)对比的过程。

1. 规则一：如果vote_zxid大于self_zxid，就认可当前收到的投票，并再次将该投票发送出去。
2. 规则二：如果vote_zxid小于self_zxid，那么坚持自己的投票，不做任何变更。
3. 规则三：如果vote_zxid等于self_zxid，那么就对比两者的SID，如果vote_sid大于self_sid，那么就认可当前收到的投票，并再次将该投票发送出去。
4. 规则四：如果vote_zxid等于self_zxid，并且vote_sid小于self_sid，那么坚持自己的投票，不做任何变更。

　　结合上面规则，给出下面的集群变更过程。

![Lead选举集群变更过程](616953-20161202213100568-693960760.png)

### 确定Leader

经过第二轮投票后，集群中的每台机器都会再次接收到其他机器的投票，然后开始统计投票，如果一台机器收到了超过半数的相同投票，那么这个投票对应的SID机器即为Leader。此时Server3将成为Leader。

由上面规则可知，通常那台服务器上的数据越新（`zxid`会越大），其成为Leader的可能性越大，也就越能够保证数据的恢复。如果`zxid`相同，则`sid`越大机会越大。



## Leader选举实现细节

### 服务器状态

服务器具有四种状态，分别是LOOKING、FOLLOWING、LEADING、OBSERVING。

**LOOKING**：寻找Leader状态。当服务器处于该状态时，它会认为当前集群中没有Leader，因此需要进入Leader选举状态。

**FOLLOWING**：跟随者状态。表明当前服务器角色是Follower。

**LEADING**：领导者状态。表明当前服务器角色是Leader。

**OBSERVING**：观察者状态。表明当前服务器角色是Observer。

### 投票数据结构

每个投票中包含了两个最基本的信息，所推举服务器的`sid`和`zxid`，投票（Vote）在Zookeeper中包含字段如下

| 属性          | 说明                                                         |
| :------------ | :----------------------------------------------------------- |
| id            | 被推举 Leader 的 `sid`                                       |
| zxid          | 被推举 Leader 的`zxid`                                       |
| electionEpoch | 投票的轮次，逻辑时钟，用来判断多个投票是否在同一轮选举周期中，该值在服务端是一个自增序列，每次进入新一轮的投票后，都会对该值进行加1操作 |
| peerEpoch     | 被推举 Leader 的 epoch                                       |
| state         | 当前服务器的状态                                             |

一次 Leader 选举过程，属于同一个 `electionEpoch`，结束时，会选出新的 Leader；服务器节点，在比较 `(sid，zxid)` 之前，会先比较选举轮次 `electionEpoch`，只有同一轮次的 Leader 投票信息才是有效的：

1. 外部投票轮次 > 内部投票轮次，更新内部投票，并且触发当前节点投票信息的**重新广播**
2. 外部投票轮次 < 内部投票轮次，直接忽略当前的外部投票
3. 外部投票轮次 = 内部投票轮次，进一步比较 `(sid，zxid)`

疑问：Leader 负责执行所有的事务操作，一次事务操作，

1. Leader 如何将事务操作同步到 Follower 和 Observer ？同步、异步？
2. 如何保证同步过程中，事务一定执行成功？事务失败的影响？

Leader 上执行的事务状态，通过 `Zab` 状态更新的广播协议，更新到 Follower 和 Observer。

### QuorumCnxManager：网络I/O

每台服务器在启动的过程中，会启动一个QuorumPeerManager，负责各台服务器之间的底层Leader选举过程中的网络通信。

#### 消息队列

QuorumCnxManager内部维护了一系列的队列，用来保存接收到的、待发送的消息以及消息的发送器，除接收队列以外，其他队列都按照SID分组形成队列集合，如一个集群中除了自身还有3台机器，那么就会为这3台机器分别创建一个发送队列，互不干扰。

- **recvQueue**：消息接收队列，用于存放那些从其他服务器接收到的消息。
- **queueSendMap**：消息发送队列，用于保存那些待发送的消息，按照`sid`进行分组。
- **senderWorkerMap**：发送器集合，每个SenderWorker消息发送器，都对应一台远程Zookeeper服务器，负责消息的发送，也按照SID进行分组。
- **lastMessageSent**：最近发送过的消息，为每个`sid`保留最近发送过的一个消息。

#### 建立连接

为了能够相互投票，Zookeeper集群中的所有机器都需要两两建立起网络连接。QuorumCnxManager在启动时会创建一个ServerSocket来监听Leader选举的通信端口(默认为3888)。开启监听后，Zookeeper能够不断地接收到来自其他服务器的创建连接请求，在接收到其他服务器的TCP连接请求时，会进行处理。为了避免两台机器之间重复地创建TCP连接，Zookeeper只允许`sid`大的服务器主动和其他机器建立连接，否则断开连接。在接收到创建连接请求后，服务器通过对比自己和远程服务器的`sid`值来判断是否接收连接请求，如果当前服务器发现自己的`sid`更大，那么会断开当前连接，然后自己主动和远程服务器建立连接。一旦连接建立，就会根据远程服务器的SID来创建相应的消息发送器SendWorker和消息接收器RecvWorker，并启动。

#### 消息接收与发送

**消息接收**：由消息接收器RecvWorker负责，由于Zookeeper为每个远程服务器都分配一个单独的RecvWorker，因此，每个RecvWorker只需要不断地从这个TCP连接中读取消息，并将其保存到recvQueue队列中。

**消息发送**：由于Zookeeper为每个远程服务器都分配一个单独的SendWorker，因此，每个SendWorker只需要不断地从对应的消息发送队列中获取出一个消息发送即可，同时将这个消息放入lastMessageSent中。在SendWorker中，一旦Zookeeper发现针对当前服务器的消息发送队列为空，那么此时需要从lastMessageSent中取出一个最近发送过的消息来进行再次发送，这是为了解决接收方在消息接收前或者接收到消息后服务器挂了，导致消息尚未被正确处理。同时，Zookeeper能够保证接收方在处理消息时，会对重复消息进行正确的处理。

### FastLeaderElection：选举算法核心

#### 选票管理

- **sendqueue**：选票发送队列，用于保存待发送的选票。
- **recvqueue**：选票接收队列，用于保存接收到的外部投票。
- **WorkerReceiver**：选票接收器。其会不断地从QuorumCnxManager中获取其他服务器发来的选举消息，并将其转换成一个选票，然后保存到recvqueue中，在选票接收过程中，如果发现该外部选票的选举轮次小于当前服务器的，那么忽略该外部投票，同时立即发送自己的内部投票。
- **WorkerSender**：选票发送器，不断地从sendqueue中获取待发送的选票，并将其传递到底层QuorumCnxManager中。

#### 算法核心

![FastLeaderElection模块是如何与底层网络I/O进行交互](616953-20161206114702772-1120304539.png)

上图展示了FastLeaderElection模块是如何与底层网络I/O进行交互的。Leader选举的基本流程如下: 

1. **自增选举轮次**。Zookeeper规定所有有效的投票都必须在同一轮次中，在开始新一轮投票时，会首先对logicalclock进行自增操作。

2. **初始化选票**。在开始进行新一轮投票之前，每个服务器都会初始化自身的选票，并且在初始化阶段，每台服务器都会将自己推举为Leader。

3. **发送初始化选票**。完成选票的初始化后，服务器就会发起第一次投票。Zookeeper会将刚刚初始化好的选票放入sendqueue中，由发送器WorkerSender负责发送出去。

4. **接收外部投票**。每台服务器会不断地从recvqueue队列中获取外部选票。如果服务器发现无法获取到任何外部投票，那么就会立即确认自己是否和集群中其他服务器保持着有效的连接，如果没有连接，则马上建立连接，如果已经建立了连接，则再次发送自己当前的内部投票。

5. **判断选举轮次**。在发送完初始化选票之后，接着开始处理外部投票。在处理外部投票时，会根据选举轮次来进行不同的处理。
   - **外部投票的选举轮次大于内部投票**。若服务器自身的选举轮次落后于该外部投票对应服务器的选举轮次，那么就会立即更新自己的选举轮次(logicalclock)，并且清空所有已经收到的投票，然后使用初始化的投票来进行PK以确定是否变更内部投票。最终再将内部投票发送出去。
   - **外部投票的选举轮次小于内部投票**。若服务器接收的外选票的选举轮次落后于自身的选举轮次，那么Zookeeper就会直接忽略该外部投票，不做任何处理，并返回步骤4。
   - **外部投票的选举轮次等于内部投票**。此时可以开始进行选票PK。

6. **选票PK**。在进行选票PK时，符合任意一个条件就需要变更投票。
   - 若外部投票中推举的Leader服务器的选举轮次大于内部投票，那么需要变更投票。
   - 若选举轮次一致，那么就对比两者的`zxid`，若外部投票的`zxid`大，那么需要变更投票。
   - 若两者的ZXID一致，那么就对比两者的`sid`，若外部投票的`sid`大，那么就需要变更投票。

7. **变更投票**。经过PK后，若确定了外部投票优于内部投票，那么就变更投票，即使用外部投票的选票信息来覆盖内部投票，变更完成后，再次将这个变更后的内部投票发送出去。

8. **选票归档**。无论是否变更了投票，都会将刚刚收到的那份外部投票放入选票集合recvset中进行归档。recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票（按照服务队的SID区别，如{(1, vote1), (2, vote2)...}）。

9. **统计投票**。完成选票归档后，就可以开始统计投票，统计投票是为了统计集群中是否已经有过半的服务器认可了当前的内部投票，如果确定已经有过半服务器认可了该投票，则终止投票。否则返回步骤4。

10. **更新服务器状态**。若已经确定可以终止投票，那么就开始更新服务器状态，服务器首选判断当前被过半服务器认可的投票所对应的Leader服务器是否是自己，若是自己，则将自己的服务器状态更新为LEADING，若不是，则根据具体情况来确定自己是FOLLOWING或是OBSERVING。

　　以上10个步骤就是FastLeaderElection的核心，其中步骤4-9会经过几轮循环，直到有Leader选举产生。



## FastLeaderElection源码分析

### 类的继承关系　

```java
public class FastLeaderElection implements Election {}
```

　　说明：FastLeaderElection实现了Election接口，其需要实现接口中定义的lookForLeader方法和shutdown方法，其是标准的Fast Paxos算法的实现，各服务器之间基于TCP协议进行选举。

### 类的内部类

FastLeaderElection有三个较为重要的内部类，分别为Notification、ToSend、Messenger。

#### Notification类

Notification表示收到的选举投票信息（其他服务器发来的选举投票信息），其包含了被选举者的id、zxid、选举周期等信息，其buildMsg方法将选举信息封装至ByteBuffer中再进行发送。

```java
/**
 * Notifications are messages that let other peers know that
 * a given peer has changed its vote, either because it has
 * joined leader election or because it learned of another
 * peer with higher zxid or same zxid and higher server id
 */

static public class Notification {
    /*
     * Format version, introduced in 3.4.6
     */

    public final static int CURRENTVERSION = 0x2;
    int version;

    /*
     * Proposed leader
     */
    long leader;

    /*
     * zxid of the proposed leader
     */
    long zxid;

    /*
     * Epoch
     */
    long electionEpoch;

    /*
     * current state of sender
     */
    QuorumPeer.ServerState state;

    /*
     * Address of sender
     */
    long sid;
    
    QuorumVerifier qv;
    /*Notification
     * epoch of the proposed leader
     */
    long peerEpoch;
}
```

#### ToSend类

ToSend表示发送给其他服务器的选举投票信息，也包含了被选举者的id、zxid、选举周期等信息。

```java
/**
 * Messages that a peer wants to send to other peers.
 * These messages can be both Notifications and Acks
 * of reception of notification.
 */
static public class ToSend {
    static enum mType {crequest, challenge, notification, ack}

    ToSend(mType type,
            long leader,
            long zxid,
            long electionEpoch,
            ServerState state,
            long sid,
            long peerEpoch,
            byte[] configData) {

        this.leader = leader;
        this.zxid = zxid;
        this.electionEpoch = electionEpoch;
        this.state = state;
        this.sid = sid;
        this.peerEpoch = peerEpoch;
        this.configData = configData;
    }

    /*
     * Proposed leader in the case of notification
     */
    long leader;

    /*
     * id contains the tag for acks, and zxid for notifications
     */
    long zxid;

    /*
     * Epoch
     */
    long electionEpoch;

    /*
     * Current state;
     */
    QuorumPeer.ServerState state;

    /*
     * Address of recipient
     */
    long sid;

    /*
     * Used to send a QuorumVerifier (configuration info)
     */
    byte[] configData = dummyData;

    /*
     * Leader epoch
     */
    long peerEpoch;
}
```

#### Messenger类

Messenger包含了WorkerReceiver和WorkerSender两个内部类:

```java
protected class Messenger {
        // 选票发送器
        WorkerSender ws;
        // 选票接收器
        WorkerReceiver wr;
    }
```

构造函数:

```java
Messenger(QuorumCnxManager manager) {
		// 创建WorkerSender
    this.ws = new WorkerSender(manager);

    this.wsThread = new Thread(this.ws,
            "WorkerSender[myid=" + self.getId() + "]");
    this.wsThread.setDaemon(true);

	  // 创建WorkerReceiver
    this.wr = new WorkerReceiver(manager);

    this.wrThread = new Thread(this.wr,
            "WorkerReceiver[myid=" + self.getId() + "]");
    this.wrThread.setDaemon(true);
}
```

##### WorkerReceiver

```java
/**
 * Receives messages from instance of QuorumCnxManager on
 * method run(), and processes such messages.
 */

class WorkerReceiver extends ZooKeeperThread  {
    volatile boolean stop;
  	// 服务器之间的连接
    QuorumCnxManager manager;

    WorkerReceiver(QuorumCnxManager manager) {
        super("WorkerReceiver");
        this.stop = false;
        this.manager = manager;
    }

    public void run() {

        Message response;
        while (!stop) {
            // Sleeps on receive
            try {
              	// 从recvQueue中取出一个选举投票消息（从其他服务器发送过来）
                response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
              	// 无投票，跳过
                if(response == null) continue;

                // 大小校验
                ... 
                
                response.buffer.clear();

                // Instantiate Notification and set its attributes
	              // 创建接收通知
                Notification n = new Notification();

                int rstate = response.buffer.getInt();
                long rleader = response.buffer.getLong();
                long rzxid = response.buffer.getLong();
                long relectionEpoch = response.buffer.getLong();
                long rpeerepoch;

                int version = 0x0;
                if (!backCompatibility28) {
                    rpeerepoch = response.buffer.getLong();
                    if (!backCompatibility40) {
                        /*
                         * Version added in 3.4.6
                         */
                        
                        version = response.buffer.getInt();
                    } else {
                        LOG.info("Backward compatibility mode (36 bits), server id: {}", response.sid);
                    }
                } else {
                    LOG.info("Backward compatibility mode (28 bits), server id: {}", response.sid);
                    rpeerepoch = ZxidUtils.getEpochFromZxid(rzxid);
                }

                QuorumVerifier rqv = null;

                // check if we have a version that includes config. If so extract config info from message.
                if (version > 0x1) {
                    int configLength = response.buffer.getInt();
                    byte b[] = new byte[configLength];

                    response.buffer.get(b);
                                               
                    synchronized(self) {
                        try {
                            rqv = self.configFromString(new String(b));
                            QuorumVerifier curQV = self.getQuorumVerifier();
                            if (rqv.getVersion() > curQV.getVersion()) {
                                LOG.info("{} Received version: {} my version: {}", self.getId(),
                                        Long.toHexString(rqv.getVersion()),
                                        Long.toHexString(self.getQuorumVerifier().getVersion()));
                                if (self.getPeerState() == ServerState.LOOKING) {
                                    LOG.debug("Invoking processReconfig(), state: {}", self.getServerState());
                                    self.processReconfig(rqv, null, null, false);
                                    if (!rqv.equals(curQV)) {
                                        LOG.info("restarting leader election");
                                        self.shuttingDownLE = true;
                                        self.getElectionAlg().shutdown();

                                        break;
                                    }
                                } else {
                                    LOG.debug("Skip processReconfig(), state: {}", self.getServerState());
                                }
                            }
                        } catch (IOException e) {
                            LOG.error("Something went wrong while processing config received from {}", response.sid);
                       } catch (ConfigException e) {
                           LOG.error("Something went wrong while processing config received from {}", response.sid);
                       }
                    }                          
                } else {
                    LOG.info("Backward compatibility mode (before reconfig), server id: {}", response.sid);
                }
               
                /*
                 * If it is from a non-voting server (such as an observer or
                 * a non-voting follower), respond right away.
                 */
                if(!validVoter(response.sid)) {
                    Vote current = self.getCurrentVote();
                    QuorumVerifier qv = self.getQuorumVerifier();
                    ToSend notmsg = new ToSend(ToSend.mType.notification,
                            current.getId(),
                            current.getZxid(),
                            logicalclock.get(),
                            self.getPeerState(),
                            response.sid,
                            current.getPeerEpoch(),
                            qv.toString().getBytes());

                    sendqueue.offer(notmsg);
                } else {
                    // Receive new message
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("Receive new notification message. My id = "
                                + self.getId());
                    }

                    // State of peer that sent this message
                  	// 推选者的状态
                    QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                    switch (rstate) {
                    case 0:
                        ackstate = QuorumPeer.ServerState.LOOKING;
                        break;
                    case 1:
                        ackstate = QuorumPeer.ServerState.FOLLOWING;
                        break;
                    case 2:
                        ackstate = QuorumPeer.ServerState.LEADING;
                        break;
                    case 3:
                        ackstate = QuorumPeer.ServerState.OBSERVING;
                        break;
                    default:
                        continue;
                    }

                    n.leader = rleader;
                    n.zxid = rzxid;
                    n.electionEpoch = relectionEpoch;
                    n.state = ackstate;
                    n.sid = response.sid;
                    n.peerEpoch = rpeerepoch;
                    n.version = version;
                    n.qv = rqv;
                    /*
                     * Print notification info
                     */
                    if(LOG.isInfoEnabled()){
                        printNotification(n);
                    }

                    /*
                     * If this server is looking, then send proposed leader
                     */

                    if(self.getPeerState() == QuorumPeer.ServerState.LOOKING){
                        recvqueue.offer(n);

                        /*
                         * Send a notification back if the peer that sent this
                         * message is also looking and its logical clock is
                         * lagging behind.
                         */
                        if((ackstate == QuorumPeer.ServerState.LOOKING)
                                && (n.electionEpoch < logicalclock.get())){
                            // 获取自己的投票
                          	Vote v = getVote();
                            QuorumVerifier qv = self.getQuorumVerifier();
                            // 构造ToSend消息
                          	ToSend notmsg = new ToSend(ToSend.mType.notification,
                                    v.getId(),
                                    v.getZxid(),
                                    logicalclock.get(),
                                    self.getPeerState(),
                                    response.sid,
                                    v.getPeerEpoch(),
                                    qv.toString().getBytes());
                          	// 放入sendqueue队列，等待发送
                            sendqueue.offer(notmsg);
                        }
                    } else {
                        /*
                         * If this server is not looking, but the one that sent the ack
                         * is looking, then send back what it believes to be the leader.
                         */
                        Vote current = self.getCurrentVote();
                        if(ackstate == QuorumPeer.ServerState.LOOKING){
                            if(LOG.isDebugEnabled()){
                                LOG.debug("Sending new notification. My id ={} recipient={} zxid=0x{} leader={} config version = {}",
                                        self.getId(),
                                        response.sid,
                                        Long.toHexString(current.getZxid()),
                                        current.getId(),
                                        Long.toHexString(self.getQuorumVerifier().getVersion()));
                            }

                            QuorumVerifier qv = self.getQuorumVerifier();
                            ToSend notmsg = new ToSend(
                                    ToSend.mType.notification,
                                    current.getId(),
                                    current.getZxid(),
                                    current.getElectionEpoch(),
                                    self.getPeerState(),
                                    response.sid,
                                    current.getPeerEpoch(),
                                    qv.toString().getBytes());
                            sendqueue.offer(notmsg);
                        }
                    }
                }
            } catch (InterruptedException e) {
                LOG.warn("Interrupted Exception while waiting for new message" +
                        e.toString());
            }
        }
        LOG.info("WorkerReceiver is down");
    }
}
```

WorkerReceiver实现了Runnable接口，是选票接收器。其会不断地从QuorumCnxManager中获取其他服务器发来的选举消息，并将其转换成一个选票，然后保存到recvqueue中，在选票接收过程中，如果发现该外部选票的选举轮次小于当前服务器的，那么忽略该外部投票，同时立即发送自己的内部投票。其是将QuorumCnxManager的Message转化为FastLeaderElection的Notification。

其中，WorkerReceiver的主要逻辑在run方法中，其首先会从QuorumCnxManager中的recvQueue队列中取出其他服务器发来的选举消息，消息封装在Message数据结构中。然后判断消息中的服务器id是否包含在可以投票的服务器集合中，若不是，则会将本服务器的内部投票发送给该服务器，其流程如下: 

```java
/*
 * If it is from a non-voting server (such as an observer or
 * a non-voting follower), respond right away.
 */
if(!validVoter(response.sid)) {
  	// 获取自己的投票
    Vote current = self.getCurrentVote();
    QuorumVerifier qv = self.getQuorumVerifier();
  	// 构造ToSend消息
    ToSend notmsg = new ToSend(ToSend.mType.notification,
            current.getId(),
            current.getZxid(),
            logicalclock.get(),
            self.getPeerState(),
            response.sid,
            current.getPeerEpoch(),
            qv.toString().getBytes());

  	// 放入sendqueue队列，等待发送
    sendqueue.offer(notmsg);
} 
```

若包含该服务器，则根据消息（Message）解析出投票服务器的投票信息并将其封装为Notification，然后判断当前服务器是否为LOOKING，若为LOOKING，则直接将Notification放入FastLeaderElection的recvqueue（区别于recvQueue）中。然后判断投票服务器是否为LOOKING状态，并且其选举周期小于当前服务器的逻辑时钟，则将本（当前）服务器的内部投票发送给该服务器，否则，直接忽略掉该投票。其流程如下: 

```java
/*
 * If this server is looking, then send proposed leader
 */

if(self.getPeerState() == QuorumPeer.ServerState.LOOKING){  // 本服务器为LOOKING状态
    // 将消息放入recvqueue中
  	recvqueue.offer(n);

    /*
     * Send a notification back if the peer that sent this
     * message is also looking and its logical clock is
     * lagging behind.
     */
    if((ackstate == QuorumPeer.ServerState.LOOKING)	// 推选者服务器为LOOKING状态
            && (n.electionEpoch < logicalclock.get())){	// 选举周期小于逻辑时钟
        // 创建新的投票
      	Vote v = getVote();
        QuorumVerifier qv = self.getQuorumVerifier();
        // 构造新的发送消息（本服务器自己的投票）
      	ToSend notmsg = new ToSend(ToSend.mType.notification,
                v.getId(),
                v.getZxid(),
                logicalclock.get(),
                self.getPeerState(),
                response.sid,
                v.getPeerEpoch(),
                qv.toString().getBytes());
      	// 将发送消息放置于队列，等待发送
        sendqueue.offer(notmsg);
    }
} 
```

若本服务器的状态不为LOOKING，则会根据投票服务器中解析的version信息来构造ToSend消息，放入sendqueue，等待发送，起流程如下: 

```java
} else {	// 本服务器状态不为LOOKING
        /*
         * If this server is not looking, but the one that sent the ack
         * is looking, then send back what it believes to be the leader.
         */
		  	// 获取当前投票
        Vote current = self.getCurrentVote();
        if(ackstate == QuorumPeer.ServerState.LOOKING){	// 为LOOKING状态
            if(LOG.isDebugEnabled()){
                LOG.debug("Sending new notification. My id ={} recipient={} zxid=0x{} leader={} config version = {}",
                        self.getId(),
                        response.sid,
                        Long.toHexString(current.getZxid()),
                        current.getId(),
                        Long.toHexString(self.getQuorumVerifier().getVersion()));
            }

            QuorumVerifier qv = self.getQuorumVerifier();
          	// 构造ToSend消息
            ToSend notmsg = new ToSend(
                    ToSend.mType.notification,
                    current.getId(),
                    current.getZxid(),
                    current.getElectionEpoch(),
                    self.getPeerState(),
                    response.sid,
                    current.getPeerEpoch(),
                    qv.toString().getBytes());
          	// 将发送消息放置于队列，等待发送
            sendqueue.offer(notmsg);
        }
    }
}
```

##### WorkerSender

```java
/**
 * This worker simply dequeues a message to send and
 * and queues it on the manager's queue.
 */

class WorkerSender extends ZooKeeperThread {
    volatile boolean stop;
    QuorumCnxManager manager;

    WorkerSender(QuorumCnxManager manager){
        super("WorkerSender");
        this.stop = false;
        this.manager = manager;
    }

    public void run() {
        while (!stop) {
            try {
                ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
                if(m == null) continue;

                process(m);
            } catch (InterruptedException e) {
                break;
            }
        }
        LOG.info("WorkerSender is down");
    }

    /**
     * Called by run() once there is a new message to send.
     *
     * @param m     message to send
     */
    void process(ToSend m) {
        ByteBuffer requestBuffer = buildMsg(m.state.ordinal(),
                                            m.leader,
                                            m.zxid,
                                            m.electionEpoch,
                                            m.peerEpoch,
                                            m.configData);

        manager.toSend(m.sid, requestBuffer);

    }
}
```

WorkerSender也实现了Runnable接口，为选票发送器，其会不断地从sendqueue中获取待发送的选票，并将其传递到底层QuorumCnxManager中，其过程是将FastLeaderElection的ToSend转化为QuorumCnxManager的Message。

### 类的构造函数　

```java
public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
    this.stop = false;
    this.manager = manager;
    starter(self, manager);
}
```

构造函数中初始化了stop字段和manager字段，并且调用了starter函数: 

```java
private void starter(QuorumPeer self, QuorumCnxManager manager) {
    this.self = self;
    proposedLeader = -1;
    proposedZxid = -1;

    sendqueue = new LinkedBlockingQueue<ToSend>();
    recvqueue = new LinkedBlockingQueue<Notification>();
    this.messenger = new Messenger(manager);
}
```

### 核心函数分析

#### sendNotifications函数　

```java
private void sendNotifications() {
    for (long sid : self.getCurrentAndNextConfigVoters()) {   // 遍历投票参与者集合
        QuorumVerifier qv = self.getQuorumVerifier();
       
	      // 构造发送消息
      	ToSend notmsg = new ToSend(ToSend.mType.notification,
                proposedLeader,
                proposedZxid,
                logicalclock.get(),
                QuorumPeer.ServerState.LOOKING,
                sid,
                proposedEpoch, qv.toString().getBytes());
        if(LOG.isDebugEnabled()){
            LOG.debug("Sending Notification: " + proposedLeader + " (n.leader), 0x"  +
                  Long.toHexString(proposedZxid) + " (n.zxid), 0x" + Long.toHexString(logicalclock.get())  +
                  " (n.round), " + sid + " (recipient), " + self.getId() +
                  " (myid), 0x" + Long.toHexString(proposedEpoch) + " (n.peerEpoch)");
        }
      
	      // 将发送消息放置于队列
        sendqueue.offer(notmsg);
    }
}
```

其会遍历所有的参与者投票集合，然后将自己的选票信息发送至上述所有的投票者集合，其并非同步发送，而是将ToSend消息放置于sendqueue中，之后由WorkerSender进行发送。

#### totalOrderPredicate函数

```java
/**
 * Check if a pair (server id, zxid) succeeds our
 * current vote.
 *
 * @param id    Server identifier
 * @param zxid  Last zxid observed by the issuer of this vote
 */
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    LOG.debug("id: " + newId + ", proposed id: " + curId + ", zxid: 0x" +
            Long.toHexString(newZxid) + ", proposed zxid: 0x" + Long.toHexString(curZxid));
    if(self.getQuorumVerifier().getWeight(newId) == 0){
        return false;
    }

    /*
     * We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same
     *  as current zxid, but server id is higher.
     */

    return ((newEpoch > curEpoch) ||
            ((newEpoch == curEpoch) &&
            ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
}
```

该函数将接收的投票与自身投票进行PK，查看是否消息中包含的服务器id是否更优，其按照epoch、zxid、id的优先级进行PK。

#### termPredicate函数

```java
/**
 * Termination predicate. Given a set of votes, determines if have
 * sufficient to declare the end of the election round.
 * 
 * @param votes
 *            Set of votes
 * @param vote
 *            Identifier of the vote received last
 */
protected boolean termPredicate(Map<Long, Vote> votes, Vote vote) {
    SyncedLearnerTracker voteSet = new SyncedLearnerTracker();
    voteSet.addQuorumVerifier(self.getQuorumVerifier());
    if (self.getLastSeenQuorumVerifier() != null
            && self.getLastSeenQuorumVerifier().getVersion() > self
                    .getQuorumVerifier().getVersion()) {
        voteSet.addQuorumVerifier(self.getLastSeenQuorumVerifier());
    }

    /*
     * First make the views consistent. Sometimes peers will have different
     * zxids for a server depending on timing.
     */
    for (Map.Entry<Long, Vote> entry : votes.entrySet()) { // 遍历已经接收的投票集合
        if (vote.equals(entry.getValue())) { // 将等于当前投票的项放入set
            voteSet.addAck(entry.getKey());
        }
    }

  	//统计set，查看投某个id的票数是否超过一半
    return voteSet.hasAllQuorums();
}
```

该函数用于判断Leader选举是否结束，即是否有一半以上的服务器选出了相同的Leader，其过程是将收到的选票与当前选票进行对比，选票相同的放入同一个集合，之后判断选票相同的集合是否超过了半数。

#### checkLeader函数　　

```java
/**
 * In the case there is a leader elected, and a quorum supporting
 * this leader, we have to check if the leader has voted and acked
 * that it is leading. We need this check to avoid that peers keep
 * electing over and over a peer that has crashed and it is no
 * longer leading.
 *
 * @param votes set of votes
 * @param   leader  leader id
 * @param   electionEpoch   epoch id
 */
protected boolean checkLeader(
        Map<Long, Vote> votes,
        long leader,
        long electionEpoch){

    boolean predicate = true;

    /*
     * If everyone else thinks I'm the leader, I must be the leader.
     * The other two checks are just for the case in which I'm not the
     * leader. If I'm not the leader and I haven't received a message
     * from leader stating that it is leading, then predicate is false.
     */

    if(leader != self.getId()){  // 自己不为leader
        if(votes.get(leader) == null) predicate = false;  // 还未选出leader
        else if(votes.get(leader).getState() != ServerState.LEADING) predicate = false; // 选出的leader还未给出ack信号，其他服务器还不知道leader
    } else if(logicalclock.get() != electionEpoch) {  // 逻辑时钟不等于选举周期
        predicate = false;
    }

    return predicate;
}
```

该函数检查是否已经完成了Leader的选举，此时Leader的状态应该是LEADING状态。

#### lookForLeader函数

```java
/**
 * Starts a new round of leader election. Whenever our QuorumPeer
 * changes its state to LOOKING, this method is invoked, and it
 * sends notifications to all other peers.
 */
public Vote lookForLeader() throws InterruptedException {
    // 注册jmx
    ...
      
    try {
        HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

        HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = finalizeWait;

        synchronized(this){
	          // 更新逻辑时钟，每进行一轮选举，都需要更新逻辑时钟
            logicalclock.incrementAndGet();
	          // 更新选票
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }

        LOG.info("New election. My id =  " + self.getId() +
                ", proposed zxid=0x" + Long.toHexString(proposedZxid));
        // 向其他服务器发送自己的选票
      	sendNotifications();

        /*
         * Loop in which we exchange notifications until we find a leader
         */

        while ((self.getPeerState() == ServerState.LOOKING) &&
                (!stop)){  // 本服务器状态为LOOKING并且还未选出leader
     
            // 从recvqueue接收队列中取出投票
            Notification n = recvqueue.poll(notTimeout,
                    TimeUnit.MILLISECONDS);

            /*
             * Sends more notifications if haven't received enough.
             * Otherwise processes new notification.
             */
            if(n == null){
                if(manager.haveDelivered()){
                    sendNotifications();
                } else {
                    manager.connectAll();
                }

                /*
                 * Exponential backoff
                 */
                int tmpTimeOut = notTimeout*2;
                notTimeout = (tmpTimeOut < maxNotificationInterval?
                        tmpTimeOut : maxNotificationInterval);
                LOG.info("Notification time out: " + notTimeout);
            } 
            else if (validVoter(n.sid) && validVoter(n.leader)) { // 投票者集合中包含接收到消息中的服务器id
                /*
                 * Only proceed if the vote comes from a replica in the current or next
                 * voting view for a replica in the current or next voting view.
                 */
                switch (n.state) {
                case LOOKING:
                    // If notification > current, replace and send messages out
                    if (n.electionEpoch > logicalclock.get()) {  // 其选举周期大于逻辑时钟
                        // 重新赋值逻辑时钟
                        logicalclock.set(n.electionEpoch);
                        recvset.clear();
                        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) { // 选出较优的服务器
                          	// 更新选票  
                          	updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else { // 无法选出较优的服务器
                            updateProposal(getInitId(),
                                    getInitLastLoggedZxid(),
                                    getPeerEpoch());
                        }
                        sendNotifications();
                    } else if (n.electionEpoch < logicalclock.get()) { // 选举周期小于逻辑时钟，不做处理
                        if(LOG.isDebugEnabled()){
                            LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                                    + Long.toHexString(n.electionEpoch)
                                    + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
                        }
                        break;
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                            proposedLeader, proposedZxid, proposedEpoch)) { // 等于，并且能选出较优的服务器
                        // 更新选票
                      	updateProposal(n.leader, n.zxid, n.peerEpoch);
                      	// 发送消息
                        sendNotifications();
                    }

                    if(LOG.isDebugEnabled()){
                        LOG.debug("Adding vote: from=" + n.sid +
                                ", proposed leader=" + n.leader +
                                ", proposed zxid=0x" + Long.toHexString(n.zxid) +
                                ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
                    }

                    // don't care about the version if it's in LOOKING state
                    // recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                    if (termPredicate(recvset,
                            new Vote(proposedLeader, proposedZxid,
                                    logicalclock.get(), proposedEpoch))) { // 若能选出leader


                        // Verify if there is any change in the proposed leader
                        while((n = recvqueue.poll(finalizeWait,
                                TimeUnit.MILLISECONDS)) != null){
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    proposedLeader, proposedZxid, proposedEpoch)){
                                recvqueue.put(n);
                                break;
                            }
                        }

                        /*
                         * This predicate is true once we don't read any new
                         * relevant message from the reception queue
                         */
                        if (n == null) {
                            self.setPeerState((proposedLeader == self.getId()) ?
                                    ServerState.LEADING: learningState());
                            Vote endVote = new Vote(proposedLeader,
                                    proposedZxid, logicalclock.get(), 
                                    proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    LOG.debug("Notification from observer: " + n.sid);
                    break;
                case FOLLOWING:
                case LEADING:
                    /*
                     * Consider all notifications from the same epoch
                     * together.
                     */
                    if(n.electionEpoch == logicalclock.get()){ // 与逻辑时钟相等
                      	// 将该服务器和选票信息放入recvset中
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        if(termPredicate(recvset, new Vote(n.version, n.leader,
                                        n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                                        && checkLeader(outofelection, n.leader, n.electionEpoch)) { // 判断是否完成了leader选举
                            // 设置本服务器的状态
                          	self.setPeerState((n.leader == self.getId()) ?
                                    ServerState.LEADING: learningState());
                            // 创建投票信息
                          	Vote endVote = new Vote(n.leader, 
                                    n.zxid, n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }

                    /*
                     * Before joining an established ensemble, verify that
                     * a majority are following the same leader.
                     */
                    outofelection.put(n.sid, new Vote(n.version, n.leader, 
                            n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                    if (termPredicate(outofelection, new Vote(n.version, n.leader,
                            n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                            && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                        synchronized(this){
                            logicalclock.set(n.electionEpoch);
                            self.setPeerState((n.leader == self.getId()) ?
                                    ServerState.LEADING: learningState());
                        }
                        Vote endVote = new Vote(n.leader, n.zxid, 
                                n.electionEpoch, n.peerEpoch);
                        leaveInstance(endVote);
                        return endVote;
                    }
                    break;
                default:
                    LOG.warn("Notification state unrecoginized: " + n.state
                          + " (n.state), " + n.sid + " (n.sid)");
                    break;
                }
            } else {
                if (!validVoter(n.leader)) {
                    LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                }
                if (!validVoter(n.sid)) {
                    LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                }
            }
        }
        return null;
    } finally {
      // 卸载注册的jmx
    	...
    }
}
```

该函数用于开始新一轮的Leader选举，其首先会将逻辑时钟自增，然后更新本服务器的选票信息（初始化选票），之后将选票信息放入sendqueue等待发送给其他服务器，其流程如下: 

```java
synchronized(this){
    logicalclock.incrementAndGet();
    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
}

LOG.info("New election. My id =  " + self.getId() +
        ", proposed zxid=0x" + Long.toHexString(proposedZxid));
sendNotifications();
```

之后每台服务器会不断地从recvqueue队列中获取外部选票。如果服务器发现无法获取到任何外部投票，就立即确认自己是否和集群中其他服务器保持着有效的连接，如果没有连接，则马上建立连接，如果已经建立了连接，则再次发送自己当前的内部投票，其流程如下: 

```java
// 从recvqueue接收队列中取出投票
Notification n = recvqueue.poll(notTimeout,
        TimeUnit.MILLISECONDS);

/*
 * Sends more notifications if haven't received enough.
 * Otherwise processes new notification.
 */
if(n == null){ // 无法获取选票
    if(manager.haveDelivered()){ // manager已经发送了所有选票消息（表示有连接）
      // 向所有其他服务器发送消息  
      sendNotifications();
    } else { // 还未发送所有消息（表示无连接）
      	// 连接其他每个服务器
        manager.connectAll();
    }

    /*
     * Exponential backoff
     */
    int tmpTimeOut = notTimeout*2;
    notTimeout = (tmpTimeOut < maxNotificationInterval?
            tmpTimeOut : maxNotificationInterval);
    LOG.info("Notification time out: " + notTimeout);
} 
```

在发送完初始化选票之后，接着开始处理外部投票。在处理外部投票时，会根据选举轮次来进行不同的处理。　　

- **外部投票的选举轮次大于内部投票**。若服务器自身的选举轮次落后于该外部投票对应服务器的选举轮次，那么就会立即更新自己的选举轮次(logicalclock)，并且清空所有已经收到的投票，然后使用初始化的投票来进行PK以确定是否变更内部投票。最终再将内部投票发送出去。
- **外部投票的选举轮次小于内部投票**。若服务器接收的外选票的选举轮次落后于自身的选举轮次，那么Zookeeper就会直接忽略该外部投票，不做任何处理。
- **外部投票的选举轮次等于内部投票**。此时可以开始进行选票PK，如果消息中的选票更优，则需要更新本服务器内部选票，再发送给其他服务器。

之后再对选票进行归档操作，无论是否变更了投票，都会将刚刚收到的那份外部投票放入选票集合recvset中进行归档，其中recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票，然后开始统计投票，统计投票是为了统计集群中是否已经有过半的服务器认可了当前的内部投票，如果确定已经有过半服务器认可了该投票，然后再进行最后一次确认，判断是否又有更优的选票产生，若无，则终止投票，然后最终的选票，其流程如下: 

```java
// If notification > current, replace and send messages out
if (n.electionEpoch > logicalclock.get()) {  // 其选举周期大于逻辑时钟
    // 重新赋值逻辑时钟
  	logicalclock.set(n.electionEpoch);
    // 清空所有接收到的所有选票
  	recvset.clear();
    if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
            getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {  // 进行PK，选出较优的服务
        // 更新选票
      	updateProposal(n.leader, n.zxid, n.peerEpoch);
    } else {  // 无法选出较优的服务器
        // 更新选票
      	updateProposal(getInitId(),
                getInitLastLoggedZxid(),
                getPeerEpoch());
    }
  	// 发送本服务器的内部选票消息
    sendNotifications();
} else if (n.electionEpoch < logicalclock.get()) {	// 选举周期小于逻辑时钟，不做处理，直接忽略
    if(LOG.isDebugEnabled()){
        LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                + Long.toHexString(n.electionEpoch)
                + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
    }
    break;
} else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
        proposedLeader, proposedZxid, proposedEpoch)) { // PK，选出较优的服务器
    // 更新选票
  	updateProposal(n.leader, n.zxid, n.peerEpoch);
    // 发送消息
  	sendNotifications();
}

if(LOG.isDebugEnabled()){
    LOG.debug("Adding vote: from=" + n.sid +
            ", proposed leader=" + n.leader +
            ", proposed zxid=0x" + Long.toHexString(n.zxid) +
            ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
}

// don't care about the version if it's in LOOKING state
// recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票
recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

if (termPredicate(recvset,
        new Vote(proposedLeader, proposedZxid,
                logicalclock.get(), proposedEpoch))) { // 若能选出leader

    // Verify if there is any change in the proposed leader
    while((n = recvqueue.poll(finalizeWait,
            TimeUnit.MILLISECONDS)) != null){ // 遍历已经接收的投票集合
        if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                proposedLeader, proposedZxid, proposedEpoch)){ // 选票有变更，比之前提议的Leader有更好的选票加入
          	// 将更优的选票放在recvset中
            recvqueue.put(n);
            break;
        }
    }

    /*
     * This predicate is true once we don't read any new
     * relevant message from the reception queue
     */
    if (n == null) { // 表示之前提议的Leader已经是最优的
      	// 设置服务器状态
        self.setPeerState((proposedLeader == self.getId()) ?
                ServerState.LEADING: learningState());
      	// 最终的选票
        Vote endVote = new Vote(proposedLeader,
                proposedZxid, logicalclock.get(), 
                proposedEpoch);
      	// 清空recvqueue队列的选票
        leaveInstance(endVote);
      	// 返回选票
        return endVote;
    }
}
```

若选票中的服务器状态为FOLLOWING或者LEADING时，其大致步骤会判断选举周期是否等于逻辑时钟，归档选票，是否已经完成了Leader选举，设置服务器状态，修改逻辑时钟等于选举周期，返回最终选票，其流程如下: 

```java
if(n.electionEpoch == logicalclock.get()){  // 与逻辑时钟相等
	  // 将该服务器和选票信息放入recvset中
    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
    if(termPredicate(recvset, new Vote(n.version, n.leader,
                    n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                    && checkLeader(outofelection, n.leader, n.electionEpoch)) {  // 已经完成了leader选举
      	// 设置本服务器的状态
        self.setPeerState((n.leader == self.getId()) ?
                ServerState.LEADING: learningState());
	      // 最终的选票
        Vote endVote = new Vote(n.leader, 
                n.zxid, n.electionEpoch, n.peerEpoch);
      	// 清空recvqueue队列的选票
        leaveInstance(endVote);
        return endVote;
    }
}

/*
 * Before joining an established ensemble, verify that
 * a majority are following the same leader.
 */
outofelection.put(n.sid, new Vote(n.version, n.leader, 
        n.zxid, n.electionEpoch, n.peerEpoch, n.state));
if (termPredicate(outofelection, new Vote(n.version, n.leader,
        n.zxid, n.electionEpoch, n.peerEpoch, n.state))
        && checkLeader(outofelection, n.leader, n.electionEpoch)) {	// 已经完成了leader选举
    synchronized(this){
      	// 设置逻辑时钟
        logicalclock.set(n.electionEpoch);
      	// 设置状态
        self.setPeerState((n.leader == self.getId()) ?
                ServerState.LEADING: learningState());
    }
  	// 最终选票
    Vote endVote = new Vote(n.leader, n.zxid, 
            n.electionEpoch, n.peerEpoch);
  	// 清空recvqueue队列的选票
    leaveInstance(endVote);
  	// 返回选票
    return endVote;
}
```

## 参考资料

- Zookeeper目录: https://www.cnblogs.com/leesf456/p/6239578.html
- Zookeeper源码分析目录: https://www.cnblogs.com/leesf456/p/6518040.html