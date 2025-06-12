## 启动kafka线程（代码）

```linux
bin/kafka-server-start.sh config/server.properties &
```

 

## MQ队列概述

MQ队列是消息队列，是保存消息的一个容器，本质是一个队列



​                 |———————————————————————|

[生产者]  ---->   |[消息3] [消息2] [消息1]|   ---->    [消费者]
                 |———————————————————————|

## 消息队列使用场景

异步处理

减少请求响应时间，实现非核心流程异步化，提高系统相应性能

应用解耦

生产者与消费者无需直接交互。

流量削峰

应对流量突增，提高系统稳定性。

日志处理

持久化存储，防止数据丢失，提高系统扩展性 

顺序保证

大部分消息队列本身就排序的，并且保证数据会按照特定的顺序来处理



## kafka概述

### kafka是什么

kafka是一种高吞吐量的分布式发布订阅消息系统，由Java和Scala（可以将经过编译的文件也放到jvm中运行 ）

### kafka主要特性

消息的持久化，这种结构对于即使以TB的消息存储也能保持长时间的稳定性能

高吞吐量，非常普通的硬件kafka也能支持每秒数百万的消息

支持通过kafka服务器和消费机集群来分区消息

### kafka架构

备份放在其他机器上永不丢失

<img src="C:\Users\陈宣玮\AppData\Roaming\Typora\typora-user-images\image-20250307154754653.png" alt="image-20250307154754653" style="zoom:80%;" />

