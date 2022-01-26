## paxos的算法演进

![](http://image.clickear.top/20220126145714.png)
![paxos](http://image.clickear.top/分布式算法(Paxos).png)


## [[MultiPaxos]] 、[[ZAB]]、[[Raft]] 为什么是等价算法？
核心的原理一样
1.  通过随机超时来实现无活锁的选主过程。
2.  通过主节点来发起写操作。同时只有1个主节点。
3.  通过心跳来检测存活性
4.  通过[Quorum](https://www.notion.so/Quorum-f31058d3ebfc4b5bb175b5181196f0a3)机制来保证一致性

[RAFT](https://blog.csdn.net/weixin_34401479/article/details/90588562?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0.pc_relevant_default&spm=1001.2101.3001.4242.1&utm_relevant_index=3)

具体细节上可以有差异：
譬如是全部节点都能参与选主，还是部分节点能参与，
譬如Quorum中是众生平等还是各自带有权重，
譬如该怎样表示值的方式


