《Kafka核心技术与实战》学习笔记

只是记录自己不知道的知识点，或者经常容易忘的内容,想知道更多的信息建议看课程或者找我聊天交换技能。在课程的内容基础之上，补充了一些自己知道的知识点。

![](G:\写作\workspace\文章仓库\images\《Kafka核心技术与实战》学习笔记\kafka-core_tech.png)

## Kafka的认知

- 分布式消息引擎平台
- 分布式实时流式处理平台

早期Kafka社区对Kafka的定位为⼀个分布式、分区化且带备份功能的提交⽇志（Commit Log）服务,近期在官网彻底更改为分布式实时流式处理平台。

## Kafka流式处理框架的优势

- 更容易实现端到端的正确性（Correctness）
- 轻量型，嵌入式流式计算的定位

## 避免不必要的Rebalance

- session.timeout.ms
-  heartbeat.interval.ms 
- max.poll.interval.ms 
- GC参数

session.timout.ms决定了Consumer存活性的时间间隔

heartbeat.interval.ms决定存活心跳发送间隔。

max.poll.interval.ms 它限定了Consumer端应⽤程序两次调⽤poll⽅法的最⼤时间间隔。

## 消费者TCP管理

消费者实例在**KafkaConsumer.poll**建立TCP连接，主要分为3类连接：

1. 确定协调者和获取集群元数据。
2.  连接协调者，令其执⾏组成员管理操作。 
3. 执⾏实际的消息获取。

第一类连接仅在开始前建立，稍后（第三类创建成功）就会销毁，consumer实例会长期保留2,3类连接。

Consumer实例会长期建立broker数量（分区所在broker数量）+1个连接。

**TCP连接的三个时机：**

1. 发起FindCoordinator请求时
2. 连接协调者时
3. 消费数据时

**何时关闭TCP连接：**

1. ⼿动调⽤KafkaConsumer.close()⽅法
2. 执⾏Kill命令
3. Kafka⾃动关闭（是由消费者端参数connection.max.idle.ms控制的，该参数现在的默认值是9分钟）

## 跨集群备份解决⽅案

1. MirrorMaker
2. Uber的uReplicator⼯具
3. LinkedIn开发的Brooklin Mirror Maker⼯具
4. Confluent公司研发的Replicator⼯具

[MirrorMaker2.0](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0)已经发布（2.4.0+）基于connector重写了MirrorMaker。目前我们公司使用自研的Kafka connector实现跨集群数据镜像。

## Kafka副本机制

### Kafka判断Follower 是否与Leader同步的标准

Broker端参数**replica.lag.time.max.ms**参数值，这个参数的含义是Follower副本能够落后Leader副本的最⻓时 间间隔，当前默认值是10秒。

### Unclean领导者选举（Unclean Leader Election）

Broker端参数unclean.leader.election.enable控制是否允许Unclean领导者选举，true提高了高可用，降低了数据一致性。推荐设置为false

