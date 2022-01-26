# 总结



# 常青笔记
+ 基于[[Paxos]]
+ 选主过程，也是[[领导者模型]]，即只有1个提案者。解决了活锁问题。当票数相同时，使用**随机超时心跳**。通过**心跳**来检测存活性，如果超时了，则重新发起选择leader。
+ 通过主节点来发起写操作，并进行日志复制。[[多数派]] 基于[[状态机复制]] 的日志复制来实现各个节点的数据一致性。
+ 是[[AP]]模型
+ [[脑裂]]

# 重点摘要
这是一种通过对日志进行管理来达到一致性的算法，其是一种 [[AP]] 的一致性算法。Raft 通过选举 Leader 并由 Leader 节点负责管理日志复制来实现各个节点间数据的一致性。

## 三种角色
在 Raft 中，节点有三种角色： 
### Leader：
唯一负责处理客户端写请求的节点；也可以处理客户端读请求；同时负责日志复制工作。[[状态机复制]]
### Candidate：
Leader 选举的候选人，其可能会成为 Leader 
### Follower：
可以处理客户端读请求；负责同步来自于 Leader 的日志；当接收到其它 Cadidate 的投票请求后可以进行投票；当发现 Leader 挂了，其会转变为 Candidate 发起 Leader 选举。
term，任期，相当于 Paxos 中的 epoch，表示一个新的 leader 上任了。

## 动画
[RAFT动画演示](http://thesecretlivesofdata.com/raft/)
Q&A环节:
1. Q: 怎么选leader？
A: 若 follower 在心跳超时范围内没有接收到来自于 leader 的心跳，则认为 leader 挂了。发起选举 --> 投票 --> 广播结果。
---
Q: 如果票数相同怎么处理？
A: 进行随机超时重新选择。

---

## 数据同步
Raft 算法一致性的实现，是基于[[状态机复制|日志复制状态机]]的。状态机的最大特征是，不同 Server 中的状态机若当前状态相同，然后接受了相同的输入，则一定会得到相同的输出。是一个[[AP]]模型。
### 过程
![同步过程](http://image.clickear.top/20220127002258.png)

当 leader 接收到 client 的写操作请求后，大体会经历以下流程： 1. leader 将数据封装为日志 
1. leader将日志并行发送给所有 follower，然后等待接收 follower 响应 
2. 当 leader 接收到过半响应后，将日志 commit 到自己的状态机，状态机会输出一个结果， 同时日志状态变为了 committed 同时 leader 还会通知所有 follower 
3. 将日志 apply 到它们本地的状态机，日志状态变为了 applied 
4. 在 apply 通知发出的同时，leader 也会向 client 发出成功处理的响应
### [[AP]]模型
![](http://image.clickear.top/20220127002340.png)
Log 由 term index、log index 及 command 构成。为了保证可用性，各个节点中的日志可 以不完全相同，但 leader 会不断给 follower 发送 log，以使各个节点的 log 最终达到相同。 即 raft 算法不是强一致性的，而是[[最终一致性]]的

## 演进

![[分布式算法#paxos的算法演进]]


## Multipaxos、ZAB、Raft为什么是等价算法？
![[分布式算法#MultiPaxos 、 ZAB 、 Raft 为什么是等价算法？]]


## Leader宕机
总原则: 
1. 客户端重试（处理未成功）
2. 有点类似数据库的二阶段提交。先prepare，在commit。
	1. 

### （1） 请求到达前 Leader 挂了
client 发送写操作请求到达 Leader 之前 Leader 就挂了，因为请求还没有到达集群，所以 这个请求对于集群来说就没有存在过，对集群数据的一致性没有任何影响。Leader 挂了之 后，会选举产生新的 Leader。
由于 Stale Leader（失效的 Leader）并未向 client 发送成功处理响应，所以 client 会重新 发送该写操作请求（若 Client 具有重试机制的话)
### （2） 未开始同步数据前 Leader 挂了
client 发送写操作请求给 Leader，请求到达 Leader 后，Leader 还没有开始向 Followers 复制数据 Leader 就挂了。这时集群会选举产生新的 Leader，Stale Leader 重启后会作为Follower 重新加入集群，并同步新 Leader 中的数据以保证数据一致性。之前接收到 client 的 数据被丢弃。 由于 Stale Leader 并未向 client 发送成功处理响应，所以 client 会重新发送该写操作请求 （若 Client 具有重试机制的话）
### （3） 同步完部分后 Leader 挂了
client 发送写操作请求给 Leader，Leader 接收完数据后开始向 Follower 复制数据。在部分 Follower 复制完后 Leader 挂了（可以过半也可以不过半）。由于 Leader 挂了，就会发起新 的 Leader 选举。 
+ 若 Leader 产生于已经复制完日志的 Follower，其会继续将前面接收到的写操作请求完成，并向 client 进行响应。 
+ 若 Leader 产生于尚未复制日志的 Follower，那么原来已经复制过日志的 Follower 则会将这个没有完成的日志放弃。由于 client 没有接收到响应，所以 client 会重新发送该写操作请求（若 Client 具有重试机制的话）。

### （4） apply 通知发出后 Leader 挂了
client 发送写操作请求给 Leader，Leader 接收完数据后开始向 Follower 复制数据。Leader 成功接收到过半 Follower 复制完毕的响应后，Leader 将日志写入到状态机。此时 Leader 向 Follower 发送 apply 通知。在发送通知的同时，也会向 client 发出响应。此时 leader 挂了。
由于 Stale Leader 已经向 client 发送成功接收响应，且 apply 通知已经发出，说明这个写操作请求已经被 server 成功处理。