---
group:
  title: 基础知识
  order: 2
order: 1
---

# Spring 必知必会的技术

## 参数配置

常用的有两种方式：@Value 和@ConfigurationProperties

| Feature                                                                                                                                                              | `@ConfigurationProperties` | `@Value`                                                                                                                                                                               |
| :------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Relaxed binding](https://docs.spring.io/spring-boot/docs/3.0.0-M5/reference/htmlsingle/#features.external-config.typesafe-configuration-properties.relaxed-binding) | Yes                        | Limited (see [note below](https://docs.spring.io/spring-boot/docs/3.0.0-M5/reference/htmlsingle/#features.external-config.typesafe-configuration-properties.vs-value-annotation.note)) |
| [Meta-data support](https://docs.spring.io/spring-boot/docs/3.0.0-M5/reference/htmlsingle/#appendix.configuration-metadata)                                          | Yes                        | No                                                                                                                                                                                     |
| `SpEL` evaluation                                                                                                                                                    | No                         | Yes                                                                                                                                                                                    |

### @Value

首先需要在配置类上增加`@Configuration`

带默认值的

```
 @Value("${kafka.producer.retry:1}")
 private int retry;
```

使用 Spring Expression Language (SpEL)

```
@Value("#{'${actuator.send.business.type:eims,item}'.split(',')}")
List<String> specifyType;
```

### @ConfigurationProperties

该方式可以将相应的配置封装在具体类内，对代码组织和封装友好，推进使用这种方式

```
@ConfigurationProperties(prefix = "my.server")
public class MyServerProperties {
    private String name="test";
    @NotNull
    private String bootstrapServers;
    private Host host;
    // getters/setters ...
    // 嵌入式结构
    public static class Host {
        private String ip;
        private int port;
        // getters/setters ...
    }
}
```

```
my.server:
  name: test
  bootstrap_servers: xxxxx:9092
  host:
    ip: xxx
    port: 8081
```

**扩展：**

为了让 spring 容器可以识别该配置类，有多种方式，这里我推荐在启动类上添加注解`@ConfigurationPropertiesScan({"top.trumandu.config"})`

**配置参数**

Spring（relaxed binding ）使用一些宽松的绑定属性规则。因此，以下变体都将绑定到 hostName 属性上:

```
mail.hostName=localhost
mail.hostname=localhost
mail.host_name=localhost
mail.host-name=localhost
mail.HOST_NAME=localhost
```

为了让 ide 可以识别我们自定义的配置文件，可以添加 spring-boot-configuration-processor，这样就可以自动生成`additional-spring-configuration-metadata.json`, 进而进行提示。

### 配置文件优先级

`classpath root` > `classpath/config` > `current directory` > `current directory/config/` > `current directory/../config`

## 日志

默认使用 Logback,通常我们只用如下配置即可快速使用.

```
logging:
  level:
    root: "warn"
    org.springframework.web: "debug"
    org.hibernate: "error"
```

当无法满足一些日志场景时，Logback 扩展可以通过在`src/main/resources` 创建`logback-spring.xml`

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
	<!--<jmxConfigurator/> -->
	<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>
				%d{ISO8601} %-5level [%thread] %logger{0}: %msg%n
			</pattern>
		</encoder>
	</appender>

	<appender name="FILE"
		class="ch.qos.logback.core.rolling.RollingFileAppender">
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<fileNamePattern>logs/info.%d{yyyy-MM-dd}.log</fileNamePattern>
			<maxHistory>7</maxHistory>
		</rollingPolicy>
		<encoder>
			<pattern>
				%d{ISO8601} %-5level [%thread] %logger{0}: %msg%n
			</pattern>
		</encoder>
	</appender>

	<appender name="ERROR_FILE"
		class="ch.qos.logback.core.rolling.RollingFileAppender">
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<fileNamePattern>logs/error.%d{yyyy-MM-dd}.log</fileNamePattern>
			<maxHistory>7</maxHistory>
		</rollingPolicy>
		<encoder>
			<pattern>
				%d{ISO8601} %-5level [%thread] %logger{0}: %msg%n
			</pattern>
		</encoder>
		<filter class="ch.qos.logback.classic.filter.ThresholdFilter">
			<level>error</level>
		</filter>
	</appender>

	<logger name="top.trumandu" level="info" />
	<logger name="org.apache" level="error" />
	<logger name="org.I0Itec.zkclient" level="error" />
	<root level="info">
		<appender-ref ref="STDOUT" />
		<appender-ref ref="FILE" />
		<appender-ref ref="ERROR_FILE" />
	</root>
</configuration>
```

## Spring 容器启动以后运行任务

可以实现`ApplicationRunner`和`CommandLineRunner` 接口

```
@Component
public class MyCommandLineRunner implements CommandLineRunner {

    @Override
    public void run(String... args) {
        // Do something...
    }
}
```

优先级可以通过`@Ordered `实现

## 定时 job

首先启动定时注解配置

```
@EnableScheduling
```

通过以下代码可以快速实现定时 job

```
@Component
public class ManualGcJob {
    // 每两秒运行一次
    @Scheduled(cron = "0/2 * * * * ?")
    public void scheduleJvmGc(){
        System.out.println("manual gc");
    }
}
```

不仅通过 corn 表达式，也可以通过 fixedDelay 和 fixedRate 配置

支持配置文件配置参数

```
@Scheduled(fixedDelayString = "${category.fixed.delay:300000}", initialDelayString = "${category.fixed.delay:300000}")
public void updateCategory() {
    System.out.println("update category");
}
```

程序硬编码指定

```
@Scheduled(fixedDelay = 24 * 60 * 60 * 1000, initialDelay = 24 * 60 * 60 * 1000)
public void scheduleJvmGc() {
    System.out.println("manual gc");
}
```

fixedDelay 和 fixedRate 区别：fixedDelay需要等上一个任务完成后，再延迟相应时间才运行。

spring boot scheduling相关配置如下：
```
spring:
  task:
    scheduling:
      thread-name-prefix: "scheduling-"
      pool:
        size: 2
```

## 参考

1. [11 个 Spring 最常用的功能扩展点，手摸手实战](https://mp.weixin.qq.com/s?__biz=MzAxNTM4NzAyNg==&mid=2247501376&idx=1&sn=d6d8d43c46db709f096cebcb5bb1359c&chksm=9b8656bdacf1dfab780011978a2282635e0f01228f4aadf9cfe6f7a86e50addb6a0fd493e55f)
