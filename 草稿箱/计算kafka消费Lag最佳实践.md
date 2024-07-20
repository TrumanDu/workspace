# 计算kafka消费组Lag最佳实践最新版

之前写过一篇[如何监控kafka消费Lag情况](https://blog.trumandu.top/2019/04/13/%E5%A6%82%E4%BD%95%E7%9B%91%E6%8E%A7kafka%E6%B6%88%E8%B4%B9Lag%E6%83%85%E5%86%B5/)，五年前写的，在google上访问量很大，最近正好需要再写这个功能，就查看了最新API，发现从`2.5.0`版本后新增了`listOffsets`方法，让计算Lag更简单方便和安全,代码量有质的下降，因为舍弃一些功能，代码精简的了很多。

## 实践

这里我用最新版做演示，在pom文件中增加依赖
```
<dependency>
	<groupId>org.apache.kafka</groupId>
	<artifactId>kafka-clients</artifactId>
	<version>3.7.1</version>
</dependency>
```

首先初始化AdminClient
```
Properties config = new Properties();
config.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, hosts);
AdminClient adminClient = AdminClient.create(config);
```
然后根据topic和groupId计算Lag,这种方案要比之前方式优雅了很多。

```
    public long getConsumerLag(String topicName,String consumerGroupId) {
        try {
            // 获取主题描述
            TopicDescription topicDescription = adminClient.describeTopics(Collections.singletonList(topicName))
                    .topicNameValues()
                    .get(topicName).get();
            List<TopicPartition> partitions = topicDescription.partitions()
                    .stream()
                    .map(partitionInfo -> new TopicPartition(topicName, partitionInfo.partition()))
                    .collect(Collectors.toList());

            // 获取每个分区的最新偏移量
            Map<TopicPartition, OffsetSpec> requestLatestOffsets = new HashMap<>(partitions.size());
            for (TopicPartition partition : partitions) {
                requestLatestOffsets.put(partition, OffsetSpec.latest());
            }

            ListOffsetsResult listOffsetsResult = adminClient.listOffsets(requestLatestOffsets);
            Map<TopicPartition, Long> latestOffsets = new HashMap<>(partitions.size());
            for (TopicPartition partition : partitions) {
                latestOffsets.put(partition, listOffsetsResult.partitionResult(partition).get().offset());
            }

            // 获取消费者组的当前偏移量
            ListConsumerGroupOffsetsSpec listConsumerGroupOffsetsSpe = new ListConsumerGroupOffsetsSpec();
            listConsumerGroupOffsetsSpe.topicPartitions(partitions);
            Map<String, ListConsumerGroupOffsetsSpec> groupSpecs = new HashMap<String, ListConsumerGroupOffsetsSpec>(){
                {
                    put(consumerGroupId, listConsumerGroupOffsetsSpe);
                }
            };
            Map<TopicPartition, OffsetAndMetadata> currentOffsets = adminClient.listConsumerGroupOffsets(groupSpecs)
                    .partitionsToOffsetAndMetadata().get();

            long lag = 0;
            for (TopicPartition partition : partitions) {
                OffsetAndMetadata offsetMetadata = currentOffsets.get(partition);
                long currentOffset = offsetMetadata != null ? offsetMetadata.offset() : 0;
                lag += latestOffsets.get(partition) - currentOffset;
            }
            return lag;
        } catch (InterruptedException | ExecutionException e) {
            return -1;
        }
    }
```
**注意**

API 对kafka的兼容性，我在kakfa 服务器版本`2.6.0`测试通过，更低版本，建议自测！

**该方法对消息过期，计算Lag存在一定错误，请注意！！！**

对于[如何监控kafka消费Lag情况](https://blog.trumandu.top/2019/04/13/%E5%A6%82%E4%BD%95%E7%9B%91%E6%8E%A7kafka%E6%B6%88%E8%B4%B9Lag%E6%83%85%E5%86%B5/)原文中consumerOffset存在一处错误，为避免reblance,不要`subscribe`,建议更改为以下
```
KafkaConsumers<String, String>  kafkaConsumer  = new KafkaConsumers<>(consumerProps);
	List<PartitionInfo> patitions = kafkaConsumer.partitionsFor(topicName);
	List<TopicPartition>topicPatitions = new ArrayList<>();
	patitions.forEach(patition->{
		TopicPartition topicPartition = new TopicPartition(topicName,patition.partition());
		topicPatitions.add(topicPartition);
	});
	Map<TopicPartition, Long> result = kafkaConsumer.endOffsets(topicPatitions);
```