[0.11.0](http://kafka.apache.org/documentation/#upgrade_1100_notable)以后的版本默认都是false,从[2.1.0版本](http://kafka.apache.org/documentation/#upgrade_210_notable)开始复写动态配置的话，默认为启用



## Kafka请求是怎么被处理

![](http://static.trumandu.top/kafka_request.png)

num.network.threads 决定网络线程池的数量，默认值为3

num.io.threads 控制IO线程池的数量，默认值为8

## Kafka JMX指标监控

- BytesIn/BytesOut：即Broker端每秒⼊站和出站字节数。你要确保这组值不要接近你的⽹络带宽，否则这通常都表示⽹卡已 被“打满”，很容易出现⽹络丢包的情形。 
- NetworkProcessorAvgIdlePercent：即⽹络线程池线程平均的空闲⽐例。通常来说，你应该确保这个JMX值⻓期⼤于 30%。如果⼩于这个值，就表明你的⽹络线程池⾮常繁忙，你需要通过增加⽹络线程数或将负载转移给其他服务器的⽅ 式，来给该Broker减负。 
- RequestHandlerAvgIdlePercent：即I/O线程池线程平均的空闲⽐例。同样地，如果该值⻓期⼩于30%，你需要调整I/O线程 池的数量，或者减少Broker端的负载。
- UnderReplicatedPartitions：即未充分备份的分区数。所谓未充分备份，是指并⾮所有的Follower副本都和Leader副本保持 同步。⼀旦出现了这种情况，通常都表明该分区有可能会出现数据丢失。因此，这是⼀个⾮常重要的JMX指标。 
- ISRShrink/ISRExpand：即ISR收缩和扩容的频次指标。如果你的环境中出现ISR中副本频繁进出的情形，那么这组值⼀定 是很⾼的。这时，你要诊断下副本频繁进出ISR的原因，并采取适当的措施。 
- ActiveControllerCount：即当前处于激活状态的控制器的数量。正常情况下，Controller所在Broker上的这个JMX指标值应 该是1，其他Broker上的这个值是0。如果你发现存在多台Broker上该值都是1的情况，⼀定要赶快处理，处理⽅式主要是查 看⽹络连通性。这种情况通常表明集群出现了脑裂。脑裂问题是⾮常严重的分布式故障，Kafka⽬前依托ZooKeeper来防⽌ 脑裂。但⼀旦出现脑裂，Kafka是⽆法保证正常⼯作的。

## Kafka常用Broker动态配置

以下为常用Broker动态配置，不用重启Broker即可生效

1. log.retention.ms。
2. num.io.threads和num.network.threads
3. 与SSL相关的参数（ssl.keystore.type、ssl.keystore.location、ssl.keystore.password和ssl.key.password）。。
4. num.replica.fetchers。

## 调优Kafka

- 保持客户端版本与Kafka broker版本一致，这样能享受Zero Copy
- JVM堆⼤⼩设置成6～8GB(自己推荐6g,具体详见linked数据)

### 增大吞吐量

当需要调整吞吐量的情况时可以考虑如下调整参数：

| 作用范围 | 参数                                                         |
| :------- | :----------------------------------------------------------- |
| Broker   | 1.适当增加数num.replica.fetchers数量，但不要超过CPU数量 2.调整GC参数以避免经常性GC |
| Producer | 1.增加消息批次的⼤⼩以及批次缓存时间，即batch.size和linger.ms 2.配置压缩算法，lz4/zstd 3.acks=0或1，4.retries=0 5.如果多个线程共享Producer，适当增大buffer.memory |
| Consumer | 增加fetch.min.bytes参数值。默认是1字节                       |

### 降低延时

当需要调整延时的情况时可以考虑如下调整参数：

| 作用范围 |                    参数                     |
| :------: | :-----------------------------------------: |
|  Broker  |        增加num.replica.fetchers数量         |
| Producer | 1.设置linger.ms=0 2.不要启⽤压缩 3.置acks=1 |
| Consumer |              fetch.min.bytes=1              |



## 消费者组状态及各个状态流转

![](G:\写作\workspace\文章仓库\images\《Kafka核心技术与实战》学习笔记\kafka_group_state.png)

![](G:\写作\workspace\文章仓库\images\《Kafka核心技术与实战》学习笔记\kafka_group_change_status.png)

## ⾼⽔位和LEO的更新机制

### 高水位作用

1. 消息可见性
2. 帮助kafka完成副本同步

### Leader副本 

处理⽣产者请求的逻辑如下：

1. 写⼊消息到本地磁盘。 

2. 更新分区⾼⽔位值。 
   i. 获取Leader副本所在Broker端保存的所有远程副本LEO值{LEO-1，LEO-2，……，LEO-n}。 
   ii. 获取Leader副本LEO：leader_leo。
   iii. 更新currentHW = min((leader_leo, LEO-1，LEO-2，……，LEO-n)。

处理Follower副本拉取消息的逻辑如下：

1. 读取磁盘（或⻚缓存）中的消息数据。
2. 使⽤Follower副本发送请求中的位移值更新远程副本LEO值。 
3. 更新分区⾼⽔位值（具体步骤与处理⽣产者请求的步骤相同）。 

### Follower副本 
从Leader拉取消息的处理逻辑如下： 
1. 写⼊消息到本地磁盘。 
2. 更新LEO值。 
3. 更新⾼⽔位值。
    i. 获取Leader发送的⾼⽔位值：currentHW（leader）。
    ii. 获取步骤2中更新过的LEO值：currentLEO。
    iii. 更新⾼⽔位为min(currentHW, currentLEO)。

文字比较费解，还是看图。原文和我写的有点出入，不过我感觉这样才是正确的！

![](G:\写作\workspace\文章仓库\images\《Kafka核心技术与实战》学习笔记\2020-10-14-kafka_core-hw.png)

## Leader Epoch

⾼⽔位更新错配导致的各种不⼀致问题，因此引入了Leader Epoch

所谓Leader Epoch，我们⼤致可以认为是Leader版本。它由两部分数据组成。 

1. Epoch。⼀个单调增加的版本号。每当副本领导权发⽣变更时，都会增加该版本号。⼩版本号的Leader被认为是过期 Leader，不能再⾏使Leader权⼒。
2.  起始位移（Start Offset）。Leader副本在该Epoch值上写⼊的⾸条消息的位移。

Kafka Broker会在内存中为每个分区都缓存Leader Epoch数据（会持久化,leader-epochcheckpoint⽂件），来避免重启截断日志的情况发生。

![](https://img-blog.csdnimg.cn/20190814184301298.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hZG9uZ3l1MTI1OTg5MjkzNg==,size_16,color_FFFFFF,t_70)