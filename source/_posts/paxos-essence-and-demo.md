---
title: paxos精髓及示例
date: 2019-07-31 18:58:42
categories: 
- 分布式一致性
- paxos算法
tags:
- paxos
- 分布式一致性
---

## Paxos精髓

paxos协议用来解决的问题可以用一句话来简化: 

> proposer将发起提案（value）给所有accpetor，超过半数accpetor获得批准后，proposer将提案写入accpetor内，最终所有accpetor获得一致性的确定性取值，且后续不允许再修改。

协议分为两大阶段，每个阶段又分为a/b两小步骤：

### 准备阶段（占坑阶段）

#### Phase 1a: *Prepare*

Proposer选择一个提议编号n，向所有的Acceptor广播Prepare（n）请求。



#### Phase 1b: *Promise*

Acceptor接收到Prepare（n）请求，若提议编号n比之前接收的Prepare请求都要大，则承诺将不会接收提议编号比n小的提议，并且带上之前Accept的提议中编号小于n的最大的提议，否则不予理会。



### 接受阶段（提交阶段）

#### Phase 2a: *Accept*

​	1. 如果未超过半数accpetor响应，直接转为提议失败；

​	2. 如果超过多数Acceptor的承诺，又分为不同情况：

​			2.1 如果所有Acceptor都未接收过值（都为null），那么向所有的Acceptor发起自己的值和提议编号n，记住，一定是所有Acceptor都没接受过值；

​			2.2 如果有部分Acceptor接收过值，那么从所有接受过的值中选择对应的提议编号最大的作为提议的值，提议编号仍然为n。但此时Proposer就不能提议自己的值，只能信任Acceptor通过的值，维护一但获得确定性取值就不能更改原则；


#### Phase 2b: *Accepted*

Acceptor接收到提议后，如果该提议版本号不等于自身保存记录的版本号（第一阶段记录的），不接受该请求，相等则写入本地。



整个paxos协议过程看似复杂难懂，但只要把握和理解这两点就基本理解了paxos的精髓：

1. 理解第一阶段accpetor的处理流程：如果本地已经写入了，不再接受和同意后面的所有请求，并返回本地写入的值；如果本地未写入，则本地记录该请求的版本号，并不再接受其他版本号的请求，简单来说只信任最后一次提交的版本号的请求，使其他版本号写入失效；

2. 理解第二阶段proposer的处理流程：未超过半数accpetor响应，提议失败；超过半数的accpetor值都为空才提交自身要写入的值，否则选择非空值里版本号最大的值提交，最大的区别在于是提交的值是自身的还是使用以前提交的。



### Basic Paxos信息流协议过程举例

看这个最简单的例子：1个processor，3个Acceptor，无learner。

目标：proposer向3个aceptort 将name变量写为v1。

第一阶段A：proposer发起prepare（name，n1）,n1是递增提议版本号，发送给3个Acceptor，说，我现在要写name这个变量，我的版本号是n1
第一阶段B：Acceptor收到proposer的消息，比对自己内部保存的内容，发现之前name变量（null，null）没有被写入且未收到过提议，都返回给proposer，并在内部记录name这个变量，已经有proposer申请提议了，提议版本号是n1;
第二阶段A：proposer收到3个Acceptor的响应，响应内容都是：name变量现在还没有写入，你可以来写。proposer确认获得超过半数以上Acceptor同意，发起第二阶段写入操作：accept（v1,n1），告诉Acceptor我现在要把name变量协议v1,我的版本号是刚刚获得通过的n1;
第二阶段B：accpetor收到accept（v1,n1），比对自身的版本号是一致的，保存成功，并响应accepted（v1,n1）；
结果阶段：proposer收到3个accepted响应都成功，超过半数响应成功，到此name变量被确定为v1。


以上是正常的paxos协议提议确定流程，是不是很简单，很容易理解呢？

确定你理解了上面的例子再往后看。

这是最简单也最容易理解的例子，但真实情况远比这个复杂，还有以下问题：

