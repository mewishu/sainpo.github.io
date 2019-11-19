---
title: paxos-by-example
date: 2019-07-29 20:31:28
categories: 
- 分布式一致性
- paxos算法
tags:
- paxos
- 分布式一致性
---

Paxos算法在分布式领域具有非常重要的地位。但是Paxos算法有两个比较明显的缺点：1.难以理解 2.工程实现更难。

网上有很多讲解Paxos算法的文章，但是质量参差不齐。看了很多关于Paxos的资料后发现，学习Paxos最好的资料是论文[Paxos Made Simple](https://www.cnblogs.com/YaoDD/p/6150498.html)，其次是中、英文版维基百科对Paxos的介绍。本文试图带大家一步步揭开Paxos神秘的面纱。

## Paxos是什么

> Paxos算法是基于**消息传递**且具有**高度容错特性**的**一致性算法**，是目前公认的解决**分布式一致性**问题**最有效**的算法之一。

Google Chubby的作者Mike Burrows说过这个世界上**只有一种**一致性算法，那就是Paxos，其它的算法都是**残次品**。

虽然Mike Burrows说得有点夸张，但是至少说明了Paxos算法的地位。然而，Paxos算法也因为晦涩难懂而臭名昭著。本文的目的就是带领大家深入浅出理解Paxos算法，不仅理解它的执行流程，还要理解算法的推导过程，作者是怎么一步步想到最终的方案的。只有理解了推导过程，才能深刻掌握该算法的精髓。而且理解推导过程对于我们的思维也是非常有帮助的，可能会给我们带来一些解决问题的思路，对我们有所启发。

最近在看paxos算法, 看了不少博客, 都感觉总结得不够到位,甚至有理解偏差的地方,倒是在博客的引用文章里面找到了一篇英文文献,看完后豁然开朗茅塞顿开.

于是把那篇英文原文翻译了一下, 加深一下理解

英文文献虽是2012年的了, 但是短小精悍,值得一看,原文地址是:

> [angus.nyc/2012/paxos-by-example/ ](https://angus.nyc/2012/paxos-by-example/)

文章中不全是直译, 有些地方改成了我自己的理解, 所以有可能有理解错的地方, 还望指出.



## 关于作者: Lamport

提到Paxos算法，我们不得不首先来介绍下Paxos算法的作者Leslie Lamport(莱斯利·兰伯特, http://www.lamport.org )及其对计算机科学尤其是分布式计算领域的杰出贡献。作为2013年的新科图灵奖得主，Lamport是计算机科学领域一位拥有杰出成就的传奇人物，其先后多次荣获ACM和IEEE以及其他各类计算机重大奖项。Lamport对时间时钟、面包店算法、拜占庭将军问题以及Paxos算法的创造性研究，极大地推动了计算机科学尤其是分布式计算的发展，全世界无数工程师得益于他的理论，其中Paxos算法的提出，正是Lamport多年的研究成果。

说起Paxos理论的发表，还有一段非常有趣的历史故事。Lamport早在1990年就已经将其对Paxos算法的研究论文[《**The PartTime Parliament**》](http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-paxos.pdf)提交给ACM TOCS Jnl.的评审委员会了，但是由于Lamport“创造性”地使用了故事的方式来进行算法的描述，导致当时委员会的工作人员没有一个能够正确地理解他对算法的描述，时任主编要求Lamport使用严谨的数据证明方式来描述该算法，否则他们将不考虑接受这篇论文。遗憾的是，Lamport并没有接收他们的建议，当然也就拒绝了对论文的修改，并撤销了对这篇论文的提交。在后来的一个会议上，Lamport还对此事耿耿于怀：“为什么这些搞理论的人一点幽默感也没有呢？”

幸运的是，还是有人能够理解Lamport那公认的令人晦涩的算法描述的。1996年，来自微软的Butler Lampson在WDAG96上提出了重新审视这篇分布式论文的建议，在次年的WDAG97上，麻省理工学院的Nancy Lynch也公布了其根据Lamport的原文重新修改后的[《**Revisiting the Paxos Algorithm**》](http://research.microsoft.com/en-us/um/people/blampson/60-PaxosAlgorithm/Acrobat.pdf)，“帮助”Lamport用数学的形式化术语定义并证明了Paxos算法。于是在1998年的ACM TOCS上，这篇延迟了9年的论文终于被接受了，也标志着Paxos算法正式被计算机科学接受并开始影响更多的工程师解决分布式一致性问题。

后来在2001年，Lamport本人也做出了让步，这次他放弃了故事的描述方式，而是使用了通俗易懂的语言重新讲述了原文，并发表了[《**Paxos Made Simple**》](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf)——当然，Lamport甚为固执地认为他自己的表述语言没有歧义，并且也足够让人明白Paxos算法，因此不需要数学来协助描述，于是整篇文章还是没有任何数学符号。好在这篇文章已经能够被更多的人理解，相信绝大多数的Paxos爱好者也都是从这篇文章开始慢慢进入了Paxos的神秘世界。

由于Lamport个人自负固执的性格，使得Paxos理论的诞生可谓一波三折。关于Paxos理论的诞生过程，后来也成为了计算机科学领域被广泛流传的学术趣事。

![Lamport本尊](Lamport.png)

> 图0: Lamport本尊

## 译文开始

本文通过一个实例描述了一个叫Paxos的分布式一致性算法

分布式一致性算法通常是用于让多个计算机节点就某个单值的修改达成一致, 例如事务的提交commit或回滚rollback, 典型的思想就是二阶段提交或三阶段提交.

算法并不关心这个单值是什么,只关心最后只有唯一一个值被所有的节点选中, 对该值的修改达成一致

在一个分布式系统中这非常的难, 因为不同机器之前的消息可能会丢失, 或是被无限延迟, 甚至机器本身也会挂掉

Paxos算法可以保证最终只会有一个值会被选中, 但是不能保证如果集群中的大多数节点挂掉后,还能选中一个值

### 概述

Paxos的节点可以是proposer, acceptor,learner中的任何一种角色.proposer会提议一个它想被采纳的值, 一般是通过发送一个包含该提议值的提案给集群中的每个acceptor.  然后acceptor会独立的决定是否接受这个提案--它可能会接收到多个proposer的不同提案, 决定好后会把它的决定发给learner.  learner的作用是判断一个值是否已经被接受.在Paxos算法中, 只有一个值被大多数的acceptor选中, 才算被Paxos算法选中.在实际项目中, 一个节点可能担任多种角色, 但是在这个例子里面, 每一种角色都是一个独立的节点.

![Paxos的基本架构](Figure1-BasicPaxos.png)

> 图1 Paxos的基本架构. 几个proposer向acceptor提交提案. 当一个acceptor接受了一个值后, 它把它的决定发给learner

### Paxos算法例子

在标准的Paxos算法中, 一个proposer会发送两种类型的消息给acceptors: prepare request 和accept request. 在算法的第一阶段, proposer会发送一个prepare request给每一个acceptor, 这个prepare request包含一个提议值value和一个提案号number.每一个proposer的提案号number必须是正数, 单调递增的,独一无二的自然数<sup>[[1\]](#fn1)</sup>.

在下面图2的例子中, 有两个proposer, 都在发送prepare request. proposer A的请求([n=2,v=8])比proposer B的请求([n=4,v=5])先到达acceptor X和acceptor Y, 而proposer B的请求先到达acceptor Z

![Paxos 2](Figure2-Paxos.png)

> 图2: Paxos. proposer A和B各发送了一个prepare request 给每一个accetor. 在这个例子中, proposer A的请求先到达acceptor X和Y, 而proposer B的请求先到达了acceptor Z.

如果一个acceptor接收到了一个prepare request而又没接收过其他的提案, 这个acceptor会回应一个prepare response, 并保证不会接受其他比当前提案号number更小的提案. 下面图3展示了每个acceptor是如何响应它们接收到的prepare request的

![Paxos 3](Figure3-Paxos.png)

> 图3: 每个acceptor都对它接收到的第一个prepare request 进行prepare response.

最终, acceptor Z接收到了proposer A的请求<sup>[[2\]](#fn2)</sup>, acceptor X和Y收到了proposer B的请求.

如果一个acceptor之前已经接收了一个提案号更高的prepare request的话, 那么后面收到的prepare request就会被忽略, 这个例子中就是acceptor Z会忽略掉proposer A的请求(Z已经接收了B的提案n=4).

如果acceptor之前没有接收过更高提案号的提案, 它会保证忽略其他提案号比这个更低的请求(注意是包括prepare request和accept request),然后在prepare response中把它已经选中的提案号最高的提议(包括这个提议的值)发送给proposer.在这个例子就是acceptor X和Y给proposer B的响应(X和Y已经接受了一个B的[n=2,v=8]的提案,当再收到[n=4, v=5]的提案时, 它保证会忽略任何n<4的prepare request和accept request, 然后把[n=2, v=8]的prepare response回去给B)

![Paxos 4](Figure4-Paxos.png)

> 图4: acceptor Z 忽略了proposer A的请求, 因为它已经接收到了更高的提议号(4 > 2). acceptor X和Y给proposer B响应了它见过最高的提议号的提案, 并承诺会忽略提案号更小的提案

一旦proposer 收到了大多数acceptor的prepare response, 它就可以开始给所有的acceptor发送accept request了. 因为proposer A接收到prepare response中表明在它之前没有更早的提案. 所以它的accept request中的提案号和提议值都是跟prepare request中的一样.然而,这个accept request被所有的acceptor都拒绝了,因为它们都承诺了不会接收提议号比4更新的提案(因为都见过proposer B的提案了)

Proposer B的accept request情况就跟proposer A的有所不同了. 首先它的提案号还是它之前的提案号(n=4), 但是提议值却不是它之前的提议值了(它之前提议的是v=5), 而是它在prepare response中接收到的提议值(v=8)<sup>[[3\]</sup>](#fn3)

![Paxos 5](Figure5-Paxos.png)

> 图5: proposer B也发送了一个accept request给每一个acceptor.acceptor request中的提案号是它之前的提案号(4), 而提议值是它在prepare response中收到的值(8, 来自proposer A的提案[n=2, v=8])

如果acceptor收到了一个提案号不小于它见过的accept request, 它会接受这个accept request并且发送一个通知给到learner节点. 当learner节点发现大多数acceptor都已经接受了一个值的时候,Paxos算法就认为这个值已经被选中.

![Paxos 6](Figure6-OnceAvalueHas.png)

当一个值被paxos选中后,后续的流程中将不能改变这个值.例如,当另一个proposer C发送一个提案号更高的prepare request(例如, [n=6, v=7])过来时,所有的acceptor都会响应它之前已经选中的提议(n=4,v=8).这会要求proposer C在后续的acceptor request中也只能发送[n=6, v=8]的提议, 即只能确认一遍已经选中的这个值. 再后面, 如果有少量的acceptor还没选中一个值的话, 这个流程会保证最终所有的节点都会一致地选中同一个值.

## 译文结束

------

1. 提案号的唯一性的保证不是Paxos算法的内容 [↩︎](#fnref1)
2. 这个也可能不会发送, 但是这个算法中也是有可能的 [↩︎](#fnref2)
3. 注意这个是它从prepare response中收到的最高的提案号. 在这个例子中, proposer B的提案号(n=4)是比proposer A的提案号(n=2)大. 但是在它的prepare request的响应中, 它只收到了proposer A的提议([n=2, v=8]). 如果它在prepare response中没有收到其他的提案的话, 它就会发送自己的提案([n=4,v=5]) [↩︎](#fnref3)



## 参考资料

- NEAT ALGORITHMS - PAXOS(小动画可以辅助读懂): http://harry.me/blog/2014/12/27/neat-algorithms-paxos/

- paxos-by-example(通过一个例子理解paxos算法): https://juejin.im/post/5d159590f265da1b86089a89

- Paxos Made Live: an amazing paper from Google describing the challenges of their Paxos implementation in Chubby: http://static.googleusercontent.com/media/research.google.com/en//archive/paxos_made_live.pdf

- A Quora thread explaining Paxos in a few different ways: https://www.quora.com/Distributed-Systems/What-is-a-simple-explanation-of-the-Paxos-algorithm

- Raft - An Understandable Consensus Algorithm. Raft is another conensus algorithm designed for humans to understand, which if you can tell from the above wall of text might be a problem with Paxos. https://ramcloud.stanford.edu/raft.pdf

- 左耳听风 - 推荐阅读：分布式数据调度相关论文: http://10-02.com/180118-32%20_%20%E6%8E%A8%E8%8D%90%E9%98%85%E8%AF%BB%EF%BC%9A%E5%88%86%E5%B8%83%E5%BC%8F%E6%95%B0%E6%8D%AE%E8%B0%83%E5%BA%A6%E7%9B%B8%E5%85%B3%E8%AE%BA%E6%96%87.html

- 

  