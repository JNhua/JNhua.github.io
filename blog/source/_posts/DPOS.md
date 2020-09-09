---
title: 分布式共识算法
copyright: true
date: 2020-08-21 12:54:24
categories:
- Cryptography
tags:
---

分布式系统中需要共识算法来保证一致性。主要介绍Paxos和Raft两种。

<!-- more -->

# Paxos

用于达成共识性问题，即对多个节点产生的值，该算法能保证只选出唯一一个值。主要有三类节点：

* 提议者（Proposer）：提议一个值；
* 接受者（Acceptor）：对每个提议进行投票；
* 告知者（Learner）：被告知投票的结果，不参与投票过程。
  ![2e28af608c6c3a970a27becbbdaf7830](DPOS/4990FBD6-0AFD-4C25-9FD1-3E983330C9E1.jpg)

## 执行过程

规定一个提议包含两个字段：[n, v]，其中 n 为序号（具有唯一性），v 为提议值。

### 1. Prepare 阶段

下图演示了两个 Proposer 和三个 Acceptor 的系统中运行该算法的初始过程，每个 Proposer 都会向所有 Acceptor 发送 Prepare 请求。
![b89ed9e47cef15c83e61dc97bf1e0fdc](DPOS/1709B826-3F8D-4AC6-B7D0-1CBBBBD22174.png)
当 Acceptor 接收到一个 Prepare 请求，包含的提议为 [n1, v1]，并且之前还未接收过 Prepare 请求，那么发送一个 Prepare 响应，设置当前接收到的提议为 [n1, v1]，并且保证以后不会再接受序号小于 n1 的提议。如下图，Acceptor X 在收到 [n=2, v=8] 的 Prepare 请求时，由于之前没有接收过提议，因此就发送一个 [no previous] 的 Prepare 响应，设置当前接收到的提议为 [n=2, v=8]，并且保证以后不会再接受序号小于 2 的提议。其它的 Acceptor 类似。
![9f2998a5cf9e637bd89f01d25fa208d9](DPOS/91D616AF-BB7C-47C1-A5A0-EA27E44D0F5D.jpg)
如果 Acceptor 接收到一个 Prepare 请求，包含的提议为 [n2, v2]，并且之前已经接收过提议 [n1, v1]。如果 n1 &gt; n2，那么就丢弃该提议请求；否则，发送 Prepare 响应，该 Prepare 响应包含之前已经接收过的提议 [n1, v1]，设置当前接收到的提议为 [n2, v2]，并且保证以后不会再接受序号小于 n2 的提议。如下图，Acceptor Z 收到 Proposer A 发来的 [n=2, v=8] 的 Prepare 请求，由于之前已经接收过 [n=4, v=5] 的提议，并且 n &gt; 2，因此就抛弃该提议请求；Acceptor X 收到 Proposer B 发来的 [n=4, v=5] 的 Prepare 请求，因为之前接收到的提议为 [n=2, v=8]，并且 2 &lt;= 4，因此就发送 [n=2, v=8] 的 Prepare 响应，设置当前接收到的提议为 [n=4, v=5]，并且保证以后不会再接受序号小于 4 的提议。Acceptor Y 类似。
![e9871a19edc2ce86be42f40370afce65](DPOS/3FD2C361-8DC9-42DB-8BF9-2A7AD589FF73.jpg)

### 2. Accept 阶段

当一个 Proposer 接收到超过一半 Acceptor 的 Prepare 响应时，就可以发送 Accept 请求。Proposer A 接收到两个 Prepare 响应之后，就发送 [n=2, v=8] Accept 请求。该 Accept 请求会被所有 Acceptor 丢弃，因为此时所有 Acceptor 都保证不接受序号小于 4 的提议。Proposer B 过后也收到了两个 Prepare 响应，因此也开始发送 Accept 请求。需要注意的是，Accept 请求的 v 需要取它收到的最大提议编号对应的 v 值，也就是 8。因此它发送 [n=4, v=8] 的 Accept 请求。
![d29e4a5e9bc89c187674ebd884a7f571](DPOS/81ECBC9B-E58C-4C81-8820-E5393CCA45CA.png)

### 3. Learn 阶段

Acceptor 接收到 Accept 请求时，如果序号大于等于该 Acceptor 承诺的最小序号，那么就发送 Learn 提议给所有的 Learner。当 Learner 发现有大多数的 Acceptor 接收了某个提议，那么该提议的提议值就被 Paxos 选择出来。
![b9a37e5de8950c24bd5275f5253404ef](DPOS/4AA38D03-BC1D-4610-B6F8-6B0B8E1F0607.jpg)

## 约束条件

### 1. 正确性

指只有一个提议值会生效。因为 Paxos 协议要求每个生效的提议被多数 Acceptor 接收，并且 Acceptor 不会接受两个不同的提议，因此可以保证正确性。

### 2. 可终止性

