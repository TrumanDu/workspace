# Kafka

## Producer 有哪些常用的参数?

```
bootstrap.servers
acks
retries
batch.size
linger.ms
```

```
delivery.timeout.ms
max.block.ms
enable.idempotence
max.in.flight.requests.per.connection
```



## Consumer 有哪些常用的参数?

```
bootstrap.servers
group.id
enable.auto.commit
auto.offset.reset
max.poll.records
session.timeout.ms
heartbeat.interval.ms
```

```
isolation.level
allow.auto.create.topics
```

### Consumer端分区分配策略

`RangeAssignor`（默认值）,`RoundRobinAssignor`,`StickyAssignor`

通常是通过`partition.assignment.strategy`配置生效。

更详细查看[kafka consumer 源码分析（二）分区分配策略](http://blog.trumandu.top/2019/06/27/kafka-consumer-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%BA%8C%EF%BC%89%E5%88%86%E5%8C%BA%E5%88%86%E9%85%8D%E7%AD%96%E7%95%A5/)

## 更多

[Kafka面试题与答案全套整理](http://blog.trumandu.top/2019/04/13/Kafka%E9%9D%A2%E8%AF%95%E9%A2%98%E4%B8%8E%E7%AD%94%E6%A1%88%E5%85%A8%E5%A5%97%E6%95%B4%E7%90%86/)