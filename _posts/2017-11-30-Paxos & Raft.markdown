---
title:  “Paxos & Raft”
mathjax: true
layout: post
date:   2017-11-30 08:00:12 +0800
categories: distributed system
---

本文内容攒自网上，主要是对分布式协议的学习。

## basic-paxos
Paxos的目的是多个参与者达成一致观点，其基于一个原则，即一致的观点不会在传递过程中反转。

# basic-paxos
basic paxos的角色中，谨记一个原则：每个acceptor维护了自己知道的最大的协议号 vx，且
每个acceptor不会同意任意一项小于等于vx的协议的prepare请求。


basic paxos是由client发起的同步过程，在两阶段返回前，client不能得到成功的返回。
* 第一阶段a（发送prepare），proposer向acceptor提出一个协议，这里的协议可以理解为client
发送过来期望多节点一起保存的一致性内容，举例：一句日志，某个配置等
* 第一阶段b（计算协议vn），根据半数以上acceptor的返回，选择 max{va,vb,vc} = vn，这里
的vx（va，vb，vc中的某一个）理解为这个acceptor已知的最大协议号，acceptor一旦返回了vx后，则表明：
  + acceptor在接下来的prepare请求中，会返回的vx自增1
  + acceptor不会accept任何小于vx的协议请求，只会accept大于vx的协议请求
* 第二阶段a（发送决议好的vn），把vn发送给acceptor
* 第二阶段b，在半数acceptor返回了成功后，再返回client成功通过协议

引用wiki上的流程图：  
![image01]({{site.baseurl}}/image/basic-paxos.png)

1. proposer是可以有多个，所以他叫做proposer，而不叫leader
2. proposer并不强制要发送propose到全部acceptor，只要通过的有半数以上，就认为协议是成功的
3. 容易看出，第二阶段的时候client仍需要等着，只有在第二阶段大半acceptor返回accepted后，client才能得到成功的信息
4. 有一些错误场景中，proposer会互相锁住对方的递交

# basic-paxos的问题
basic-paxos协议在多个proposer中选择最大协议号：max{va,vb,vc} = vn。一个极端的例子如下：  
![image02]({{site.baseurl}}/image/proposer-loop.png)

大家交替propose的结果就是所有的prepare计算得到的vn，全部中途作废，accept动作一个都没有正常执行。

## multi-paxos
最重要的问题在于，只允许一个proposer。

multi-paxos将集群状态分成了两种：
* 选主状态，由集群中的任意节点拉票发起选主，拉票中带上自己的vx，通过收集集群中半数以上的vx，
来更新自己的vx值，得到目前集群通过的最大vx = vn
* 强leader状态，leader对vn的演变了如指掌，每次把vn的值直接在一阶段中发送给acceptor，
和basic paxos的区别：basic paxos一阶段的时候，proposer对vn的值一无所知，要依赖一阶段的结果来算vn

![image03]({{site.baseurl}}/image/multi-paxos.png)

流程图中没有了basic paxos的两阶段，变成了一个一阶段的递交协议：
1. 发起者（leader）直接告诉Acceptor，准备递交协议号为I+1的协议
2. 收到了大部分acceptor的回复后（图中是全部），acceptor就直接回复client协议成功写入

## Raft
multi-paxos和raft，在选定了leader状态下的行为模式一模一样。不同点在于：
* raft仅允许日志最多的节点当选为leader，而multi-paxos则相反，任意节点都可以当选leader
* raft不允许日志的空洞，这也是为了比较方便和拉平两个节点的日志方便，而multi-paxos则允许日志出现空洞。

[比较raft ，basic paxos以及multi-paxos]:https://zhuanlan.zhihu.com/p/25664121
