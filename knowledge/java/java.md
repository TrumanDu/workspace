# Java 基础

## JVM 内存分区

-   程序计数器
-   堆
-   本地栈
-   元空间（存类和类加载器的元数据）
-   虚拟机栈

## Java 内存模型

调用栈和本地变量存放在线程栈上，对象存放在堆上。[<sup>1</sup>](#reference)

![](https://pic3.zhimg.com/80/v2-af520d543f0f4f205f822ec3b151ad46_720w.jpg)

## JVM 内存回收算法

标记清除、引用计数、复制、标记压缩、分代回收、增量式回收

垃圾收集器（CMS、G1、ZGC）

## 基础知识

### equals() 与 ==

== : 它的作用是判断两个对象的地址是不是相等。即，判断两个对象是不是同一个对象。

equals() : 它的作用也是判断两个对象是否相等。但它一般有两种使用情况(前面第 1 部分已详细介绍过)：

情况 1，类没有覆盖 equals()方法。则通过 equals()比较该类的两个对象时，等价于通过“==”比较这两个对象。

情况 2，类覆盖了 equals()方法。一般，我们都覆盖 equals()方法来两个对象的内容相等；若它们的内容相等，则返回 true(即，认为这两个对象相等)。

### hashCode

hashCode 返回的并不一定是对象的（虚拟）内存地址，具体取决于运行时库和 JVM 的具体实现

开发中需要注意的是修改 equals 方法，记得重写 hashCode 方法

## Java 集合框架

### ArrayList

1.5 倍扩容

### HashMap

#### HashMap 的容量必须为 2 的 N 次方？

计算 hash 槽的时候可以将求余转换成按位与，提法计算速度。即 capacity 为 2 的 N 次方时 hash%capacity==hash&(capacity-1)

#### Hashmap 为什么是二倍扩容

capacity 为 2 的幂次方，capacity-1 的二进制会全为 1，位运算时可以充分散列，避免不必要的哈希冲突。

计算 hash 函数，俗称扰动函数，就是说让高位参与到桶位置的计算，减少冲突。

```
static final int hash(Object key) {
        int h;
        // ^ ：按位异或
        // >>>:无符号右移，忽略符号位，空位都以0补齐
        // hashcode与高16位异或,生成的数据带有高位特性，参与桶计算的时候，可以起到扰动
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

#### LinkedHashMap

数组+双向循环链表

更多查看[**Java 集合框架常见面试题**](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/collection/Java%E9%9B%86%E5%90%88%E6%A1%86%E6%9E%B6%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98.md)

## 线程

在 Java 中有两类线程：User Thread(用户线程)、Daemon Thread(守护线程)

用户线程，不随着其他人的死亡而死亡，只有两种情况死掉，1.在 run 中异常终止，2.正常把 run 执行完毕。

守护线程，随着用户线程的死亡而死亡，当用户线程死完了守护线程也死了。

### 线程状态

-   初始(NEW)
-   运行(RUNNABLE)
-   阻塞(BLOCKED)
-   等待(WAITING)
-   超时等待(TIMED_WAITING)
-   终止(TERMINATED)

BLOCKED 由于等待锁进入（无法进入同步方法/代码块来）阻塞状态，一般是 synchronized

WAITING 当 wait，join，park 方法调用时，进入 waiting 状态。

### 线程池

利用线程工厂类可以指定线程池中线程名称

```java

ThreadFactory threadFactory = new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "thread_pool_" + r.hashCode());
            }
        };