如果其中的某个Acceptor没响应怎么处理？
如果只写成功了一个accpetor又怎么处理，写成功两个呢？
如果多个proposer并发写会导致accpetor写成不同值吗？
learner角色是做什么用？
为什么是超过半数同意？

#### paxos特殊情况下的处理

**第一种情况：Proposer提议正常，未超过accpetor失败情况**

问题：还是上面的例子，如果第二阶段B，只有2个accpetor响应接收提议成功，另外1个没有响应怎么处理呢？

处理：proposer发现只有2个成功，已经超过半数，那么还是认为提议成功，并把消息传递给learner，由learner角色将确定的提议通知给所有accpetor，最终使最后未响应的accpetor也同步更新，通过learner角色使所有Acceptor达到最终一致性。



**第二种情况：Proposer提议正常，但超过accpetor失败情况**

问题：假设有2个accpetor失败，又该如何处理呢？

处理：由于未达到超过半数同意条件，proposer要么直接提示失败，要么递增版本号重新发起提议，如果重新发起提议对于第一次写入成功的accpetor不会修改，另外两个accpetor会重新接受提议，达到最终成功。



**情况再复杂一点：还是一样有3个accpetor，但有两个proposer。**

**情况一：proposer1和proposer2串行执行**

proposer1和最开始情况一样，把name设置为v1，并接受提议。

proposer1提议结束后，proposer2发起提议流程：

第一阶段A：proposer1发起prepare（name，n2）

第一阶段B：Acceptor收到proposer的消息，发现内部name已经写入确定了，返回（name,v1,n1）

第二阶段A：proposer收到3个Acceptor的响应，发现超过半数都是v1，说明name已经确定为v1，接受这个值，不再发起提议操作。



**情况二：proposer1和proposer2交错执行**

proposer1提议accpetor1成功，但写入accpetor2和accpetor3时，发现版本号已经小于accpetor内部记录的版本号（保存了proposer2的版本号），直接返回失败。

proposer2写入accpetor2和accpetor3成功，写入accpetor1失败，但最终还是超过半数写入v2成功，name变量最终确定为v2；

proposer1递增版本号再重试发现超过半数为v2，接受name变量为v2，也不再写入v1。name最终确定还是为v2



**情况三：proposer1和proposer2第一次都只写成功1个Acceptor怎么办**

都只写成功一个，未超过半数，那么Proposer会递增版本号重新发起提议，这里需要分多种情况：

1. 3个Acceptor都响应提议，发现Acceptor1{v1,n1} ,Acceptor2{v2,n2},Acceptor{null,null}，Processor选择最大的{v2,n2}发起第二阶段，成功后name值为v2;
2. 2个Acceptor都响应提议，
   - 如果是Acceptor1{v1,n1} ,Acceptor2{v2,n2}，那么选择最大的{v2,n2}发起第二阶段，成功后name值为v2;
   - 如果是Acceptor1{v1,n1} ,Acceptor3{null,null}，那么选择最大的{v1,n1}发起第二阶段，成功后name值为v1;
   - 如果是Acceptor2{v2,n2} ,Acceptor3{null,null}，那么选择最大的{v2,n2}发起第二阶段，成功后name值为v2;
3. 只有1个Acceptor响应提议，未达到半数，放弃或者递增版本号重新发起提议
   **可以看到，都未达到半数时，最终值是不确定的！**



## 参考资料