- produce：消息生产者，像kafka broker发消息的客户端
- consumer：消息消费者，像kafka broker取消息的客户端
- topic：理解为一个队列
- Consumer Group(消费者组）：是kafka用来实现一个topic消息的广播（发送给全体消费者）和单播（发送给任意一个消费者）
- broker：一台服务器就是一个broker，一个集群由多个broker，一个broker容纳多个topic
- partition：为了实现扩展性，一个非常大的topic 可以分布到多个broker（即服务器）上，一个topic 可以分为多个partition，每个partition是一个有序的队列，kafka只保证按一个partition中的顺序将消息发给consumer
- Offset ：kafka的存储文件都是按照offset.kafka来命名，用offset做名字的好处是方便查找。

### zookeeper在kafka集群中的作用

管理和协调kafka集群中的各个组件，确保集群中的高可用性和一致性

#### 集群成员管理

1. Broker注册和发现：每个Kafka Broker在启动时会向ZooKeeper 注册自己的信息
2. Broker状态监控：（zookeeper中的统一集群管理）

#### 分区分配和领导选举

1. 分区分配：每个主题可以分为多个分区，，确保每个分区都有一个leader和follower
2. 领导选举：leader下线时，会选举一个新的leader，确保分区可用

#### 配置管理

可以动态修改配置，不需要进行重启整个集群

全局配置统一，确保broker和客户端都能访问到一致的配置

#### 消费者组管理

消费者组注册：向zookeeper注册信息，zookeeper负责管理这些用户组成员关系（建个目录，目录下的都是自己组的）

消费者组协调：确保一个分区制备一个消费者使用，避免重复消费

#### 偏移量管理

偏移量存储：新版本中zookeeper仍然可以存储偏移量

#### 元数据管理

kafka的主题元数据信息存储到zookeeper中确保所有broker和客户端都能访问到一致的元数据

集群中的元数据信息存储到zookeeper中确保集群的一致性和高可用性

#### 故障检测和恢复

通过监控各个组件zookeeper会及时检测并通知其他组件进行处理

协助重新分区和选举leader

## kafka集群部署

问题：

hive依赖hadoop吗（对）

zk必须部署到hadoop的机器上吗？（错）（Zookeeper 和 Hadoop 没有强制的绑定关系，可以独立部署在不同的服务器上。）

kafka必须和zk在一个集群上吗，kafka的部署机器必须也是zk的部署机器吗（错）

### kafka流程API(背诵)

#### kafka写入消息

写入方式：生产者采用**推**的模式将消息发布到broker，每条消息都追加到分区（patition）中

#### 分区

![image-20250310160501413](C:\Users\陈宣玮\AppData\Roaming\Typora\typora-user-images\image-20250310160501413.png)

- 生产者随机链接一个broker，根据topic发送meta协议请求;
- broker返回topic对应的所有分区数据，包括leader和leader所在的broker；
- 生产者会跟所有broker都建立好TCP长链接；
- 生产者在客户端根据消息发送策略计算要发送到的分区的broker；
- 生产者发送消息，客户端在消息积赞到发送条件时触发批量发送；

#### 分区策略

轮询策略，随机策略，按消息键保序策略

- 轮询策略：默认的分区策略，能够保证消息最大限度的被平均分配到所有分区
- 随机策略（已过时）：生产的消息被随机分配到不同的分区
- 按消息键保序策略：每条消息定义消息键key，相同的消息键确保能被被写入同一个分区

#### 写入ACK

ACK是生产者发送数据到指定的topic之后topic返回一个ACK表示topic接收成功可以进行下一次发送

通过配置acks参数0，1，-1（all） 三种级别

0：最快（最低延迟），一接收还没落盘就返回ack，当broker故障时可能丢失数据

1：成功落盘之后返回ack，在follower同步成功之前leader故障，将丢失数据

-1（all）：成功罗盘并且leader和follower全部罗盘成功返回ack，（follower并不是全部写入才返回ack，是落盘一部分就返回）

#### 生产者重试机制

创建生产者时，指定retries参数，向broker发送消息抛出异常时，并且异常为RetriableException可以按照指定次数进行重试，

可以重试情况：未达到delivery时间；重试剩余次数大于0；异常为RetriableException或者使用事务管理器时允许重试

#### 副本

同一个partition可能会有多个replication（对应server.properties 配置中default.replication.factor=N）。没有replication的情况下，一旦broker岩机，其上所有patition的数据都不可被消费，同时producer也不能再将
数据存于其上的patition。引入replication之后，同一个partition可能会有多个replication，而这时需要在这些
replication之间选出一个leader，producer和consumer只与这个leader交互，其它replication作为follower从
leader 中复制数据。

#### 写入流程

- poducer先从zookeeper的"/brokers/./state"节点找到该partition的leader
- producer将消息发送给该leader
- leader将消息写入本地log
- followers从leaderpull消息，写入本地log后向leader发送ACK
- leader收到所有ISR中的replication的ACK后，增加HW（high watermark，最后commit 的offset）并向producer发送ACK（在 Kafka 中表示 **ISR（同步副本集合）中所有副本都成功复制的最高偏移量**。它的作用是 **确定消费者可以读取的最大消息偏移量**，确保数据一致性和可靠性。）

#### 架构优化

常规的架构正常使用是没有问题的，主要的问题在于当生产者暴增时，会造成客户端与broker有大量TCP链接，而且是长链接。曾经一度我们生产者发展到20w的时候，broker大概有10个物理机，造成每个物理机上长期有20-30w的TCP链接。导致有些生产者容易被挤掉线，并且掉线后很难再找到链接上来的机会；改到在broker前面增加了四层负载的方式，保障所有生产者和消费者都能顺畅链接；

![image-20250310193723052](C:\Users\陈宣玮\AppData\Roaming\Typora\typora-user-images\image-20250310193723052.png)





### broker保存消息

存储方式

顺序写磁盘（顺序写磁盘效率高于随机写内存，没有内存容量限制，保障kafka吞吐率）

物理上把topic分成多个或者一个patition（对应配置信息中的num.partition=3），每个patition对应一个文件夹，对应机器上都会有一份topic的文件

存储策略

无论是否被消费都会保存所有消息

删除旧数据的的策略：基于时间，基于大小

kafka读取特定消息的时间复杂度为o(1)与文件大小无关，删除过期文件与提高性能无关

zookeeper管理kafka存储的数据

- 生产者无需注册，直接上传
- kafka集群需要注册，保存集群相关配置信息
- 消费着需要注册，因为有消费者组概念（防止消费者的重复读取）

### 消费信息

消费者组

消费者是以消费者组（一个以上消费者组成）形式工作，共同消费一个topic。在一个消费者组中，消费者可以消费多个partition但是不能重复消费，不同组中同一个partition可以被多个消费者组消费。如果group成员有一个失败了，那么其他消费者会自动负载均衡读取之前失败的消费者读取的分区

消费方式

拉的形式从broker中来读取数据，可以控制消费速度，kafka中缓存大量数据，不担心数据丢失

### Controller

是kafka集群的老大，选举产生的

负责管理和协调kafka集群的，管理集群中所有分区的状态并执行相应的管理操作，kafka集群只能有一个controller，每个broker都会参与竞选，controller崩溃时，会选出下一个承担上一任controller的所有工作

#### controller选举过程

一个节点启动时会尝试在zookeeper上创建controller这个临时节点，第一个创建成功的即为controller，成为contraller之后会增加版本号，更新epoch的节点值，其他未成为controller转为监听这个controller节点，当controller崩溃时，其他未成为controller进行又一次选举，

#### 管理状态

维护的状态分为两类：每台broker上的分区副本（副本状态）和每个分区的leader副本信息（分区状态）

#### Controller职责

- 更新集群元数据信息
  当分区信息发生变更，controller将变更信息发送包装为一种请求，发送给集群中的每一个broker，在客户端请求数据时总是能获得到最新最及时的数据

- 创建topic
  启动时创建一个zookeeper监听器，在zookeeper的broker/topic节点下创建对应的znode，把分区信息以及对应的副本列表写入这个znode，监听器一旦监控到该目录下有新增znode就立即出发topic创建逻辑（新建topic分区和确定leader和ISR），更新集群元数据信息，并且再次创建一个新的监听器用于监听该节点的变更，这样发生变化也可以通知到

- 删除topic
  向zookeeper下的admin/delete topic新增节点znode，启动时创建一个监听器监听该节点下的变化，有新增节点就开启删除topic的逻辑（停止副本运行，删除副本日志数据）
  直接删除topic则：                                        
  **部分 Broker 崩溃后重启**，如果还保存着旧的 Topic 数据，可能会导致不一致。
  **Controller 发生故障或重启**，如果它在删除过程中崩溃，可能会导致部分数据残留，影响整个集群的稳定性。

- 分区重分配
  重新分配副本所在的broker位置，达到更加均衡的效果,(分区副本重分配的过程实际上是先扩展再收缩的过程。controller首先将分区副本集合进行扩展（旧副本集合与新副本集合的合集），等待它们全部与leader保持同步之后将leader设置为新分配方案中的副本，最后执行收缩阶段，将分区副本集合缩减成分配方案中的副本集合)

- preferred leader副本选举
  在集群运行过程中，分区中的leader会因为种种原因，不再是preferred leader，用户可以通过命令来使得leader重新成为preferred leader（减少因 Leader 频繁变更带来的影响。）手动调节和自动调节

- topic分区扩展
  topic现有分区可能不足以支撑客户端的业务量，就增加分区(创建topic之后注册一个新的监听器监听分目录数据的变化，一旦增加topic分区就会触发执行创建任务)

- broker加入集群
  每个broker成功启动之后都会在zookeeper上创建一个znode，如果要动态维护就需要注册一个zookeeper监听器监控该目录数据变化，当有新broker加入时会执行broker启动任务，更新集群元数据信息

- broker崩溃
  当前broker在zookeeper上注册的是znode临时节点，因此一旦崩溃，broker与zookeeper的会话会失效并且导致临时节点删除，在上面监控的被用来监听那些一位内崩溃而退出集群的broker列表，若发现有broker子目录"消失”，controller便立即可知该broker退出集群，从而开启broker退出逻辑，最后更新集群元数据并同步到其他broker上。
  
- broker受控关闭
  自然关闭，由即将关闭的broker发向controller，等待关闭的broker处于阻塞状态，直到接收到broker端发出的收到成功指令，成功关闭

- controller leader选举

- 避免脑裂

  由于出现问题，contraller与zookeeper会话超时，kafka就会选出一个新老大叫contraller2然后老controller发现有新老大就会完成退出操作，使自己变成一个普通节点

### producer拦截器

#### 什么是拦截器

interceptor使得用户在消息发送前以及producer回调逻辑前有机会对消息做一些定制化需求，同时，producer允许用户指定多个interceptor按序作用于同一条消息从而形成一个拦截链

- configure（configs）
  获取配置信息和初始化数据时调用。
- onSend（ProducerRecord）
  用户可以在该方法中对消息做任何操作，但最好保证不要修改消息所属的topic和分区，否则会影响目标分区的计算
- onAcknowledgement（RecordMetadata，Exception）
  该方法会在消息被应答之前或消息发送失败时调用，并且通常都是在producer回调逻辑触发之前
- close
  关闭interceptor，主要用于执行一些资源清理工作

### kafka streaming

#### 简介

客户端库，来构建高伸缩性、高弹性、高容错性的分布式应用或微服务，不具备完整功能，比如调度和资源管理器。

差异分析

- 应用部署
  kafkaStreams应用由开发者自行管理生命周期，在Flink中，流处理应用建模为单个流处理计算逻辑，封装成Flink作业.Spark类似，Strom中称为拓扑，作业的生命周期由框架管理。Fink这类框架同时存在资源管理器，作业所需资源由资源管理器支持，可以借助Yarn、Kubernetes这类外部资源管理器实现，也可以通过standalone集群方式使用内置资源管理器。（更轻量级，写入写出都在kafka）
  总结：kafkaStreams应用完全由开发者控制管理

- 上下游数据源
  不借助KafkaConnector组件只能从kafka读数据以及写数据，即使使用组件，组件本身缺陷也会影响到应用。总结：只支持与Kafka集群交互，没有提供开箱即用的外部数据连接器。
- 协调方式
  分布式协调方面．KafkaStreams依赖集群提供的协调功能来提供高容错性和高伸缩性（通过消费者组机制实现），更轻量级，可随意增加流处理应用节点，即使应用节点异常也能重新分配给其他节点，而其他框架通过专属节点实现（比如zookeeper节点），需要使用特定API开启检查点机制，介入到错误恢复的处理中。

#### 特点

- 功能强大：高扩展性，弹性，容错
- 轻量级：无需专门的集群，一个库而不是框架
- 完全集成：kafka版本完全兼容，易于集成到溴铵有的应用程序
- 实时性：毫秒级延迟，并非微批处理，允许迟到数据

#### 为什么需要kafka streaming

1. 简单：直接在kafka基础上使用
2. 减少学习成本：不用再学spark  
3. 在线调整并行度

### 状态机

#### 副本状态机

1. NewReplica：
   controller可以在在partition重新分配期间创建新的replicas。在这种状态下，副本只能成为follower状态更改请求。有效的前置状态为NonExistentReplica
2. OnlineReplica
   一旦启动了replica并为其partition分配了部分replica，它就处于这种状态。在这种状态下，它可以成为leader或follower状态更改请求。有效的前置状态为NewReplica、OnlineReplica、OfflineReplica和ReplicaDeletionlneligiblet
3. OfflineReplica
   如果replica死亡，它将移动到此状态。当承载replica的broker关闭时，就会发生这种情况。有效的前置状态为NewReplica、OnlineReplica、OfflineReplica和ReplicaDeletionlneligible
4. ReplicaDeletionStarted
   如果开始删除replica，它将移动到此状态。有效的前置状态为OfflineReplica
5. ReplicaDeletionSuccessful
   如果replica在响应删除replica请求时没有错误代码，则将其移动到此状态。有效的前置状态为ReplicaDelegationStarteded
6. ReplicaDeletionlneligible
   如果replica删除失败，则将其移动到此状态。有效的前置状态为ReplicaDelegationStarted和OfflineReplica
7. OnExistentReplica
   如果replica被成功删除，它将移动到此状态。有效的前置状态为ReplicaDelegationSuccessful

#### 分区状态机

定义了分区可以处于的状态和转换状态的前置状态

1. NonExistentPartition（不存在的分区。）
   此状态表示分区从未创建或创建后删除。有效的前置状态（如果存在）是OfflinePartition
2. NewPartition
   创建后，分区处于NewPartition状态。在这种状态下，分区应该分配了副本，但还没有leader/isr。有效的前置状态为NonExistentPartition
3. OnlinePartition？
   一旦为分区选出了领导者，它就处于OnlinePartition状态。有效的前置状态为NewPartition/OfflinePartition
4. OfflinePartition
   如果在成功选举领导者后，分区的领导者死亡，则分区将移动到OfflinePartition状态。有效的先前状态为NewPartition/OnimePartition





