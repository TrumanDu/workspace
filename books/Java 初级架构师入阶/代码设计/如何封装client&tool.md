---
group:
  title: 代码设计
  order: 4
order: 3
---

# 如何封装 client/tool

## 单例

工具类，模板类，可以确定只用生成一个实例的，考虑这种设计模式，善用例模式。

在类中构造方法善用重载

### Demo

```text
public class ZooKeeperTemplate implements AutoCloseable {

    private static final Logger logger = LoggerFactory.getLogger(ZooKeeperTemplate.class);

    /** zookeeper session timeout: 30 seconds */
    private static final int DEFAULT_SESSION_TIMEOUT_MILLISECONDS = 30_000;
    /** zookeeper connect timeout: 60 seconds */
    private static final int DEFAULT_CONNECT_TIMEOUT_MILLISECONDS = 60_000;
    /** znode path separator */
    private static final String PATH_SEPARATOR = "/";

    public static ZooKeeperTemplate getInstance(String zooKeeperHost) {
        return getInstance(zooKeeperHost, null, null);
    }

    public static ZooKeeperTemplate getInstance(String zooKeeperHost, Integer sessionTimeoutMilliseconds, Integer connectTimeoutMilliseconds) {
        return new ZooKeeperTemplate(zooKeeperHost, sessionTimeoutMilliseconds, connectTimeoutMilliseconds);
    }

    private final String zooKeeperHost;
    private final Integer sessionTimeoutMilliseconds;
    private final Integer connectTimeoutMilliseconds;

    private final ZooKeeper zoo;

    private ZooKeeperTemplate(String zooKeeperHost, Integer sessionTimeoutMilliseconds, Integer connectTimeoutMilliseconds) {
        this.zooKeeperHost = zooKeeperHost;
        this.sessionTimeoutMilliseconds = defaultValue(sessionTimeoutMilliseconds, DEFAULT_SESSION_TIMEOUT_MILLISECONDS);
        this.connectTimeoutMilliseconds = defaultValue(connectTimeoutMilliseconds, DEFAULT_CONNECT_TIMEOUT_MILLISECONDS);
        this.zoo = connect();
    }

    //TODO 省略
  }
```

## Builder 构建器

这个设计模式也是数据创建型的，如果参与大于 4 个，推荐使用这种模式。

### Demo

```text
public class RedisTemplate implements Closeable {

    private JedisCluster jedisCluster;
    private String keyPrefix;

    private static String clientName = "xaecbd-sns";

    private RedisTemplate(RedisBuilder builder) {
        final String[] hosts = builder.hosts.split(RedisConstants.HOST_SPLIT);
        keyPrefix = builder.prefix;
        Set<HostAndPort> nodes = new HashSet<HostAndPort>() {
            private static final long serialVersionUID = 5341345879054512402L;

            {
                for (String hostAndPort : hosts) {
                    String[] array = hostAndPort.split(":");
                    if (array.length > 1) {
                        add(new HostAndPort(array[0], Integer.parseInt(array[1])));
                    } else {
                        add(new HostAndPort(array[0], RedisConstants.DEFAULT_PORT));
                    }
                }
            }
        };
        this.jedisCluster = new JedisCluster(nodes, builder.timeout, builder.timeout, builder.retry, builder.password, clientName,new GenericObjectPoolConfig<Jedis>());
    }

    //TODO 省略其他逻辑代码

    public static class Builder {
        private String hosts;
        private String prefix;
        private int timeout = BinaryJedisCluster.DEFAULT_TIMEOUT;
        private int retry = BinaryJedisCluster.DEFAULT_MAX_ATTEMPTS;
        private String password = null;

        public RedisBuilder(String hosts) {
            this.hosts = hosts;
        }

        public RedisBuilder timeout(int timeout) {
            this.timeout = timeout;
            return this;
        }

        public RedisBuilder retry(int retry) {
            this.retry = retry;
            return this;
        }

        public RedisBuilder password(String password) {
            this.password = password;
            return this;
        }

        public RedisBuilder keyPrefix(String prefix) {
            this.prefix = prefix;
            return this;
        }

        public RedisTemplate builder() {
            return new RedisTemplate(this);
        }
    }
}
```

## 封装技巧

### 返回值

使用 Optinal 包装返回值，避免返回值为 null

### 统一校验

在类中的多个方法都需要相同的参数校验的，抽象出统一校验，jdk 源码中就存在大量这样的例子。

下面举个例子：

```text
private void assertConnected() {
        if(zoo == null) {
            throw new SnsException("ZooKeeper is not connected");
        }
    }

    private void assertPathValid(String path) {
        PathUtils.validatePath(path);
}

public void delete(String path) {
        assertConnected();
        assertPathValid(path);
       //省略
    }

public void create(String path) {
        assertConnected();
        assertPathValid(path);
       //省略
    }
```

### AutoCloseable/Closeable

需要关闭资源的，记得实现 AutoCloseable/Closeable
