# Elasticsearch面试指南

## 1. Field data 与Doc values区别

|             | Doc Values                     | Field Data                                    |
| ----------- | ------------------------------ | --------------------------------------------- |
| 何时创建    | 索引时，和倒排索引一起创建     | 搜索时动态创建                                |
| 创建位置    | 磁盘文件                       | JVM Heap                                      |
| 优点        | 避免大量内存占用               | 索引速度快，不占用额外的磁盘空间              |
| 缺点        | 降低索引速度，占用额外磁盘空间 | 文档过多时，动态创建开销大，占用过多 JVM Heap |
| ES 默认采用 | ES 2.x 之后                    | ES 1.x 及以前                                 |
| text字段    | 不支持                         | 支持（默认禁用）                              |

如果确认不需要该字段排序和聚合，那么可以将它关闭`"doc_values": false`

## 2. 说你们公司ES的集群架构，索引数据大小，分片有多少，以及一些调优手段？

6个集群，60TB.

### 调优手段

**部署层面：**

1）最好是64GB内存的物理机器

3）尽量使用SSD

4）避免集群跨越大的地理距离，一般一个集群的所有节点位于一个数据中心中。

5）设置堆内存：节点内存/2，不要超过32GB。一般来说设置export ES_HEAP_SIZE=32g环境变量，比直接写-Xmx32g -Xms32g更好一点。

6）关闭缓存swap。

7）增加文件描述符，设置一个很大的值

8）不要随意修改垃圾回收器（CMS）和各个线程池的大小。

9）通过设置gateway.recover_after_nodes、gateway.expected_nodes、gateway.recover_after_time可以在集群重启的时候避免过多的分片交换，这可能会让数据恢复从数个小时缩短为几秒钟。

**索引层面：**

1）使用批量请求并调整其大小：每次批量数据 5–15 MB 大是个不错的起始点。

2）段合并：Elasticsearch默认值是20MB/s，对机械磁盘应该是个不错的设置。如果你用的是SSD，可以考虑提高到100-200MB/s。如果你在做批量导入，完全不在意搜索，你可以彻底关掉合并限流。另外还可以增加 index.translog.flush_threshold_size 设置，从默认的512MB到更大一些的值，比如1GB，这可以在一次清空触发的时候在事务日志里积累出更大的段。

3）如果你的搜索结果不需要近实时的准确度，考虑把每个索引的index.refresh_interval 改到30s。

4）如果你在做大批量导入，考虑通过设置index.number_of_replicas: 0 关闭副本。

5）需要大量拉取数据的场景，可以采用scan & scroll api来实现，而不是from/size一个大范围。

**存储层面：**

1）基于数据+时间滚动创建索引，每天递增数据。控制单个索引的量，一旦单个索引很大，存储等各种风险也随之而来，所以要提前考虑+及早避免。

2）冷热数据分离存储，热数据（比如最近3天或者一周的数据），其余为冷数据。对于冷数据不会再写入新数据，可以考虑定期force_merge加shrink压缩操作，节省存储空间和检索效率。

## 3. index生命周期

ILM 定义了四种解析阶段：

- **Hot**: 该索引正在积极地更新和查询。
- **Warm**: 索引不再更新，但是仍然被查询.
- **Cold**: 索引不再更新，很少被查询 ，索引仍然是可查询的，但是查询很慢。
- **Delete**: 该索引不再需要，可以被安全的删除

在不同的阶段可以做不行的动作：

- Hot
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-set-priority.html)
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-unfollow.html)
  - [Force Merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-forcemerge.html)
  - [Rollover](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-rollover.html)
- Warm
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-set-priority.html)
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-unfollow.html)
  - [Read-Only](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-readonly.html)
  - [Allocate](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-allocate.html)
  - [Shrink](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-shrink.html)
  - [Force Merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-forcemerge.html)
