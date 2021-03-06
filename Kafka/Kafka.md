# Kafka

**参考文档**

[Kafka 概述快速入门 - CSDN博客](https://achang.blog.csdn.net/article/details/121307740)

[Kafka 基础 - CSDN博客](https://achang.blog.csdn.net/article/details/123054073)

[Kafka 3.x 教程](https://www.bilibili.com/video/BV1vr4y1677k?p=1)

&emsp;

## 定义

Kafka 的目标是实现一个为处理实时数据提供一个统一、高吞吐、低延迟的平台。是分布式**发布-订阅**消息系统，是一个分布式的，可划分的，冗余备份的持久性的日志服务。发布/订阅指的是消息的发布者不会将消息直接发送给特定的订阅者，而是将发布的消息分为不同的类别，订阅者只接收感兴趣的消息。

&emsp;

## Kafka 的基础架构

![外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传imgsxnfh3UX1636797115275C/Users/PePe/AppData/Roaming/Typora/typorauserimages/image20211113173946565png](https://img-blog.csdnimg.cn/797194be47b947bf989b01a87b5b6e78.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

1. Producer：消息生产者，就是向 Kafka broker 发消息的客户端

2. Consumer：消息消费者，向 Kafka broker 取消息的客户端

3. Consumer Group (CG)：消费组，由多个消费者组成。 消费组内每个消费者负责消费不同分区的数据，一个分区只能由组内一个消费者消费；消费组之间互不影响，一个消息可以被多个消费组消费，但是同一个消费组中只能有一个消费者去消费。所有的消费者都属于某个消费组，即消费者组是逻辑上的一个订阅者

4. Broker：一台 Kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker 可以容纳多个 topic

5. Topic：消息主题，生产者和消费者面向的都是同一个消费主题

6. Partition：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker（即服务器）上，一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列；

7. Replica：副本，为保证集群中的某个节点发生故障时，该节点上的 partition 数据不丢失，且 Kafka 仍然能够继续工作，Kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本，包括一个 leader 和若干个 follower。

8. Leader：每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是 leader。

9. Follower：每个分区多个副本中的“从”，实时从 leader 中同步数据，保持和 leader 数据的同步。leader 发生故障时，某个 follower 会成为新的 leader。
   
   &emsp;

## 生产者

### 发送原理

在消息发送的过程中，涉及到了两个线程—— `main` 线程和 `Sender` 线程。在 `main` 线程
中创建了一个双端队列 `RecordAccumulator`。`main` 线程将消息发送给 `RecordAccumulator`， `Sender` 线程不断从 `RecordAccumulator` 中拉取消息发送到 `Kafka Broker`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/06a6dd353ee14d98bc5788bd8cc87898.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

* 同步发送：一定是逐条发送，第一条响应到达后才会请求第二条。

* 异步发送：可以发送一条，也可以批量发送多条，特性是不需要等第一次响应就可以发送第二次。

&emsp;

### 分区

![在这里插入图片描述](https://img-blog.csdnimg.cn/9e1b1c96fb664a588544bbb0c26555e0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

分区包括以下几个优点：

* 便于合理使用存储资源，每个 `Partition` 在一个 `Broker` 上存储，可以把海量的数据按照分区切割成一块一块数据存储在多台 `Broker` 上。合理控制分区的任务，可以实现**负载均衡**的效果，让存储更加灵活。（假设一台服务器能存储的硬盘资源是10T，另一个是100T，分区后就可以根据每台服务器的存储能力动态分配）

* 提高并行度，生产者可以以分区为单位发送数据，消费者可以以分区为单位进行消费数据。

![](/Users/yuhangliu/Desktop/Screen%20Shot%202022-04-30%20at%2015.36.48.png)

分区的策略如下：

* 指明 `partition` 的情况下，直接将指明的值作为 `partition` 值。

* 没有指明 `partition` 值但是传递的消息体有 `key` 的情况下，`key` 的哈希值与 `topic`的 `partition` 数进行取余得到 `partition` 值。

* 既没有 `partition` 值又没有 `key` 值的情况下，Kafka 采用 Sticky Partition (黏性分区器)，会随机选择一个分区，并尽可能一直使用该分区，待该分区的 `batch` 已满或者已完成，Kafka 再随机一个分区进行使用 (和上一次的分区不同)。

&emsp;

### 提升吞吐量的参数

**batch.size** 控制每一批次拉取的量。

**linger.ms** 控制等待时间，最多等待这么久就发送一次，即使拉取量没达到阈值。

**compression.type** 采用数据压缩。

**RecordAccumulator** 缓冲区大小。

&emsp;

### 数据可靠性

#### ACK 应答原理

根据上图发送原理可以知道，应答 ACK 有三种不同的方式：

* **ACK 是 0 的时候，代表生产者发送过来数据，不需要等数据落盘应答就可以继续发送**。这种情况下，如果生产者把消息发送给 `Leader`，而 `Leader` 在同一时刻出现故障，还没有来得及和 `Follower` 同步，这时会出现数据丢失的情况。

* **ACK 是 1 的时候，代表生产者发送过来的数据， Leader 收到并且落盘之后即返回应答，不需要等到同步完成**。这种情况下，应答完成后，`Leader` 还没来的及同步消息就出现故障，新的被选举出来的 `Leader` 并没有该信息，同时因为应答已经发送出去，生产者不会再发送该消息，造成数据丢失。

* **ACK 是 -1 的时候，代表生产者发送过来的数据，Leader 和 ISR 队列里面所有的节点收齐数据后才会应答**。

&emsp;

![[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-lQVkkjrQ-1636885037649)(C:/Users/PePe/AppData/Roaming/Typora/typora-user-images/image-20211114142912652.png)]](https://img-blog.csdnimg.cn/3fc9e15ba56b47368507e31cc53ca7ad.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

上图是针对 ACK 是 -1 的情况。

&emsp;

第三种显然是最保险的，但是会有一个问题：`Leader` 收到数据，所有 `Follower` 都开始同步数据，但有一个 `Follower` ，因为某种故障，迟迟不能与 `Leader`进行同步，就会导致迟迟不能发送 ACK，那这个问题怎么解决呢? 实际上，`Leader`维护了一个动态的 `in-sync replica set` (ISR)，意为和 `Leader` 保持同步的 Follower + Leader 集合。如果一个 `Follower` 长时间未向 `Leader` 发送通信请求或同步数据，则该 `Follower` 将被踢出 ISR。该时间阈值由 `replica.lag.time.max.ms` 参数设定，默认30秒。**根据这种机制，为了确保可靠性，也需要满足该分区的副本个数大于等于2**。

&emsp;

总结一下，上面三种方式，可靠性从上至下逐渐提升，效率逐渐降低。在生产环境中，第一种很少使用，第二种一般用于传输普通日志，允许个别数据丢失，第三种适用于对可靠性高要求的场景。

&emsp;

#### 数据去重

假设采用上述第三种 ACK 应答方式，当完成所有的同步之后，`Leader` 出现故障，应答没有成功发送。当新的 Leader 被选拔出来后，因为没有收到 ACK，生产者会发送同样的数据，已经同步的数据会被再次同步。

&emsp;

数据传递的语义有以下几种：

* **至少一次 (At Least Once)** = ACK 级别设置为 -1 + 分区副本大于等于 2 + ISR 里应答的最小副本数量大于等于 2。

* **最多一次 (At Most Once)** = ACK 级别设置为 0。

* **精确一次 (Exactly Once)** = 幂等性 + 至少一次。要求数据既不能重复也不丢失，可以通过幂等性和事务实现。

&emsp;

幂等性就是指生产者不论向 `Broker` 发送多少次重复数据，`Broker` 端都只会持久化一条，保证了不重复。重复数据的判断标准：具有 `<PID, Partition, SeqNumber>` 相同主键的消息提交时，`Broker` 只会持久化一条。其中 `PID` 是 Kafka 每次重启都会分配一个新的，`Partition` 表示分区号，`Sequence Number` 是单调自增的，所以**幂等性只能保证的是在单分区单会话内不重复**。在生产环境里，开启幂等性的参数 `enable.idempotence` 默认为 true，false 关闭。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7852dd363c534be6a288136b17e0d6c3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

&emsp;

为了确保多次会话内数据仍然不重复，需要使用事务。在 Kafka 中，开启事务必须开启幂等性。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d807aef6755d47179849a4ef0b3e2943.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

通过事务就可以确保即使客户端出现故障重启后也可以继续正常工作。

&emsp;

#### 数据顺序

在单一分区内，数据是有序的；对于多分区，分区与分区间的数据是无序的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d1f32c9d78294bdab8d5c49d755500be.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

在 Kafka1.x 以后，启用幂等后，Kafka 服务端会缓存生产者发来的最近5个 request 的元数据，故无论如何，都可以保证最近5个 request 的数据都是有序的。

&emsp;

## Broker 工作流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/3a92435a435c4ebcb1f2ee0197e991ad.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)



### Kafka 的副本

副本的作用是提高数据可靠性。Kafka 默认副本 1 个，生产环境一般配置为 2 个，保证数据可靠性；太多副本会增加磁盘存储空间，增加网络上数据传输，降低效率。在 Kafka 中，副本分为 `Leader` 和 `Follower`。Kafka 生产者只会把数据发往 `Leader`，然后 `Follower` 找 `Leader` 进行同步数据。Kafka 分区中的所有副本统称为 `AR (Assigned Repllicas)`。AR = ISR + OSR，其中：

* ISR 表示和 `Leader` 保持同步的 `Follower` 集合。如果 `Follower` 长时间未向 `Leader` 发送通信请求或同步数据，则该 `Follower` 将被踢出 ISR。该时间阈值由 `replica.lag.time.max.ms` 参数设定，默认 30s。`Leader` 发生故障之后，就会从 ISR 中选举新的 `Leader`。

* OSR 表示 `Follower` 与 `Leader` 副本同步时，延迟过多的副本。

&emsp;

#### Leader 选举

Kafka 集群中有一个 `broker` 的 `Controller` 会被选举为 `Controller Leader`，负责管理集群 `broker` 的上下线，所有 `topic` 的分区副本分配和 `Leader` 选举等工作。当出现故障后，选举原则以在 ISR 中存活为前提，按照 AR 中排在前面的优先。`Controller` 的信息同步工作是依赖于 `Zookeeper` 的。

&emsp;

#### 故障处理

`Follower` 出现故障后如下图处理。

![在这里插入图片描述](https://img-blog.csdnimg.cn/fded89821c0b4281a6de6d6560576681.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

&emsp;

`Leader` 出现故障后如下图处理。

![在这里插入图片描述](https://img-blog.csdnimg.cn/03b09bc0f2d2456c805c733c889942c0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

&emsp;

### 文件存储

`Topic` 是逻辑上的概念，而 `partition` 是物理上的概念，每个partition对应于一个 Log 文件，该 Log 文件中存储的就是生产者生产的数据。生产者生产的数据会被不断追加到该 Log 文件末端，为防止 Log 文件过大导致数据定位效率低下，Kafka采取了分片和索引机制，将每个 `partition` 分为多个 `segment` 。每个 `segment` 包括一个 `.index` 偏移量索引文件，一个 `.log` 日志文件和 `.timeindex` 时间戳索引等文件。这些文件位于一个文件夹下，该文件夹的命名规则为 topic名称+分区序号。

![[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-2ZmlCBF2-1636885037638)(C:/Users/PePe/AppData/Roaming/Typora/typora-user-images/image-20211114142618525.png)]](https://img-blog.csdnimg.cn/ef1b6568d17043c5a0e28a026500c422.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

**在 Kafka 中，默认的日志保存时间为 7 天**。

&emsp;

### 高效读写数据

1. Kafka 本身是分布式集群，可以采用分区技术，并行度高。

2. 读数据采用稀疏索引，可以快速定位要消费的数据。

3. 顺序写磁盘。Kafka 的生产者生产数据，要写入到 Log 文件中，写的过程是一直追加到文件末端，为顺序写。

4. 零拷贝。Kafka 的数据加工处理操作交由 Kafka 生产者和 Kafka 消费者处理。Kafka `Broker` 应用层不关心存储的数据，所以就不用走应用层，传输效率高。

5. `PageCache` 页缓存。Kafka 重度依赖底层操作系统提供的 `PageCache` 功能。当上层有写操作时，操作系统只是将数据写入 `PageCache`。当读操作发生时，先从 `PageCache`中查找，如果找不到，再去磁盘中读取。实际上 `PageCache` 是把尽可能多的空闲内存
   都当做了磁盘缓存来使用。

&emsp;

## 消费者

### Kafka 消费方式

消息队列常见的两种消费方式有：

* pull (拉) 模式 -- 消费者主动：消费者采用从 `broker` 中主动拉取数据。**Kafka 采用这种方式**。拉模式不足之处是，如果 Kafka 没有数据，消费者可能会陷入循环中，一直返回空数据。针对这一点，Kafka 的消费者在消费数据时会传入一个时长参数 `timeout`，如果当前没有数据可供消费，消费者会等待一段时间后再返回，这段时长即为 `timeout`。

* push (推) 模式 -- `broker` 主动：由 `broker` 决定消息发送速率，Kafka 没有采用这种方式，因为很难适应所有消费者的消费速率。

&emsp;

### 消费者工作流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/b3d26e76be0d42bfaa8fc690b8e57443.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

&emsp;

### 消费者组

`Consumer Group (CG)`：消费者组，由多个消费者组成。形成一个消费者组的条件，是所有消费者的 `groupId` 相同。

* 消费者组内每个消费者负责消费不同分区的数据，**每一个分区只能由一个组内消费者消费**。  

* **消费者组之间互不影响**。所有的消费者都属于某个消费者组，即**消费者组是逻辑上的一个订阅者**。

* 如果消费者组内有过多的消费者，超过分区数量，则一部分消费者就会空闲，不接收任何消息。

![](https://img-blog.csdnimg.cn/8cae27e6367046d6ae311a58cd4d516c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

&emsp;

### 分区消费策略

一个消费者组中有多个消费者组成，一个 `topic` 有多个 `partition` 组成，现在的问题是，到底由哪个消费者来消费哪个 `partition` 的数据？Kafka 有四种主流的分区分配策略: Range、RoundRobin、Sticky、CooperativeSticky。可以通过配置参数`partition.assignment.strategy`，修改分区的分配策略。默认策略是 Range+ CooperativeSticky。Kafka 可以同时使用多个分区分配策略。

#### Range 策略

![](https://img-blog.csdnimg.cn/88366a4e2c514ef8ab13817675866086.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

#### RoundRobin 策略

![](https://img-blog.csdnimg.cn/fb57a06f0dc749189019d089ce072740.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

#### Sticky 策略

粘性分区可以理解为分配的结果带有“粘性的”。即在执行一次新的分配之前，考虑上一次分配的结果，尽量少的调整分配的变动，可以节省大量的开销。粘性分区是 Kafka 从 0.11.x 版本开始引入这种分配策略，首先会尽量均衡的放置分区到消费者上面，在出现同一消费者组内消费者出现问题的时候，会尽量保持原有分配的分区不变化。

&emsp;

### OffSet

由于消费者在消费过程中可能会出现断电宕机等故障，待其恢复后，需要从故障前的位置的继续消费，所以消费者需要实时记录自己消费到了哪个 offset，以便故障恢复后继续消费。Kafka 0.9 版本之前，消费者默认将 offset 保存在 Zookeeper 中，从 0.9 版本开始，消费者默认将 offset 保存在 Kafka 一个内置的 topic 中，该 topic 为`__consumer_offsets`。

&emsp;

### 消费者事务

首先先来看一下什么是重复消费和漏消费。

![在这里插入图片描述](https://img-blog.csdnimg.cn/16786c63aa7847dead898fbbcfacaeb0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Zi_5piM5Zac5qyi5ZCD6buE5qGD,size_20,color_FFFFFF,t_70,g_se,x_16)

如果想完成消费者端的精准一次性消费，那么需要 Kafka 消费端将消费过程和提交 offset 过程做原子绑定。

&emsp;

### 如何提升吞吐量

1. 如果是 Kafka 消费能力不足，则可以考虑增加 `topic` 的分区数，并且同时提升消费组的消费者数量，消费者数 = 分区数 (两者缺一不可) 。

2. 如果是下游的数据处理不及时，可以提高每批次拉取的数量。批次拉取数据过少 (拉取数据/处理时间 < 生产速度)，使处理的数据小于生产的数据，也会造成数据积压。


