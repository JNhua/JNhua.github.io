---
title: Paxos
date: '2020/9/08 19:22:56'
updated: '2020/11/23 19:57:31'
tags: []
category:
  - 区块链
  - 共识协议
mathjax: true
toc: false
abbrlink: c20c6b6
---
# Paxos
用于达成共识性问题，即对多个节点产生的值，该算法能保证只选出唯一一个值。主要有三类节点：
<!--more-->
* 提议者（Proposer）：提议一个值；
* 接受者（Acceptor）：对每个提议进行投票；
* 告知者（Learner）：被告知投票的结果，不参与投票过程。

![4990FBD6-0AFD-4C25-9FD1-3E983330C9E1](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110243.jpg)

## 执行过程

规定一个提议包含两个字段：[n, v]，其中 n 为序号（具有唯一性），v 为提议值。

### 1. Prepare 阶段

下图演示了两个 Proposer 和三个 Acceptor 的系统中运行该算法的初始过程，每个 Proposer 都会向所有 Acceptor 发送 Prepare 请求。
![1709B826-3F8D-4AC6-B7D0-1CBBBBD22174](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110250.png)
当 Acceptor 接收到一个 Prepare 请求，包含的提议为 [n1, v1]，并且之前还未接收过 Prepare 请求，那么发送一个 Prepare 响应，设置当前接收到的提议为 [n1, v1]，并且保证以后不会再接受序号小于 n1 的提议。如下图，Acceptor X 在收到 [n=2, v=8] 的 Prepare 请求时，由于之前没有接收过提议，因此就发送一个 [no previous] 的 Prepare 响应，设置当前接收到的提议为 [n=2, v=8]，并且保证以后不会再接受序号小于 2 的提议。其它的 Acceptor 类似。
![91D616AF-BB7C-47C1-A5A0-EA27E44D0F5D](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110258.jpg)
如果 Acceptor 接收到一个 Prepare 请求，包含的提议为 [n2, v2]，并且之前已经接收过提议 [n1, v1]。如果 n1 &gt; n2，那么就丢弃该提议请求；否则，发送 Prepare 响应，该 Prepare 响应包含之前已经接收过的提议 [n1, v1]，设置当前接收到的提议为 [n2, v2]，并且保证以后不会再接受序号小于 n2 的提议。如下图，Acceptor Z 收到 Proposer A 发来的 [n=2, v=8] 的 Prepare 请求，由于之前已经接收过 [n=4, v=5] 的提议，并且 n &gt; 2，因此就抛弃该提议请求；Acceptor X 收到 Proposer B 发来的 [n=4, v=5] 的 Prepare 请求，因为之前接收到的提议为 [n=2, v=8]，并且 2 &lt;= 4，因此就发送 [n=2, v=8] 的 Prepare 响应，设置当前接收到的提议为 [n=4, v=5]，并且保证以后不会再接受序号小于 4 的提议。Acceptor Y 类似。
![3FD2C361-8DC9-42DB-8BF9-2A7AD589FF73](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110305.jpg)

### 2. Accept 阶段

当一个 Proposer 接收到超过一半 Acceptor 的 Prepare 响应时，就可以发送 Accept 请求。Proposer A 接收到两个 Prepare 响应之后，就发送 [n=2, v=8] Accept 请求。该 Accept 请求会被所有 Acceptor 丢弃，因为此时所有 Acceptor 都保证不接受序号小于 4 的提议。Proposer B 过后也收到了两个 Prepare 响应，因此也开始发送 Accept 请求。需要注意的是，Accept 请求的 v 需要取它收到的最大提议编号对应的 v 值，也就是 8。因此它发送 [n=4, v=8] 的 Accept 请求。
![81ECBC9B-E58C-4C81-8820-E5393CCA45CA](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110313.png)

### 3. Learn 阶段

Acceptor 接收到 Accept 请求时，如果序号大于等于该 Acceptor 承诺的最小序号，那么就发送 Learn 提议给所有的 Learner。当 Learner 发现有大多数的 Acceptor 接收了某个提议，那么该提议的提议值就被 Paxos 选择出来。
![4AA38D03-BC1D-4610-B6F8-6B0B8E1F0607](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110325.jpg)

## 约束条件

### 1. 正确性

指只有一个提议值会生效。因为 Paxos 协议要求每个生效的提议被多数 Acceptor 接收，并且 Acceptor 不会接受两个不同的提议，因此可以保证正确性。

### 2. 可终止性

指最后总会有一个提议生效。Paxos 协议能够让 Proposer 发送的提议朝着能被大多数 Acceptor 接受的那个提议靠拢，因此能够保证可终止性。

# 参考

[分布式-Paxos](https://blog.csdn.net/qq_16038125/article/details/81059609)