- Cold
  - [Set Priority](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-actions.html#ilm-set-priority-action)
  - [Unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-actions.html#ilm-unfollow-action)
  - [Allocate](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-allocate.html)
  - [Freeze](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-freeze.html)
- Delete
  - [Wait For Snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-actions.html#ilm-wait-for-snapshot-action)
  - [Delete](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-delete.html)

不同动作作用：

- **Rollover**: 滚动index生成
- **Shrink**: 减少索引的分片
- **Force merge**: Manually trigger a merge to reduce the number of segments in each shard of an index and free up the space used by deleted documents.
- **Freeze**: 标记索引为只读，最大减少内存的占用。
- **Force merges**: 标记索引为只读，合并索引的段，删除

## 4. 如何选择合适的分片和副本数？

分片数索引创建后**不可以修改**，副本数索引创建后可以通过API**随时修改**。

> 参考公式：所需的最大节点数 = 分片数 *（副本数+1） 
>
> Max number of nodes = Number of shards * (number of replicas +1)

举例：你计划5个分片和1个副本，那么所需要的最大的节点数为：5*（1+1）=10个节点。

## 5.ES是如何实现master选举的？

第一步：确认候选主节点数达标，elasticsearch.yml 设置的值 discovery.zen.minimum_master_nodes;

第二步：对所有候选主节点根据nodeId字典排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第0位）节点，暂且认为它是master节点。

第三步：如果对某个节点的投票数达到一定的值（候选主节点数n/2+1）并且该节点自己也选举自己，那这个节点就是master。否则重新选举一直到满足上述条件。

## 6. 如何解决ES集群的脑裂问题

> discovery.zen.minimum_master_nodes

## 7.关于 Lucene 的 Segement

- Lucene 索引是由多个段组成，段本身是一个功能齐全的倒排索引。
- 段是不可变的，允许 Lucene 将新的文档增量地添加到索引中，而不用从头重建索引。
- 对于每一个搜索请求而言，索引中的所有段都会被搜索，并且每个段会消耗 CPU 的时钟周、文件句柄和内存。这意味着段的数量越多，搜索性能会越低。
- 为了解决这个问题，Elasticsearch 会合并小段到一个较大的段，提交新的合并段到磁盘，并删除那些旧的小段。（段合并）

## 8.详细描述一下ES索引文档的过程？

搜索被执行成一个两阶段过程，即 **Query Then Fetch**；

**Query阶段：**
查询会广播到索引中每一个分片拷贝（主分片或者副本分片）。每个分片在本地执行搜索并构建一个匹配文档的大小为 from + size 的优先队列。PS：在搜索的时候是会查询Filesystem Cache的，但是有部分数据还在Memory Buffer，所以搜索是近实时的。
每个分片返回各自优先队列中 **所有文档的 ID 和排序值** 给协调节点，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。

**Fetch阶段：**
协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。每个分片加载并 丰富 文档，如果有需要的话，接着返回文档给协调节点。一旦所有的文档都被取回了，协调节点返回结果给客户端。

## 9.详细描述一下ES更新和删除文档的过程？

删除和更新也都是写操作，但是 Elasticsearch 中的文档是不可变的，因此不能被删除或者改动以展示其变更。

磁盘上的每个段都有一个相应的 .del 文件。当删除请求发送后，文档并没有真的被删除，而是在 .del 文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在 .del 文件中被标记为删除的文档将不会被写入新段。

在新的文档被创建时，Elasticsearch 会为该文档指定一个版本号，当执行更新时，旧版本的文档在 .del 文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉。

## 10.详细描述一下索引插入过程？

第一步：客户端向集群某节点写入数据，发送请求。（如果没有指定路由/协调节点，请求的节点扮演协调节点的角色。）

第二步：协调节点接受到请求后，默认使用文档 ID 参与计算（也支持通过 routing），得到该文档属于哪个分片。随后请求会被转到另外的节点。

```java
bash# 路由算法：根据文档id或路由计算目标的分片id
shard = hash(document_id) % (num_of_primary_shards)
```

第三步：当分片所在的节点接收到来自协调节点的请求后，会将请求写入到 Memory Buffer，然后定时（默认是每隔 1 秒）写入到F ilesystem Cache，这个从 Memory Buffer 到 Filesystem Cache 的过程就叫做 refresh；

第四步：当然在某些情况下，存在 Memory Buffer 和 Filesystem Cache 的数据可能会丢失，ES 是通过 translog 的机制来保证数据的可靠性的。其实现机制是接收到请求后，同时也会写入到 translog 中，当 Filesystem cache 中的数据写入到磁盘中时，才会清除掉，这个过程叫做 flush；

第五步：在 flush 过程中，内存中的缓冲将被清除，内容被写入一个新段，段的 fsync 将创建一个新的提交点，并将内容刷新到磁盘，旧的 translog 将被删除并开始一个新的 translog。

第六步：flush 触发的时机是定时触发（默认 30 分钟）或者 translog 变得太大（默认为 512 M）时。

## 11. 管理Elasticsearch

### - 有了副本机制为什么还需要集群备份？

副本是为了解决高可用，无法解决集群灾难

### - 集群备份

使用 snapshot API备份你的集群。 

### - 集群可以备份到哪里

> - 共享文件系统，比如 NAS
> - Amazon S3：亚马逊Web云服务
> - HDFS (Hadoop集群分布式文件系统)
> - Azure Cloud：微软云平台

## 12. 提高性能

### 1、后台什么在运行导致CPU飙升？如何排查？

热点线程APi能向你提供查找问题根源所必需的信息。 

> GET /_nodes/hot_threads?pretty

### 2、高负载场景Elasticsearch优化的常规建议？

这里是建议，不是准则。 
（1）选择正确的存储 
如：选择默认的default存储类型。

（2）按需设定刷新频率 
索引刷新频率定义：文档需要多长时间才能出现在搜索结果中。 
正确认知： 
1）刷新频率越短，查询越慢，且索引文档的吞吐率越低。 
2）默认刷新频率：1s刷新一次。 
3）无限的增加刷新时间是没有意义的，因为超过一定的值（取决于你的数据负载和数据量）之后，性能提升变得微乎其微。

（3）线程池优化 
线程池优化的必要场景：你看到节点正在填充队列并且仍然有计算能力剩余，且这些计算能力可以被指定用于处理待处理的任务。

（4）调整合并过程 
index.merge.policy.mery_factory低于默认值10，会导致更少的段，更低的RAM消耗，更快的查询速度和更慢的索引速度； 
若大于10，导致索引由更多的分段组成，更高的RAM消耗，更慢的查询速度和更快的索引速度。

（5）合理数据分布 
高索引量的使用场景：把索引分散到多个分片上来降低服务器CPU和I/O子系统的压力。

如果你的节点无法处理查询带来的负载，你可以增加更多的ES节点，并增加副本的数量，于是主分片的物理拷贝会部署到新增节点上。这样会使得文档索引慢一些，但是会给你同时处理更多查询的能力。

### 3.高负载、高查询频率场景的建议

（1）启动过滤器缓存和分片查询缓存 
过滤器缓存配置：indices.cache.filter.size 
分片查询缓存配置：indices.cache.query.enable

（2）优化查询语句结构 
书本中的过滤器已不再使用5.X以及更高版本。 
但，依然可以优化查询语句，返回核对查询同样语句返回时间，进行权衡优化。

（3）使用路由 
有着相同路由值的数据都会保存到相同的分片上。

（4）并行查询 
建议数据平均分配，多个节点可以有相同的负载。

（5）字段数据缓存和断路 
当使用聚合和排序等字段数据缓存相关操作时，遇到了内存相关的问题（内存泄漏或者GC停顿），可以考虑使用 doc value。

（6）控制size和shard_size 
主要针对聚合操作。 
增加size和shard_size能使得聚合结果更准确，代价是：更多的网络开销和内存使用； 
减少size和shard_size能使得聚合不那么精确，代价是：网络开销小和内存使用率低。

### 4、高负载、高索引吞吐量场景

（1）批量索引 
批量索引代替逐个索引文档。

（2）doc values和索引速度的取舍 
一方面：我们只关心更高的索引速度和更大的索引吞吐量，可以考虑不适用doc values。 
另一方面：如果有大量的数据，为了使用聚合和排序功能而不产生内存相关问题，唯一选择——使用 doc values。

（3）控制文档字段 
方式一：_source 控制字段输出； 
方式二：禁用_all。 
减少文档的大小和其内文本字段的数量会让索引稍快一些。

（4）索引的结构和副本 
1）主分片部署到所有的节点上，以便：并行的索引文档，加快索引的速度。 
2）过多的副本导致索引速度下降。

（5）调整预写日志。

（6）存储优化 
1）资金充足，建议购买SSD——原因：速度优势明显； 
2）资金不足，考虑ES使用多个数据路径。 
3）不要使用共享或者远程的文件系统保存索引——原因：速度慢，整体性能下降。

（7）索引期间的内存缓存 
建议给每个索引期间生效的分片分配512MB内存。 
indices.memory.index_buffer_size是设置节点的，而不是分片。



## 参考

1. [干货 |《深入理解Elasticsearch》读书笔记](https://mp.weixin.qq.com/s?__biz=MzI2NDY1MTA3OQ==&mid=2247483918&idx=1&sn=a9f2ad3c4549021e20fefafe6baa14ff&chksm=eaa82a26dddfa330eb53d6b997fe988f2c5e037caf73e2e9fab9fdbc353f6d6b362a71ae057c&scene=178&cur_album_id=1340073242396114944#rd)
2. [Elasticsearch面试题汇总与解析](https://www.wenyuanblog.com/blogs/elasticsearch-interview-questions.html)

