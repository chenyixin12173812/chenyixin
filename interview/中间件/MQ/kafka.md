# 0 各大MQ对比

 主流的消息队列中间件主要有，**Kafka、ActiveMQ、RabbitMQ、RocketMQ** 等 

 **ActiveMQ**和**RabbitMQ**这两着因为吞吐量还有**GitHub**的社区活跃度的原因，在各大互联网公司都已经基本上绝迹了，业务体量一般的公司会是有在用的，但是越来越多的公司更青睐**RocketMQ**这样的消息中间件了 

**Kafka**和**RocketMQ**一直在各自擅长的领域发光发亮，不过写这篇文章的时候我问了蚂蚁金服，字节跳动和美团的朋友，好像大家用的都有点不一样，应该都是各自的中间件，可能做过修改，也可能是**自研**的，大多**没有开源**。

就像我们公司就是是基于**Kafka**和**RocketMQ**两者的优点自研的消息队列中间件，吞吐量、可靠性、时效性等都很可观

![img](https://pic3.zhimg.com/80/v2-8234b69f5e46550bc5d95db96251d8b6_720w.jpg)

就拿**吞吐量**来说，早期比较活跃的**ActiveMQ** 和**RabbitMQ**基本上不是后两者的对手了，在现在这样大数据的年代**吞吐量是真的很重要**

至于**RocketMQ**（阿里开源的），git活跃度还可以。基本上你push了自己的bug确认了有问题都有阿里大佬跟你试试解答并修复的，我个人推荐的也是这个，他的架构设计部分跟同样是阿里开源的一个**RPC**框架是真的很像（**Dubbo**）可能是因为师出同门的原因吧。



**Kafka**我放到最后说，你们也应该知道了，压轴的这是个大哥，大数据领域，公司的日志采集，实时计算等场景，都离不开他的身影，他基本上算得上是**世界范围级别的消息队列标杆**了



# 1.消息重复产生

 1：强行kill线程，导致消费后的数据，offset没有提交（消费系统宕机、重启等）。 
 2：设置offset为自动提交，关闭kafka时，如果在close之前，调用 consumer.unsubscribe() 则有可能部分offset没提交，下次重启会重复消费。例如：

try {
    consumer.unsubscribe();
} catch (Exception e) {
}

try {
    consumer.close();
} catch (Exception e) {
}
上面代码会导致部分offset没提交，下次启动时会重复消费。 
 3:（重复消费最常见的原因）：消费后的数据，当offset还没有提交时，partition就断开连接。比如，通常会遇到消费的数据，处理很耗时，导致超过了Kafka的session timeout时间（0.10.x版本默认是30秒），那么就会re-blance重平衡，此时有一定几率offset没提交，会导致重平衡后重复消费。 
  4：当消费者重新分配partition的时候，可能出现从头开始消费的情况，导致重发问题。 
 5：当消费者消费的速度很慢的时候，可能在一个session周期内还未完成，导致心跳机制检测报告出问题。

6 ：网路不稳定，生成者不知道配置是否成功发送，重试。



# 2 [ 那么使用消息队列会带来什么问题?考虑过这些问题吗?](https://github.com/chenyixin12173812/JavaGuide/blob/master/docs/essential-content-for-interview/PreparingForInterview/美团面试常见问题总结.md#12-那么使用消息队列会带来什么问题考虑过这些问题吗) 

- **系统可用性降低：** 系统可用性在某种程度上降低，为什么这样说呢？在加入 MQ 之前，你不用考虑消息丢失或者说 MQ 挂掉等等的情况，但是，引入 MQ 之后你就需要去考虑了！
- **系统复杂性提高：** 加入 MQ 之后，你需要保证消息没有被重复消费、处理消息丢失的情况、保证消息传递的顺序性等等问题！
- **一致性问题：** 我上面讲了消息队列可以实现异步，消息队列带来的异步确实可以提高系统响应速度。但是，万一消息的真正消费者并没有正确消费消息怎么办？这样就会导致数据不一致的情况了!



# 3.Kafka 的设计时什么样的呢？

Kafka 将消息以 topic 为单位进行归纳

将向 Kafka topic 发布消息的程序成为 producers.

将预订 topics 并消费消息的程序成为 consumer.

Kafka 以集群的方式运行，可以由一个或多个服务组成，每个服务叫做一个 broker.

producers 通过网络将消息发送到 Kafka 集群，集群向消费者提供消息

# 4 .数据传输的事物定义有哪三种？



数据传输的事务定义通常有以下三种级别：

（1）最多一次: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输

（2）最少一次: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.

（3）精确的一次（Exactly once）: 不会漏传输也不会重复传输,每个消息都传输被一次而

且仅仅被传输一次，这是大家所期望的



# 5 MQ作用

1 异步

2 解耦

3 消峰

4 冗余

5 顺序保证

6 缓存

7 可扩展

8 可恢复性

# 6 Kafka 判断一个节点是否还活着有那两个条件？

（1）节点必须可以维护和 ZooKeeper 的连接，Zookeeper 通过心跳机制检查每个节点的连

接

（2）如果节点是个 follower,他必须能及时的同步 leader 的写操作，延时不能太久

# 7 producer 是否直接将数据发送到 broker 的 leader(主节点)？

producer 直接将数据发送到 broker 的 leader(主节点)，不需要在多个节点进行分发，为了帮助 producer 做到这点，所有的 Kafka 节点都可以及时的告知:哪些节点是活动的，目标topic 目标分区的 leader 在哪。这样 producer 就可以直接将消息发送到目的地了.

# 8、Kafa consumer 是否可以消费指定分区消息？

Kafa consumer 消费消息时，向 broker 发出"fetch"请求去消费特定分区的消息，consume指定消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息，customer 拥有了 offset 的控制权，可以向后回滚去重新消费之前的消息，这是很有意义。

# 9、Kafka 消息是采用 Pull 模式，还是 Push 模式？****producer 将消息推送到 broker，consumer 从 broker 拉取消息**

一些消息系统比如 Scribe 和 **Apache Flume 采用了 push 模式，将消息推送到下游的consumer**。这样做有好处也有坏处：由 broker 决定消息推送的速率，对于不同消费速率的

consumer 就不太好处理了。消息系统都致力于让 consumer 以最大的速率最快速的消费消

息，但不幸的是，push 模式下，当 broker 推送的速率远大于 consumer 消费的速率时，

consumer 恐怕就要崩溃了。最终 Kafka 还是选取了传统的 pull 模式

Pull 模式的另外一个好处是 consumer 可以自主决定是否批量的从 broker 拉取数据。Push

模式必须在不知道下游 consumer 消费能力和消费策略的情况下决定是立即推送每条消息还

是缓存之后批量推送。如果为了避免 consumer 崩溃而采用较低的推送速率，将可能导致一

次只推送较少的消息而造成浪费。Pull 模式下，consumer 就可以根据自己的消费能力去决

定这些策略

Pull 有个缺点是，如果 broker 没有可供消费的消息，将导致 consumer 不断在循环中轮询，

直到新消息到 t 达。为了避免这点，Kafka 有个参数可以让 consumer 阻塞知道新消息到达

(当然也可以阻塞知道消息的数量达到某个特定的量这样就可以批量发

**7.Kafka 存储在硬盘上的消息格式是什么？**

消息由一个固定长度的头部和可变长度的字节数组组成。头部包含了一个版本号和 CRC32

校验码。

消息长度: 4 bytes (value: 1+4+n)

版本号: 1 byte

CRC 校验码: 4 bytes

具体的消息: n bytes

**8.Kafka 高效文件存储设计特点：**

(1).Kafka 把 topic 中一个 parition 大文件分成多个小文件段，通过多个小文件段，就容易定

期清除或删除已经消费完文件，减少磁盘占用。

(2).通过索引信息可以快速定位 message 和确定 response 的最大大小。

(3).通过 index 元数据全部映射到 memory，可以避免 segment file 的 IO 磁盘操作。

(4).通过索引文件稀疏存储，可以大幅降低 index 文件元数据占用空间大小。

**9.Kafka 与传统消息系统之间有三个关键区别**

(1).Kafka 持久化日志，这些日志可以被重复读取和无限期保留

(2).Kafka 是一个分布式系统：它以集群的方式运行，可以灵活伸缩，在内部通过复制数据

提升容错能力和高可用性

(3).Kafka 支持实时的流式处理

**10.Kafka 创建 Topic 时如何将分区放置到不同的 Broker 中**

副本因子不能大于 Broker 的个数；

第一个分区（编号为 0）的第一个副本放置位置是随机从 brokerList 选择的；

其他分区的第一个副本放置位置相对于第 0 个分区依次往后移。也就是如果我们有 5 个

Broker，5 个分区，假设第一个分区放在第四个 Broker 上，那么第二个分区将会放在第五

个 Broker 上；第三个分区将会放在第一个 Broker 上；第四个分区将会放在第二个

Broker 上，依次类推；

剩余的副本相对于第一个副本放置位置其实是由 nextReplicaShift 决定的，而这个数也是

随机产生的

**11.Kafka 新建的分区会在哪个目录下创建**

在启动 Kafka 集群之前，我们需要配置好 log.dirs 参数，其值是 Kafka 数据的存放目录，

这个参数可以配置多个目录，目录之间使用逗号分隔，通常这些目录是分布在不同的磁盘

上用于提高读写性能。

当然我们也可以配置 log.dir 参数，含义一样。只需要设置其中一个即可。

如果 log.dirs 参数只配置了一个目录，那么分配到各个 Broker 上的分区肯定只能在这个

目录下创建文件夹用于存放数据。

但是如果 log.dirs 参数配置了多个目录，那么 Kafka 会在哪个文件夹中创建分区目录呢？

答案是：Kafka 会在含有分区目录最少的文件夹中创建新的分区目录，分区目录名为 Topic

名+分区 ID。注意，是分区文件夹总数最少的目录，而不是磁盘使用量最少的目录！也就

是说，如果你给 log.dirs 参数新增了一个新的磁盘，新的分区目录肯定是先在这个新的磁

盘上创建直到这个新的磁盘目录拥有的分区目录不是最少为止。

**12.partition 的数据如何保存到硬盘**

topic 中的多个 partition 以文件夹的形式保存到 broker，每个分区序号从 0 递增，

且消息有序

Partition 文件下有多个 segment（xxx.index，xxx.log）

segment 文件里的 大小和配置文件大小一致可以根据要求修改 默认为 1g

如果大小大于 1g 时，会滚动一个新的 segment 并且以上一个 segment 最后一条消息的偏移

量命名

**13.kafka 的 ack 机制**

request.required.acks 有三个值 0 1 -1

0:生产者不会等待 broker 的 ack，这个延迟最低但是存储的保证最弱当 server 挂掉的时候

就会丢数据

1：服务端会等待 ack 值 leader 副本确认接收到消息后发送 ack 但是如果 leader 挂掉后他

不确保是否复制完成新 leader 也会导致数据丢失

-1：同样在 1 的基础上 服务端会等所有的 follower 的副本受到数据后才会受到 leader 发出

的 ack，这样数据不会丢失

**14.Kafka 的消费者如何消费数据**

消费者每次消费数据的时候，消费者都会记录消费的物理偏移量（offset）的位置

等到下次消费时，他会接着上次位置继续消费

**15.消费者负载均衡策略**

一个消费者组中的一个分片对应一个消费者成员，他能保证每个消费者成员都能访问，如

果组中成员太多会有空闲的成员

**16.数据有序**

一个消费者组里它的内部是有序的

消费者组与消费者组之间是无序的

**17.kafaka 生产数据时数据的分组策略**

生产者决定数据产生到集群的哪个 partition 中

每一条消息都是以（key，value）格式

Key 是由生产者发送数据传入

所以生产者（key）决定了数据产生到集群的哪个 partition



Kafka的用途有哪些？使用场景如何？
Kafka中的ISR、AR又代表什么？ISR的伸缩又指什么
Kafka中的HW、LEO、LSO、LW等分别代表什么？
Kafka中是怎么体现消息顺序性的？
Kafka中的分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？
Kafka生产者客户端的整体结构是什么样子的？
Kafka生产者客户端中使用了几个线程来处理？分别是什么？
Kafka的旧版Scala的消费者客户端的设计有什么缺陷？
“消费组中的消费者个数如果超过topic的分区，那么就会有消费者消费不到数据”这句话是否正确？如果正确，那么有没有什么hack的手段？
消费者提交消费位移时提交的是当前消费到的最新消息的offset还是offset+1?
有哪些情形会造成重复消费？
那些情景下会造成消息漏消费？
KafkaConsumer是非线程安全的，那么怎么样实现多线程消费？
简述消费者与消费组之间的关系
当你使用kafka-topics.sh创建（删除）了一个topic之后，Kafka背后会执行什么逻辑？
topic的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？
topic的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？
创建topic时如何选择合适的分区数？
Kafka目前有那些内部topic，它们都有什么特征？各自的作用又是什么？
优先副本是什么？它有什么特殊的作用？
Kafka有哪几处地方有分区分配的概念？简述大致的过程及原理
简述Kafka的日志目录结构
Kafka中有那些索引文件？
如果我指定了一个offset，Kafka怎么查找到对应的消息？
如果我指定了一个timestamp，Kafka怎么查找到对应的消息？
聊一聊你对Kafka的Log Retention的理解
聊一聊你对Kafka的Log Compaction的理解
聊一聊你对Kafka底层存储的理解（页缓存、内核层、块层、设备层）
聊一聊Kafka的延时操作的原理
聊一聊Kafka控制器的作用
消费再均衡的原理是什么？（提示：消费者协调器和消费组协调器）
Kafka中的幂等是怎么实现的
Kafka中的事务是怎么实现的（这题我去面试6加被问4次，照着答案念也要念十几分钟，面试官简直凑不要脸）
Kafka中有那些地方需要选举？这些地方的选举策略又有哪些？
失效副本是指什么？有那些应对措施？
多副本下，各个副本中的HW和LEO的演变过程
为什么Kafka不支持读写分离？
Kafka在可靠性方面做了哪些改进？（HW, LeaderEpoch）
Kafka中怎么实现死信队列和重试队列？
Kafka中的延迟队列怎么实现（这题被问的比事务那题还要多！！！听说你会Kafka，那你说说延迟队列怎么实现？）
Kafka中怎么做消息审计？
Kafka中怎么做消息轨迹？
Kafka中有那些配置参数比较有意思？聊一聊你的看法
Kafka中有那些命名比较有意思？聊一聊你的看法
Kafka有哪些指标需要着重关注？
怎么计算Lag？(注意read_uncommitted和read_committed状态下的不同)
Kafka的那些设计让它有如此高的性能？
Kafka有什么优缺点？
还用过什么同质类的其它产品，与Kafka相比有什么优缺点？
为什么选择Kafka?
在使用Kafka的过程中遇到过什么困难？怎么解决的？
怎么样才能确保Kafka极大程度上的可靠性？
聊一聊你对Kafka生态的理解作者：应寒

 13、通过异步处理提高系统性能（削峰、减少响应所需时间） 

 14、降低系统耦合性。 

 15、使用消息队列带来的一些问题。 

 16、JMS两种消息模型。 

 17、JMS 五种不同的消息正文格式。 



# 30 kafka为什么这么快

 分区并行、ISR机制、顺序写入、页缓存、高效序列化等等，零拷贝当然也是其中之一。由于Kafka的消息存储涉及到海量数据读写，所以利用零拷贝能够显著地降低延迟，提高效率。 

 https://cloud.tencent.com/developer/article/1639123 