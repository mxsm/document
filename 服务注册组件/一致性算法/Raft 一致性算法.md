- 1 复制状态

### 1. 复制机状态/Replicated state machines

复制状态机的想法是将服务器看成一个状态机，而一致性算法的目的是让多台服务器/状态机能够计算得到相同的状态，同时，如果有部分机器宕机，集群作为一个整体依然能继续工作。复制状态机一般通过复制日志（replicated log）来实现，如下图：

![](https://github.com/mxsm/document/blob/master/image/%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/%E4%B8%80%E8%87%B4%E6%80%A7%E7%AE%97%E6%B3%95/%E5%A4%8D%E5%88%B6%E7%8A%B6%E6%80%81%E6%9C%BA.png?raw=true)

服务器会将客户端发来的命令存成日志，日志是有序的。而服务器状态机执行命令的结果是确定的，这样如果每台服务器的状态机执行的命令是相同的，状态机最终的状态也会是相同的，输出的结果也会是相同的。而如何保证不同服务器间的日志是一样的呢？这就是其中的“一致性模块”的工作了。

一致性模块（consensus module）在收到客户端的命令时（②），一方面要将命令添加到自己的日志队列中，同时需要与其它服务器的一致性模块沟通，确保所有的服务器将**最终**拥有相同的日志，即使有些服务器可能挂了。实践中至少需要“**大多数(大于一半)**”服务器同步了命令才认为同步成功了。

### 2. Raft算法

raft算法从三个方面去讲解：

- **选主（Leader Election）。Raft 在同一时刻只有一个主节点能接收写命令**
- **日志复制（Log Replication）。Raft 如何将接收到的命令复制到其它服务器上，使其保持一致？**
- **安全性，为什么 Raft 在各种情况下依旧能保证各服务器的日志一致性？**

#### 2.1 选主

[https://lotabout.me/2019/Raft-Consensus-Algorithm/#%E5%A4%8D%E5%88%B6%E7%8A%B6%E6%80%81%E6%9C%BA-replicated-state-machines](https://lotabout.me/2019/Raft-Consensus-Algorithm/#复制状态机-replicated-state-machines)