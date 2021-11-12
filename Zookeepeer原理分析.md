Zookpeer学习



一、ZAB协议

Zab协议 的全称是 Zookeeper Atomic Broadcast （Zookeeper原子广播）。ZAB协议可以实现的前提ZK集群可以正常对外提供服务。Zookeeper就是根据Za协议来保证数据最终一致性。Zab协议提供两种模式：崩溃恢复和消息广播。当zk集群启动时或者leader服务器出现崩溃时，根据Zab协议，zk集群会进入崩溃恢复模式，选举出新的leader，当选举出新的leader，并且有超过集群一半数量的服务器完成和该leader数据同步的后，zk集群退出崩溃恢复模式，进入消息广播模式，整个zk集群正常对外提供服务。如果此时，加入一台服务器，则这台服务器直接进入崩溃恢复模式，和leader进行数据同步，之后，整个集群正常提供服务。

1、崩溃恢复---数据恢复

Zab协议的消息广播是原子性的，在一般情况下是不会有任何问题的，但是一旦leader出现崩溃的情况或者leader和超过集群数量一半的服务器失去了联系，那么这个leader的存在就没有意义了。此时集群进入崩溃恢复模式，重新选举leader。重新选举leader需要解决以下几个问题：

- 已经处理的消息不能被丢失。
  如果leader接收到客户端请求生成了proposal并发送给了其他follower节点后，在发送commit命令之后，突然就崩溃了，而此时，follower1节点刚好处理完了commit请求，follower2节点还没处理完commit请求。这种情况下，这条消息是不能丢弃的，那么Zab协议就必须保证选举出zxid最大的一个节点为leader，也就是follower1节点，从而保证整个集群在恢复后能从follower1节点恢复数据。
- 已经丢弃的消息不能再次出现。
  如果leader接收到客户端请求生成了proposal之后就崩溃了，因为其他的节点没有接受到请求，那么在重新选举leader之后，这条消息是被跳过的。而原leader节点恢复后注册成了follower节点，但是发现自己的状态和整个集群的状态不一致，那么此时它需要删除。

2、消息广播---原子广播

      消息广播实际上是一个简化版的2PC的过程：

      （1）leader接收到客户端请求后，将请求转化成一个事务 proposal 提议并先持久化在自己的服务器上，并且分配一个64位的自增的全局唯一的id，叫做zxid。

	  （2）因为leader和每个follower之间都维护着一个FIFO的队列。leader将proposal处理完后，就会将proposal发送到各个follower节点上。

	  （3）follower节点接收到proposal后，会以事物的方式进行持久化，成功之后，会向leader发送一个响应，也就是ack。

       （4）当leader接收到超过集群数量一半的follower的ack之后，就认为这个消息已经处理完成。此时，leader就可以发送commit命令。

	   （5）leader向所有的follower节点发送commit命令，同时自身也进行事物提交，即将此次消息写入内存，并向发起响应。

        这里可以看出，leader在接收到半数以上的ack之后就会发送commit 命令，之后，不再关注follower节点是不是commit事物成功，从而得出，zk保证的是数据最终一致性。

3、选举机制

选举条件：

     3.1、zxid最新的选择leader，事物id，自增，处理 proposal的时候生成。

一个 zxid 是64位，高 32 是纪元（epoch）编号，每经过一次 leader 选举产生一个新的 leader，新 leader 会将 epoch 号 +1。低 32 位是消息计数器，每接收到一条消息这个值 +1，新 leader 选举后这个值重置为 0。

      3.2、myid大的选择leader。

注意：3台机器的集群中，如果先启动的两台机器已经完成了选举工作，那么第三台启动后再不会参与竞争选举，即使它的myid是更大的或者zxid最新，而是会将zxid同步成leader的zxid，源码位置：Follower.java（followLeader()）。

总结：

- 新选举出来的 Leader 不能包含未提交的 Proposal 。
- 新选举的 Leader 节点中含有最大的 zxid。

4、2PC

4.1、预提交：leader本地自己先持久化数据，并将请求发送给所有的follower，其他的follower收到命令后，也进行持久化，成功后，向leader响应一个ack。如果这一步发送的时候，发现无法连接到其他的节点，那么说明集群服务基本瘫痪，leader会自己shurdown，因为它自存在的已经没有意义了。

4.2、收到回复（源码位置：Leader.java（processAck()）：leader接收follower的ack，如果成功的接收到超过半数的ack，那就说明消息处理完成，此时，leader需要再次发送commit命令给所有的follower，告诉所有的follower本次事务可以进行最终的提交了。但是，却不再关注follower是否最终提交成功。

- 过半机制： 3台机器的集群中，如果两台成功的回复了ack，当然也包含leader自身的ack，那么leader就认为请求已经处理OK。源码位置QuorumMaj.java（containsQuorum()）

4.3、提交/回滚：follower收到commit命令后，对本次事务进行最终的提交，也就是将数据写入内存。但是ZK中没有回滚机制。

- 同步机制：所有的zk服务器，都应该以leader服务器的状态和数据为准。

5、zk集群 处理请求的过程

5.1、当一个服zk服务器leader接收到写请求时：

	    （1）leader接收到客户端请求后，将请求转化成一个事务 proposal 提议并先持久化在自己的服务器上，并且分配一个64位的自增的全局唯一的id，叫做zxid。

	  （2）因为leader和每个follower之间都维护着一个FIFO的队列。leader将proposal处理完后，就会将proposal发送到各个follower节点上。

	  （3）follower节点接收到proposal后，会以事物的方式进行持久化，成功之后，会向leader发送一个响应，也就是ack。

       （4）当leader接收到超过集群数量一半的follower的ack之后，就认为这个消息已经处理完成。此时，leader就可以发送commit命令。

	   （5）leader向所有的follower节点发送commit命令，同时自身也进行事物提交，即将此次消息写入内存，并向发起响应。

5.2、当一个服zk服务器follower接收到写请求时：

（1）follower会将请求直接发送给leader，因为follower没有写请求的权限。之后的处理流程和5.1一样。

6、Zab 节点有三种状态：

- Following：当前节点是跟随者，服从 Leader 节点的命令。
- Leading：当前节点是 Leader，负责协调事务。
- Election/Looking：节点处于选举状态，正在寻找 Leader。
- Observing ： Zookeeper 引入 Observer ，Observer 不参与选举，是只读节点，跟 Zab 协议没有关系。




