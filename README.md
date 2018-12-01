# PaxosTotemGossipTree
分布式通信协议

<pre>
背景：
        在分布式中，最难解决的一个问题就是多个节点间数据同步问题，为了解决这样的问题。涌现
    出了各种协议。
</pre>

<pre>
简单即有效----totem协议
        它将多个节点组成一个令牌环。多个节点手拉手形成一个圈，大家一次的传递token,只有获取
    到token的节点才有发送消息的权利。简单有效的解决了再分布式系统中各个节点的同步问题，因
    为只有一个节点会在一个时刻发送消息，不会出现冲突。当然，如果有节点发生意外时，令牌环
    就会断掉，吃屎大家不能够通信，而是重新组建出一个新的令牌环。
</pre>

<pre>
Paxos协议
</pre>

<pre>
Gossip协议
</pre>

![](https://i.imgur.com/VNg3X1N.png)

数据同步

![](https://i.imgur.com/4bXX2hq.png)


![](https://i.imgur.com/EtzK6ql.png)

![](https://i.imgur.com/N6gDT37.png)

<pre>
Zab协议
      1:Zab协议是专门为Zookeeper实现分布式协调功能而设计。Zookeeper主要是根据Zab协议实现
        分布式系统数据一致性。
      2：Zookeeper根据Zab协议建立了主备模型完成Zookeeper集群中数据的同步。这里所说的主备
        系统架构模型是指：在Zookeeper集群中，只有一台Leader负责处理外部客户端的事务请求，
        然后Leader服务器将客户端的写操作数据同步到所有的follower节点中。

      Zab协议是在整个Zookeeper集群中只有一个节点即Leader将客户端写操作转化为事务
     （或提议proposal）。Leader节点在数据写完之后，将向所有的follower节点发送数据广播请求
      ，等待所有的follower节点反馈。在Zab协议中，只要超过半数follower节点反馈OK，Leader
      节点就会向所有的follower服务器发送commit消息，即将Leader节点上的数据同步到follower节点之上。

      消息广播模式
           1：在Zookeeper集群中数据副本的传递策略就是采用消息广播模式。Zookeeper中数据副
              本的同步方式与二阶段提交相似但是却又不同。二阶段提交要求协调者必须等待所有的
              参与者全部反馈ACK确认消息后，再发送commit消息，要求所有的参与者要么全部成功
              要么全部失败，二阶段提交会产生严重阻塞问题。

           2：ZAB协议中Leader等待follower的ACK反馈是指“只要半数以上的follower成功反馈
              即可，不需要收到全部follower反馈”

      Zookeeper中消息广播的具体步骤：
           1：客户端发起一个写操作请求
           2：Leader服务器将客户端的request请求你转化为事务proposql提案，同时为每个
              proposal分配一个全局唯一的ID，即ZXID
           3:Leader服务器与每个follower之间都有一个队列，Leader将消息发送到该队列。
           5：follower机器从队列中取出消息处理完（写入本地事务日志中）完毕后，向leader
              服务器发送ack消息。
           6：leader服务器收到半数以上的follower的ack后，即认为可以发送commit消息
           7：leader向所有follower服务器发送commit消息。

      Zookeeper采用Zab协议的核心就是只要有一台服务器提交了proposal就要确保所有的服务器
      最终都能正确提交proposal，这也是CAP/BASE最终实现一致性的一个体现。

      Leader服务器与每个follower之间都有一个单独的队列进行收发消息，使用队列消息可以做到
      异步解耦.Leader和follower之间只要往队列中发送了消息即可，如果使用同步的方式容易引起
      阻塞，性能上要下降很多。

崩溃恢复：
      1：Zookeeper集群中为保证任何所有进程能够有序的顺序执行，只能是Leader服务器接收写
         请求，即使是follower服务器接收到客户端的请求，也会转发到Leader服务器进行处理。
      2:如果Leader服务器发生崩溃，则Zab协议要求Zookeeper集群进行崩溃恢复和Leader服务器
        选举。
      3：Zab协议崩溃恢复要求满足如下两个条件：
            1：确保已经被Leader提交的proposal必须最终被所有的follower服务器提交。
            2：确保丢弃已经被Leader发出的但是没有被提交的proposal。
      5：根据上述要求：
            新选出来的Leader不能包含未提交的proposal，即新选择的Leader必须都是已经提交
         了的proposal的follower节点。同时新选举的Leader节点中含有最高的ZXID,这样的好处
         是可以避免了Leader服务器检查proposal的提交和丢弃工作。

      6：Leader服务器发生崩溃时分为如下场景：
         1：Leader在提出proposal时未提交之前已崩溃，则经过恢复后，新选举的Leader一定不能
            是刚才的Leader，因为Leader存在未提交的proposal.
         2:Leader在发送commit消息之后，崩溃，即消息已经发送到队列中，经过崩溃恢复之后，
            参与选举的follower服务器（刚才崩溃的服务器有可能已经回复执行，也属
            于follower）中有的节点已经消费了队列中所有commit消息，即该follower节点将
            会被选举为最新的Leader，剩下动作就是同步过程。

数据同步：
      1：在Zookeeper集群中新的Leader选举成功之后，Leader会将自身的提交的最大proposal的
         事务ZXID发送给其他follower节点，follower节点会根据Leader的消息进行回退或者数
         据同步操作。最终的目的是保证集群中所有节点的数据副本基本一致。
         数据同步完之后，zookeeper集群如何保证新选举的leader分配的ZXID是全局唯一呢？这个就要从ZXID的设计谈起。 
      2.1 ZXID是一个长度64位的数字，其中低32位是按照数字递增，即每次客户端发起一个
         proposal,低32位的数字简单加1。高32位是leader周期的epoch编号，，每当选举出一个
         新的leader时，新的leader就从本地事物日志中取出ZXID,然后解析出高32位的epoch
         编号，进行加1，再将低32位的全部设置为0。这样就保证了每次新选举的leader后，保证
         了ZXID的唯一性而且是保证递增的。

Zab协议原理：
        Zab协议要求每个Leader都要经历三个阶段，即发现，同步，广播。
          1）发现：即要求zookeeper集群必须选择则出一个leader进程，同时leader会维护一个
                  follower可用列表，将来客户端可以用这些follower节点进行通行。
          2）同步：Leader要负责将本身的数据与follower完成同步，做到多副本存储，这样也是
                  体现了CAP中高可用和分区容错，follower将队列中未处理完的请求消费完成后
                  ，写入本地事务日志中。
          3）广播：Leader可以接受客户端新的proposal，将新的proposal请求广播给follower.

Zookeeper设计目标：
       Zookeeper作为当今最流行的分布式应用协调框架，采用Zab协议的最大目标就是建立一个高可
       用可扩展的分布式数据主备系统。即在任何时刻只要Leader发生宕机，都能保证分布式系统数据
       的可靠性和最终一致性。
</pre>
