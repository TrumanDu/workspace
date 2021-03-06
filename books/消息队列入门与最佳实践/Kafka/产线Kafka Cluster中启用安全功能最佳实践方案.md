# 产线Kafka Cluster中启用安全功能最佳实践方案

为了保证正常业务不受影响，集群在启用安全功能的过程中还可正常提供服务，可以使用多阶段，多端口，多协议的方案。

- 以增量替换的方式添加额外的安全端口(s)。
- 客户端使用安全的端口来连接，而不是PLAINTEXT端口的（假设你是客户端需要安全连接broker）。
- 再次增量的方式依次启用broker与broker之间的安全端口（如果需要）
- 最后依次关闭PLAINTEXT端口。

本方案以`confluentinc/cp-kafka:5.2.1` 版本进行试验，SASL机制选择：`SASL/SCRAM`  特此说明。SCRAM全称为Salted Challenge Response Authentication Mechanism。

Kafka 支持如下SASL机制：

| SASL机制         | Kafka版本 | 特点                                                   |
| :--------------- | :-------- | :----------------------------------------------------- |
| SASL/OAUTHBEARER | 2.0.0     | 需自己实现接口实现token的创建和验证，需要额外Oauth服务 |
| SASL/Kerberos    | 0.9.0.0   | 需要独立部署验证服务                                   |
| SASL/PLAIN       | 0.10.0.0  | 不能动态增加用户                                       |
| SASL/SCRAM       | 0.10.2.0  | 可以动态增加用户                                       |

## 方案 

### 增量方式添加额外安全端口

#### 1.创建admin SCRAM证书

```
kafka-configs --zookeeper xx.xx.xx.xx:2181 --alter --add-config 'SCRAM-SHA-256=[password=admin]' --entity-type users --entity-name admin
# 查看admin的SCRAM证书
kafka-configs --zookeeper xx.xx.xx.xx:2181 --describe --entity-type users --entity-name admin
```

当然也可以只用`--bootstrap-server`来创建证书。

#### 2.新建`kafka_server_jaas.conf`

```
KafkaServer { org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin";
};
```

在docker volumes中添加该文件

```
/opt/app/cp-kafka-5.2.1/secrets:/etc/kafka/secrets
```

#### 3.在Env修改 Kafka 配置参数

```
KAFKA_OPTS=-Djava.security.auth.login.config=/etc/kafka/secrets/kafka_server_jaas.conf
KAFKA_LISTENERS=PLAINTEXT://{ip}:9092,SASL_PLAINTEXT://{ip}:9093
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://{ip}:9092,SASL_PLAINTEXT://{ip}:9093
KAFKA_SASL_ENABLED_MECHANISMS=SCRAM-SHA-256
KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL=SCRAM-SHA-256
KAFKA_AUTHORIZER_CLASS_NAME=kafka.security.auth.SimpleAclAuthorizer
KAFKA_SUPER_USERS=User:admin
```

在`LISTENERS`和`ADVERTISED_LISTENERS` 我们保持PLAINTEXT和SASL_PLAINTEXT都存在。这样就可以保持在过度期间，不会影响client未启用安全的服务。但最后我们是需要删掉PLAINTEXT。

其他参数是因为要使用SASL/SCRAM和ACLs增加的配置。

除了以上参数，在Confluent Kafka版本还需要添加如下参数：

```
KAFKA_CONFLUENT_SUPPORT_METRICS_ENABLE=false
KAFKA_ALLOW.EVERYONE.IF.NO.ACL.FOUND=true
```

本次方案没有开启zookeeper安全验证，因此也需要增加禁用zookeeper SASL功能,官方中推荐使用网略隔离策略来保证zookeeper安全性，当然也可以采用`mutual TLS`，本次方案未考虑。

```
ZOOKEEPER_SASL_ENABLED=false
```

然后依次重启Kafka broker,直至集群整体都启用安全验证。

### 客户端使用安全的端口来连接

此步为客户端修改配置为安全验证，在实践过程中，也可以延期做。

一般是通过Properties增加来启用安全配置。

**Producer**

```
public void producer() throws ExecutionException, InterruptedException, IOException {
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "xxx.xxx.xxx.xxx:9093");
        props.setProperty("key.serializer", StringSerializer.class.getName());
        props.setProperty("value.serializer", StringSerializer.class.getName());
        props.setProperty("security.protocol", "SASL_PLAINTEXT");
        props.setProperty("sasl.mechanism", "SCRAM-SHA-256");
        props.setProperty("sasl.jaas.config", "org.apache.kafka.common.security.scram.ScramLoginModule required  username=\"user\" password=\"user\";");

        KafkaProducerTemplate<String, String> producer = new KafkaProducerTemplate<>(props);
        while (true){
            producer.send("truman_test",System.currentTimeMillis()+"",System.currentTimeMillis()+"");
            Thread.sleep(1000);
        }
    }
```

**Consumer**:

```
public void consumer() {
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "xxx.xxx.xxx.xxx:9093");
        props.setProperty("group.id", "test");
        props.setProperty("enable.auto.commit", "true");
        props.setProperty("auto.commit.interval.ms", "1000");
        props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        props.setProperty("security.protocol", "SASL_PLAINTEXT");
        props.setProperty("sasl.mechanism", "SCRAM-SHA-256");
        props.setProperty("sasl.jaas.config", "org.apache.kafka.common.security.scram.ScramLoginModule required  username=\"user\" password=\"user\";");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("truman_test"));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
            }
        }
    }
```

