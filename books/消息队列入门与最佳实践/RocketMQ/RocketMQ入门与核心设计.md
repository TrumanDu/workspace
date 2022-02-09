# RocketMQ入门与核心设计

## 环境搭建与入门

### 环境搭建

RocketMQ由两个组件构成：**Name Server** ，**Broker**

下面我们分别部署该服务，为了避免调试环境等问题，推荐使用docker搭建，本文也是采用该方式。

首先部署**Name Server** 

```
docker run -d -v /opt/app/rocketmq/mqnamesrv/logs:/home/rocketmq/logs \
      --name rmqnamesrv \
      -e "JAVA_OPT_EXT=-Xms512M -Xmx512M -Xmn128m" \
      -p 9876:9876 \
      foxiswho/rocketmq:4.8.0 \
      sh mqnamesrv
```

然后再部署**Broker**

```
docker run -d  -v /opt/app/rocketmq/mqbroker/logs:/home/rocketmq/logs -v /opt/app/rocketmq/mqbroker/store:/home/rocketmq/store \
      -v /opt/app/rocketmq/mqbroker/conf:/home/rocketmq/conf \
      --name rmqbroker \
      -e "NAMESRV_ADDR=127.0.0.1:9876" \
      -e "JAVA_OPT_EXT=-Xms512M -Xmx512M -Xmn128m" \
      -p 10911:10911 -p 10912:10912 -p 10909:10909 \
      foxiswho/rocketmq:4.8.0 \
      sh mqbroker -c /home/rocketmq/conf/broker.conf
```

内容参考：[foxiswho/rocketmq](https://hub.docker.com/r/foxiswho/rocketmq)

### 入门

## 核心设计

## 高阶知识

## 参考

1.[]()