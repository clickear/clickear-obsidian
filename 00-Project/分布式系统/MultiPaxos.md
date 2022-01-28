


# 常青笔记
+ Multi-Paxos 思想，描述的是执行多个 Basic Paxos 实例，就**一系列值**达成共识。
+ 相比[[Paxos| basic paxos]] 优化思路是，引入[[领导者模型]]来优化。这样就省掉了多次rpc调用和活锁问题。优化 Basic Paxos 执行。






# 概念


## 优化
1. 引入 领导者模型，这样避免了活锁和多次rpc调用。
2. 优化 Basic Paxos 执行，因为只有1个提案者，所以每次都是最新的。

![multiPaxos 优化 Basic Paxos 执行](http://image.clickear.top/20220128142601.png)