指最后总会有一个提议生效。Paxos 协议能够让 Proposer 发送的提议朝着能被大多数 Acceptor 接受的那个提议靠拢，因此能够保证可终止性。

# Raft

## leader选举

### 状态转移

启动时都是follower状态。
![286e98289b427157f1d92776d11715e2](DPOS/5D9C4562-02AD-48F1-9273-3DC7D66DBCB8.png)
系统中最多只有一个 leader，如果在一段时间里发现没有 leader，则大家通过选举 - 投票选出 leader。leader 会不停的给 follower 发心跳消息，表明自己的存活状态。如果 leader 故障，那么 follower 会转换成 candidate，重新选出 leader。

### 任期

![f4521ada1e57e1ccfc895ecb9a197e08](DPOS/3D62AE1A-FA35-42DD-967B-9D63D11A3AD6.png)
term（任期）以选举（election）开始，然后就是一段或长或短的稳定工作期（normal Operation）。从上图可以看到，任期是递增的，这就充当了逻辑时钟的作用；另外，term 3 展示了一种情况，就是说没有选举出 leader 就结束了，然后会发起新的选举，这是平票 split vote 的情况。

## log replication

### 复制状态机

相同的初识状态 + 相同的输入 = 相同的结束状态。
在 raft 中，leader 将客户端请求（command）封装到一个个 log entry，将这些 log entries 复制（replicate）到所有 follower 节点，然后大家按相同顺序应用（apply）log entry 中的 command，则状态肯定是一致的。

### 请求完整流程

当系统（leader）收到一个来自客户端的写请求，到返回给客户端，整个过程从 leader 的视角来看会经历以下步骤：

* leader append log entry
* leader issue AppendEntries RPC in parallel
* leader wait for majority response
* leader apply entry to state machine
* leader reply to client
* leader notify follower apply log

## 安全

### 选举安全

在 raft 中，两点保证了这个属性：

* 一个节点某一任期内最多只能投一票；
* 只有获得 majority 投票的节点才会成为 leader。
  任一任期内最多一个 leader 被选出。

### log 匹配

如果两个节点上的某个 log entry 的 log index 相同且 term 相同，那么在该 index 之前的所有 log entry 应该都是相同的。
首先，leader 在某一 term 的任一位置只会创建一个 log entry，且 log entry 是 append-only。其次，consistency check。leader 在 AppendEntries 中包含最新 log entry 之前的一个 log 的 term 和 index，如果 follower 在对应的 term index 找不到日志，那么就会告知 leader 不一致。
当出现了 leader 与 follower 不一致的情况，leader 强制 follower 复制自己的 log。

* leader 初始化 nextIndex[x] 为 leader 最后一个 log index + 1
* AppendEntries 里 prevLogTerm prevLogIndex 来自 logs[nextIndex[x] - 1]
* 如果 follower 判断 prevLogIndex 位置的 log term 不等于 prevLogTerm，那么返回 False，否则返回 True
* leader 收到 follower 的回复，如果返回值是 False，则 nextIndex[x] -= 1, 跳转到 第2步. 
* 否则，同步 nextIndex[x] 后的所有 log entries

### leader完整性

leader 完整性：如果一个 log entry 在某个任期被提交（committed），那么这条日志一定会出现在所有更高 term 的 leader 的日志里面。
raft 与其他协议（Viewstamped Replication、mongodb）不同，raft 始终保证 leader 包含最新的已提交的日志，因此 leader 不会从 follower catchup 日志，这也大大简化了系统的复杂度。

## 边界条件

### 旧leader

raft 保证 Election safety，即一个任期内最多只有一个 leader，但在网络分割（network partition）的情况下，可能会出现两个 leader，但两个 leader 所处的任期是不同的。
leader 如果收不到 majority 节点的消息，那么可以自己 step down，自行转换到 follower 状态。

### 状态机安全

如果节点将某一位置的 log entry 应用到了状态机，那么其他节点在同一位置不能应用不同的日志。简单点来说，所有节点在同一位置（index in log entries）应该应用同样的日志。
某个 leader 选举成功之后，不会直接提交前任 leader 时期的日志，而是通过提交当前任期的日志的时候 “顺手” 把之前的日志也提交了，具体怎么实现了，在 log matching 部分有详细介绍。那么问题来了，如果 leader 被选举后没有收到客户端的请求呢，论文中有提到，在任期开始的时候发立即尝试复制、提交一条空的 log。

### leader crash

[不同情况的图解](https://www.cnblogs.com/mindwind/p/5231986.html)

## 为了避免问题的设计

* 避免平票：randomized election timeouts ；
* 保证majority票：节点的数目都是奇数个；
* 保证高可用：不是强一致性，而是最终一致性，leader 会不断尝试给 follower 发 log entries，直到所有节点的 log entries 都相同；

# 参考

[一文搞懂Raft算法](https://www.cnblogs.com/xybaby/p/10124083.html)