```

**创建线程池方式**

```java
ExecutorService executorService = Executors.newSingleThreadExecutor(threadFactory);
ExecutorService executorService = Executors.newCachedThreadPool(threadFactory);
ExecutorService executorService = Executors.newFixedThreadPool(10,threadFactory);
ExecutorService executorService = Executors.newScheduledThreadPool(10,threadFactory);
```

```
ExecutorService executorService = new ThreadPoolExecutor(10,20,50, TimeUnit.MILLISECONDS,new ArrayBlockingQueue<>(100),Executors.defaultThreadFactory());
```

### 线程执行流程

1 当一个任务通过 submit 或者 execute 方法提交到线程池的时候，如果当前池中线程数（包括闲置线程）小于 coolPoolSize，则创建一个线程执行该任务。

2 如果当前线程池中线程数已经达到 coolPoolSize，则将任务放入等待队列。

3 如果任务不能入队，说明等待队列已满，若当前池中线程数小于 maximumPoolSize，则创建一个临时线程（非核心线程）执行该任务。

4 如果当前池中线程数已经等于 maximumPoolSize，此时无法执行该任务，根据拒绝执行策略处理。

注意：当池中线程数大于 coolPoolSize，超过 keepAliveTime 时间的闲置线程会被回收掉。回收的是非核心线程，核心线程一般是不会回收的。如果设置 allowCoreThreadTimeOut(true)，则核心线程在闲置 keepAliveTime 时间后也会被回收

### 虚拟线程

虚拟线程可以解决增加系统吞吐量问题，虚拟线程更轻量。

```
// 两种写法
Thread.startVirtualThread(runnable).join();
Thread.ofVirtual().name("virtual-thread").start(runnable).join();
// 虚拟线程工厂
ThreadFactory virtualThreadFactory = Thread.ofVirtual().name("v").factory();
virtualThreadFactory.newThread(runnable).start();
// 第三种写法
try (ExecutorService executorService = Executors.newVirtualThreadPerTaskExecutor()){
    IntStream.range(0,1000).forEach(i->{
        executorService.execute(()->{
            try {
                Thread.sleep(Duration.ofSeconds(1));
                System.out.println(i);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
    });
```

## java 锁

### synchronized

实例锁（修饰实例方法），同步代码块（实例锁），class 锁（修饰静态方法）

synchronized 具有可重入性

```
classMyClass {
 public synchronized void method1() {
     method2();
 }
 public synchronized void method2() {
 }
}
```

### volatile

在某些情况下优于 synchronized

**(1)volatile 保证可见性**

被 volatile 修饰之后，保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。

**(2)volatile 无法保证原子性**

但是不能保证原子性，例如用 volatitle 修饰 int，多线程自增，仍然无法保证获取想要的值。

自增操作是不具备原子性的，它包括读取变量的原始值、进行加 1 操作、写入工作内存。那么就是说自增操作的三个子操作可能会分割开执行。

### Lock

ReentrantLock 和 ReentrantReadWriteLock 都是可重入锁

ReentrantLock 默认是非公平锁，通过构建函数参数 true 可以设置为公平锁。

ReentrantLock 具有可重入性,可支持公平锁与非公平锁，支持响应中断。

ReentrantReadWriteLock 读-读能共存，读-写不能共存，写-写不能共存(读锁读写互斥，写锁，写写互斥)

读写锁同时使用，适合读多写少，读写一致的场景

#### AQS

//TODO

更多信息详见：[深入理解 Java 并发锁](https://dunwu.github.io/javacore/concurrent/Java%E9%94%81.html)

## 并发容器

### ConcurrentHashMap

`CAS + synchronized`

put 流程

-   根据 key 计算出 hashcode 。
-   判断是否需要进行初始化。
-   `f` 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
-   如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容。
-   如果都不满足，则利用 synchronized 锁写入数据。
-   如果数量大于 `TREEIFY_THRESHOLD=8` 则要转换为红黑树。

CopyOnWriteArrayList

ConcurrentLinkedQueue

ThreadLocal

BlockingQueue

## 新特性

-   虚拟线程
-   有序集合增加通用操作
-   Record 模式
-   switch 模式匹配类型
-   字符串模板（预览）STR,RAW
-   结构化并发（预览）解决复杂并发编写复杂性问题

## 资源

1. [聊聊 JVM](https://mp.weixin.qq.com/s?__biz=MzkwNjMwMTgzMQ==&mid=2247495919&idx=1&sn=71010d91e376270afc31bbe61c8326aa&chksm=c0e82807f79fa111d7c339832f48c9542d5fc1540c1a619c1a3281040fc1631616e016d0dc36&mpshare=1&scene=1&srcid=0629v5c0bjrW8BO2PNq5rAZc&sharer_sharetime=1656463160600&sharer_shareid=8bd71a4056686012698a4daf0af0595e#rd)

## 参考

[1] [Java 内存模型（JMM）总结](https://zhuanlan.zhihu.com/p/29881777)