1. paxos算法wiki: https://en.wikipedia.org/wiki/Paxos_(computer_science)#Basic_Paxos
2. 首先，推荐的是知行学社的《分布式系统与Paxos算法视频课程》： 视频讲的非常好，很适合入门，循序渐进慢慢推导，我自己看了不下5遍，视频讲解理解更深，推荐大家都看看。
3. 推荐刘杰的《分布式系统原理介绍》 ，里面有关于paxos的详细介绍，例子非常多，也有包括paxos协议的证明过程，大而全，质量相当高的一份学习资料！
4. 推荐的一份高质量ppt《可靠分布式系统基础 Paxos 的直观解释》: https://drmingdrmer.github.io/tech/distributed/2015/11/11/paxos-slide.html；
5. 技术类的东西怎么能只停留在看上面，肯定要看代码啊，推荐微信开源的phxpaxos：https://github.com/tencent-wechat/phxpaxos，结合代码对协议理解更深。
6. [《Paxos Made live》](http://research.google.com/archive/paxos_made_live.html)。这篇论文是 Google 发表的，讨论了 paxos 的工程实践，也就是 chubby 这个众所周知的分布式服务的实现，可以结合[《The Chubby lock service for loosely-coupled distributed systems》](https://research.google.com/archive/chubby.html) 一起看。实际应用中的难点，比如 master 租约实现、group membership 变化、Snapshot 加快复制和恢复以及实际应用中遇到的故障、测试等问题，特别是最后的测试部分。非常值得一读。[《The Chubby lock service for loosely-coupled distributed systems》](https://research.google.com/archive/chubby.html) 更多介绍了 Chubby 服务本身的设计决策，为什么是分布式锁服务，为什么是粗粒度的锁，为什么是目录文件模式，事件通知、多机房部署以及应用碰到的使用问题等等。
7. Paxos 我还着重推荐阅读微信后端团队写的[系列博客](https://zhuanlan.zhihu.com/p/21438357?refer=lynncui)，包括他们开源的 [phxpaxos](https://github.com/tencent-wechat/phxpaxos/) 实现，基本上将所有问题都讨论到了，并且通俗易懂。
8. 一致性方面另一块就是 Raft 算法，按照 Google Chubby 论文里的说法，

```
`Indeed, all working protocols for asynchronous consensus we have so far  encountered have Paxos at their core.`
```

但是 Raft 真的好理解多了，我读的是[《In Search of an Understandable Consensus Algorithm》](https://docs.google.com/viewer?url=https%3A%2F%2Fraft.github.io%2Fraft.pdf)，论文写到这么详细的步骤，你不想理解都难。毕竟 Raft 号称就是一个 Understandable Consensus Algorithm。无论从任何角度，都推荐阅读这一篇论文。

首先能理解 paxos 的一些难点，其次是了解 Raft 的实现，加深对 Etcd 等系统的理解。这篇论文还有一个 250 多页的加强版[《CONSENSUS: BRIDGING THEORY AND PRACTICE》](https://ramcloud.stanford.edu/~ongaro/thesis.pdf)，教你一行一行写出一个 Raft 实现，我还没有学习，有兴趣可以自行了解。Raft 通过明确引入 leader（其实 multi paxos 引申出来也有，但是没有这么明确的表述）来负责 client 交互和日志复制，将整个算法过程非常清晰地表达出来。Raft 的算法正确性的核心在于保证 Leader Completeness ，选举算法选出来的 leader 一定是包含了所有 committed entries 的，这是因为所有 committed entries 一定会在多数派至少一个成员里存在，所以设计的选举算法一定能选出来这么一个成员作为 leader。多数派 accept 应该说是一致性算法正确性的最重要的保证。

最后，我还读了[《Building Consistent Transactions with Inconsistent Replication》](https://syslab.cs.washington.edu/papers/tapir-tr14.pdf)，包括作者的[演讲](https://www.youtube.com/watch?v=yE3eMxYJDiE)，作者也开放了[源码](https://github.com/UWSysLab/tapir)。Google Spanner 基本是将 paxos 算法应用到了极致，但是毕竟不是所有公司都是这么财大气粗搞得起 TrueTime API，架得起全球机房，控制或者承受得了事务延时。这篇论文提出了另一个思路，论文介绍的算法分为两个层次： IR 和基于其他上的 TAPIR。前者就是 Inconsistent Replication，它将操作分为两类：

- inconsistent： 可以任意顺序执行，成功执行的操作将持久化，支持 failover。

- consensus：同样可以任意顺序执行并持久化 failover，但是会返回一个唯一的一致(consensus)结果。

  

---------------------
## Java简单实现Basic Paxos算法

```java
/**
 * Alipay.com Inc.
 * Copyright (c) 2004-2019 All Rights Reserved.
 */
package paxos;

import com.google.common.base.Charsets;
import org.apache.commons.lang3.StringUtils;
import com.google.common.hash.HashFunction;
import com.google.common.hash.Hashing;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Random;

/**
 * @author xbyan
 * @version $Id: PaxosStudy.java, v 0.1 2019-07-01 5:47 PM xbyan Exp $$
 */
public class PaxosStudy {
    private static final HashFunction HASH_FUNCTION = Hashing.murmur3_32();
    private static final Random       RAMDOM        = new Random();
    private static final String[]     PROPOSALS     = { "ProjectA", "ProjectB", "ProjectC" };

    public static void main(String[] args) throws InterruptedException {
        //新建server组
        List<Acceptor> acceptors = new ArrayList<>();
        Arrays.asList("A", "B", "C", "D", "E").forEach(name -> acceptors.add(new Acceptor(name)));
        //开始投票
        Proposer.vote(new Proposal(1L, null), acceptors);
    }

    private static void printInfo(String subject, String operation,
                                  String result) throws InterruptedException {
        System.out.println(subject + ":" + operation + "<" + result + ">");
        Thread.sleep(1000);
    }

    /**
     * 对于提案的约束，第三方约束要求
     * 如果maxVote不存在，那么没有限制，下一次表决可以使用任意提案
     * 否则，下一次表决要沿用maxVote提案
     *
     * @param currentVoteNumber
     * @param proposals
     * @return
     */
    private static Proposal nextProposal(long currentVoteNumber, List<Proposal> proposals) {
        long voteNumber = currentVoteNumber + 1;
        //刚开始投票
        if (proposals.isEmpty()) {
            Proposal proposal = new Proposal(voteNumber,
                PROPOSALS[RAMDOM.nextInt(PROPOSALS.length)]);
            System.out.println("NEXT PROPOSER(刚开始投票), currentVoteNumber" + currentVoteNumber
                               + ",proposal=" + proposal);
            return proposal;
        }
        //为之前的票排序
        Collections.sort(proposals);
        Proposal maxVote = proposals.get(proposals.size() - 1);
        //列表里最大值
        long maxVoteNumber = maxVote.getVoteNumber();
        String content = maxVote.getContent();
        if (maxVoteNumber >= currentVoteNumber) {
            throw new IllegalStateException("illegal state maxVoteNumber");
        }

        if (content != null) {
            Proposal proposal = new Proposal(voteNumber, content);
            System.out.println("NEXT PROPOSER(content不为空), voteNumber=" + voteNumber + ",proposal="
                               + proposal + ",maxVote=" + maxVote);
            return proposal;
        } else {
            Proposal proposal = new Proposal(voteNumber,
                PROPOSALS[RAMDOM.nextInt(PROPOSALS.length)]);
            System.out.println("NEXT PROPOSER(content为空), voteNumber=" + voteNumber + ",proposal="
                               + proposal + ",maxVote=" + maxVote);
            return proposal;
        }
    }

    private static class Proposer {

        /**
         * @param proposal
         * @param acceptors
         */
        public static void vote(Proposal proposal,
                                Collection<Acceptor> acceptors) throws InterruptedException {
            //法定人数 n/2+1
            int quorum = Math.floorDiv(acceptors.size(), 2) + 1;
            System.out.println("quorum  " + quorum);
            int count = 0;
            while (true) {
                //开始投票
                printInfo("VOTE_ROUND", "START", ++count + "：开始向acceptor发送信息");
                List<Proposal> proposals = new ArrayList<>();

                for (Acceptor acceptor : acceptors) {
                    Promise promise = acceptor.onPrepare(proposal);
                    if (promise != null && promise.isAck()) {
                        //Acceptor批准后就可以将发起者加入发起者列表里
                        System.out
                            .println("--------------------------------------------------ACCEPTOR批准,将发起者加入队列里,proposal=" + promise.getProposal());
                        proposals.add(promise.getProposal());
                    }
                }
                if (proposals.size() < quorum) {
                    //小于法定人数
                    printInfo("PROPOSER[" + proposal + "]", "VOTE",
                        "NOT PREPARED：接收到消息的acceptor小于半数");
                    System.out.println("onPrepare阶段编号增加 重新发起投票");
                    proposal = nextProposal(proposal.getVoteNumber(), proposals);
                    continue;
                }
                System.out.println("接收到消息的acceptor大于半数，开始提案内容");
                int acceptCount = 0;
                for (Acceptor acceptor1 : acceptors) {
                    if (acceptor1.onAccept(proposal))
                        acceptCount++;
                }
                if (acceptCount < quorum) {
                    printInfo("PROPOSER[" + proposal + "]", "VOTE", "NOT ACCEPTED,接受提案的少于半数");
                    System.out.println("onAccept阶段编号增加 重新发起投票");
                    proposal = nextProposal(proposal.getVoteNumber(), proposals);
                    continue;
                }
                break;
            }

            printInfo("PROPOSER[" + proposal + "]", "VOTE", "SUCCESS，接受提案成功");

        }

    }

    private static class Acceptor {
        //上次表决结果
        private Proposal last = new Proposal();
        private String   name;

        public Acceptor(String name) {
            this.name = name;
        }

        public Promise onPrepare(Proposal proposal) throws InterruptedException {
            //假设这个过程有50%的几率失败
            if (Math.random() - 0.5 > 0) {
                printInfo("ACCEPTER_" + name, "PREPARE", "NO RESPONSE 向" + name + " 发送失败");
                return null;
            }
            if (proposal == null)
                throw new IllegalArgumentException("null proposal");
            //大于
            if (proposal.getVoteNumber() > last.getVoteNumber()) {
                Promise response = new Promise(true, last);
                last = proposal;
                printInfo("ACCEPTER_" + name, "PREPARE", "OK " + name + " 记下编号，返回信息,last=" + last);
                return response;
            } else {
                printInfo("ACCEPTER_" + name, "PREPARE", "REJECTED " + name + " 已保存有选定的编号，拒绝");
                return new Promise(false, null);
            }
        }

        public boolean onAccept(Proposal proposal) throws InterruptedException {
            //假设这个过程有50%的几率失败
            if (Math.random() - 0.5 > 0) {
                printInfo("ACCEPTER_" + name, "ACCEPT", "NO RESPONSE " + name + " 发送提案失败");
                return false;
            }

            boolean accepted = last.equals(proposal);
            printInfo("ACCEPTER_" + name, "ACCEPT",
                name + (accepted ? "OK：通过" : "NO: 不通过") + "last=" + last + ",proposal=" + proposal);
            return accepted;
        }
    }

    private static class Promise {
        private final boolean  ack;
        private final Proposal proposal;

        private Promise(boolean ack, Proposal proposal) {
            this.ack = ack;
            this.proposal = proposal;
        }

        public boolean isAck() {
            return ack;
        }

        public Proposal getProposal() {
            return proposal;
        }
    }

    private static class Proposal implements Comparable<Proposal> {
        private final long   voteNumber;
        private final String content;

        public Proposal(long voteNumber, String content) {
            this.voteNumber = voteNumber;
            this.content = content;
        }

        public Proposal() {
            this(0, null);
        }

        public long getVoteNumber() {
            return voteNumber;
        }

        public String getContent() {
            return content;
        }

        public int compareTo(Proposal o) {
            return Long.compare(voteNumber, o.voteNumber);
        }

        @Override
        public boolean equals(Object obj) {
            if (obj == null) {
                return false;
            }
            if (!(obj instanceof Proposal))
                return false;
            Proposal proposal = (Proposal) obj;
            return voteNumber == proposal.voteNumber
                   && StringUtils.equals(content, proposal.content);
        }

        @Override
        public int hashCode() {
            return HASH_FUNCTION.newHasher().putLong(voteNumber).putString(content, Charsets.UTF_8)
                .hash().asInt();
        }

        @Override
        public String toString() {
            return new StringBuilder().append(voteNumber).append(':').append(content).toString();
        }
    }
}
```