# Flink 入门笔记

Flink 是一个支持实时计算的引擎框架。

Flink 有三个角色：**Client,JobManager,TaskManager**。Client 负责向集群提交应用程序，JobManager 负责执行过程中必要的调度管理，Taskmanager 负责实际的计算。

## Flink 应用场景

-   **事件驱动型应用**,比如：信用卡交易、刷单、监控等；
-   **数据分析应用**，比如：库存分析、双 11 大屏等；
-   **数据管道应用**，也就是 ETL 场景，比如一些日志的解析等；

参考：[https://flink.apache.org/zh/what-is-flink/use-cases/](https://flink.apache.org/zh/what-is-flink/use-cases/)

## 搭建本地环境

Flink 支持两种部署模式——Session Mode 和 Application Mode

本节内容参考：[how to use Apache Flink with Docker.](https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/resource-providers/standalone/docker/#docker-setup)

### Session Mode

多个job共享JobManager

```
version: "2.2"
services:
  jobmanager:
    image: flink:latest
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager

  taskmanager:
    image: flink:latest
    depends_on:
      - jobmanager
    command: taskmanager
    scale: 1
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2
```

### Application Mode

```
version: "2.2"
services:
  jobmanager:
    image: flink:latest
    ports:
      - "8081:8081"
    command: standalone-job --job-classname com.job.ClassName [--job-id <job id>] [--fromSavepoint /path/to/savepoint [--allowNonRestoredState]] [job arguments]
    volumes:
      - /host/path/to/job/artifacts:/opt/flink/usrlib
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        parallelism.default: 2

  taskmanager:
    image: flink:latest
    depends_on:
      - jobmanager
    command: taskmanager
    scale: 1
    volumes:
      - /host/path/to/job/artifacts:/opt/flink/usrlib
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2
        parallelism.default: 2
```