### 再次增量的方式依次启用broker与broker之间的安全端口

这步在实践中也是可以省略的，不过为了安全，推荐启用该配置

在Env中增加`KAFKA_SECURITY_INTER_BROKER_PROTOCOL=SASL_PLAINTEXT`，然后再次挨个重启broker.

### 最后依次关闭PLAINTEXT端口

修改Env中`LISTENERS`和`ADVERTISED_LISTENERS`，移除其中的PLAINTEXT，挨个重启Broker，即可完成在集群中启用安全配置。

```
KAFKA_LISTENERS=SASL_PLAINTEXT://{ip}:9093
KAFKA_ADVERTISED_LISTENERS=SASL_PLAINTEXT://{ip}:9093
```

## SCRAM证书管理

**创建**

```
kafka-configs --zookeeper x.x.x.x:2181 --alter --add-config 'SCRAM-SHA-256=[password=user]' --entity-type users --entity-name user
```



## Acls

Kafka授权原语主要有以下几种：

- Read
- Write
- Create
- Delete
- Alter
- Describe
- ClusterAction
- DescribeConfigs
- AlterConfigs
- IdempotentWrite
- All

限制资源：

- Topic
- Group
- Cluster
- TransactionalId
- DelegationToken

概括来说，通过Acls可以给指定用户，指定机器host赋予限制资源的操作（授权原语）。资源的匹配模式支持三种：1.literal(默认)2.match 3.prefixed 

host可以是白名单，也可以是黑名单。

### 查看Acls信息

```
#查询所有的topic
kafka-acls --authorizer-properties zookeeper.connect=x.x.x.x:2181 --list --topic *
# 查询指定topic
kafka-acls --authorizer-properties zookeeper.connect=x.x.x.x:2181 --list --topic truman_test
```

### 增加Producer和Consumer权限

```
kafka-acls --authorizer-properties zookeeper.connect=x.x.x.x:2181 --add --allow-principal User:user --producer --topic truman_test
kafka-acls --authorizer-properties zookeeper.connect=x.x.x.x:2181 --add --allow-principal User:user --consumer --topic truman_test --group test 
```

如果需要删除的话，只用将以上命令的add修改为remove。

### 按指定前缀分配资源

```
kafka-acls --authorizer-properties zookeeper.connect=x.x.x.x:2181 --add --allow-principal User:xaecbd --producer --resource-pattern-type prefixed --topic truman 
kafka-acls --authorizer-properties zookeeper.connect=x.x.x.x:2181 --add --allow-principal User:xaecbd --consumer --resource-pattern-type prefixed --topic truman --group test 
```

这里要主要的是按前缀方式设置的话，会导致该前缀生效配置Acls,其他未安全验证的client都会受此影响，因此在迁移的过程中注意。资源的匹配模式支持三种：1.literal(默认，这种模式支持*，指匹配所有)2.match 3.prefixed 

### 限制Host

```
kafka-acls --authorizer-properties zookeeper.connect=x.x.x.x:2181 --add --allow-principal User:xaecbd --allow-host x.x.x.x --producer --topic truman_test
kafka-acls --authorizer-properties zookeeper.connect=x.x.x.x:2181 --add --allow-principal User:xaecbd --allow-host x.x.x.x --consumer --topic truman_test --group test 
```

除了白名单，还可以设置黑名单(`deny-host`)，该参数支持设置`*`，不填写该参数的话，默认为`*`代表所有，不能设置为`x.x.x.*`

除此以外还支持通过`--operation` 设置原语级。例如：`--operation Read --operation Write`

## 配额

配额不属于安全范畴，启用安全策略以后，就可以通过设置user维度配额（不开启身份认证，只能通过clientid来进行限流），配额能限制生产者和消费者的流量。

- producer_byte_rate。发布者单位时间（每秒）内可以发布到单台broker的字节数。
- consumer_byte_rate。消费者单位时间（每秒）内可以从单台broker拉取的字节数。

```
#1. 配置user+clientid。例如，user为”user1”，clientid为”clientA”。
kafka-configs  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048'  \
                      --entity-type users --entity-name user1 --entity-type clients --entity-name clientA

#2. 配置user。例如，user为”user1”
kafka-configs  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' \
                      --entity-type users --entity-name user1

#3. 配置client-id。例如，client-id为”clientA”
kafka-configs  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' \
                      --entity-type clients --entity-name clientA
```



## 参考

1. [7.5 Incorporating Security Features in a Running Cluster](https://kafka.apache.org/documentation/#security_rolling_upgrade)
2. [Configuring SCRAM](https://docs.confluent.io/platform/current/kafka/authentication_sasl/authentication_sasl_scram.html#security-considerations-for-sasl-scram)
3. [Docker Configuration Parameters](https://docs.confluent.io/platform/current/installation/docker/config-reference.html#)
4. [Configuring SCRAM](https://docs.confluent.io/platform/current/kafka/authentication_sasl/authentication_sasl_scram.html#)