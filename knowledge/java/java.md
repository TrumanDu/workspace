# Java基础

## JVM内存分区

- 程序计数器
- 堆
- 本地栈
- 元空间（存类和类加载器的元数据）
- 虚拟机栈

## Java内存模型

调用栈和本地变量存放在线程栈上，对象存放在堆上。[<sup>1</sup>](#reference)

![](https://pic3.zhimg.com/80/v2-af520d543f0f4f205f822ec3b151ad46_720w.jpg)

## JVM内存回收算法

标记清除、引用计数、复制、标记压缩、分代回收、增量式回收

垃圾收集器（CMS、G1、ZGC）

## 基础知识

### equals() 与 ==

== : 它的作用是判断两个对象的地址是不是相等。即，判断两个对象是不是同一个对象。

equals() : 它的作用也是判断两个对象是否相等。但它一般有两种使用情况(前面第1部分已详细介绍过)：

情况1，类没有覆盖equals()方法。则通过equals()比较该类的两个对象时，等价于通过“==”比较这两个对象。

情况2，类覆盖了equals()方法。一般，我们都覆盖equals()方法来两个对象的内容相等；若它们的内容相等，则返回true(即，认为这两个对象相等)。

### hashCode

hashCode返回的并不一定是对象的（虚拟）内存地址，具体取决于运行时库和JVM的具体实现

开发中需要注意的是修改equals方法，记得重写hashCode方法

## Java 集合框架

### ArrayList

1.5倍扩容

### HashMap
#### HashMap的容量必须为2的N次方？

计算hash槽的时候可以将求余转换成按位与，提法计算速度。即capacity为2的N次方时hash%capacity==hash&(capacity-1)

#### Hashmap为什么是二倍扩容

capacity为2的幂次方，capacity-1的二进制会全为1，位运算时可以充分散列，避免不必要的哈希冲突。

计算hash函数，俗称扰动函数，就是说让高位参与到桶位置的计算，减少冲突。

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

更多查看[**Java集合框架常见面试题**](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/collection/Java%E9%9B%86%E5%90%88%E6%A1%86%E6%9E%B6%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98.md)

## 线程

在Java中有两类线程：User Thread(用户线程)、Daemon Thread(守护线程)

用户线程，不随着其他人的死亡而死亡，只有两种情况死掉，1.在run中异常终止，2.正常把run执行完毕。

守护线程，随着用户线程的死亡而死亡，当用户线程死完了守护线程也死了。

### 线程状态

- 初始(NEW)
- 运行(RUNNABLE)
- 阻塞(BLOCKED)
- 等待(WAITING)
- 超时等待(TIMED_WAITING)
- 终止(TERMINATED)

BLOCKED由于等待锁进入（无法进入同步方法/代码块来）阻塞状态，一般是synchronized

WAITING 当wait，join，park方法调用时，进入waiting状态。

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

1 当一个任务通过submit或者execute方法提交到线程池的时候，如果当前池中线程数（包括闲置线程）小于coolPoolSize，则创建一个线程执行该任务。

2 如果当前线程池中线程数已经达到coolPoolSize，则将任务放入等待队列。

3 如果任务不能入队，说明等待队列已满，若当前池中线程数小于maximumPoolSize，则创建一个临时线程（非核心线程）执行该任务。

4 如果当前池中线程数已经等于maximumPoolSize，此时无法执行该任务，根据拒绝执行策略处理。

注意：当池中线程数大于coolPoolSize，超过keepAliveTime时间的闲置线程会被回收掉。回收的是非核心线程，核心线程一般是不会回收的。如果设置allowCoreThreadTimeOut(true)，则核心线程在闲置keepAliveTime时间后也会被回收

## java锁

### synchronized

实例锁（修饰实例方法），同步代码块（实例锁），class 锁（修饰静态方法）

synchronized具有可重入性
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

在某些情况下优于synchronized

**(1)volatile保证可见性**

被volatile修饰之后，保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。

**(2)volatile无法保证原子性**

但是不能保证原子性，例如用volatitle修饰int，多线程自增，仍然无法保证获取想要的值。

自增操作是不具备原子性的，它包括读取变量的原始值、进行加1操作、写入工作内存。那么就是说自增操作的三个子操作可能会分割开执行。

### Lock

ReentrantLock和ReentrantReadWriteLock 都是可重入锁

ReentrantLock默认是非公平锁，通过构建函数参数true可以设置为公平锁。

ReentrantLock具有可重入性,可支持公平锁与非公平锁，支持响应中断

ReentrantReadWriteLock 读-读能共存，读-写不能共存，写-写不能共存(读锁读写互斥，写锁，写写互斥)

读写锁同时使用，适合读多写少，读写一致的场景

#### AQS

//TODO

## 并发容器

### ConcurrentHashMap

`CAS + synchronized`

put 流程

- 根据 key 计算出 hashcode 。
- 判断是否需要进行初始化。
- `f` 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
- 如果当前位置的 `hashcode == MOVED == -1`,则需要进行扩容。
- 如果都不满足，则利用 synchronized 锁写入数据。
- 如果数量大于 `TREEIFY_THRESHOLD=8` 则要转换为红黑树。

CopyOnWriteArrayList

ConcurrentLinkedQueue

ThreadLocal

BlockingQueue

<div id="reference"></div>

## 参考

[1] [Java内存模型（JMM）总结](https://zhuanlan.zhihu.com/p/29881777)

