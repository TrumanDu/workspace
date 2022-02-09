# Java NIO扫盲篇

## 概述

对于java世界，想了解高性能网络编程。那么就必须了解NIO,和常用的网络编程框架，以及高性能的网络编程模式。这里面缺一不可，本篇只是管中窥豹，将自己学习的过程和笔记记录下来，希望给工作中很难接触这方面的同学带来一点点帮助！

## Java NIO是什么？

NIO 是一种同步非阻塞的 I/O 模型，在 Java 1.4 中引入了 NIO 框架，对应 `java.nio` 包，提供了 `Channel` 、`Selector`、`Buffer` 等抽象。

NIO 中的 N 可以理解为 Non-blocking，不单纯是 New。它支持面向缓冲的，基于通道的 I/O 操作方法。 NIO 提供了与传统 BIO 模型中的 `Socket` 和 `ServerSocket` 相对应的 `SocketChannel` 和 `ServerSocketChannel` 两种不同的套接字通道实现,两种通道都支持阻塞和非阻塞两种模式。阻塞模式使用就像传统中的支持一样，比较简单，但是性能和可靠性都不好；非阻塞模式正好与之相反。对于低负载、低并发的应用程序，可以使用同步阻塞 I/O 来提升开发速率和更好的维护性；对于高负载、高并发的（网络）应用，应使用 NIO 的非阻塞模式来开发。

### NIO 的基本流程

通常来说 NIO 中的所有 IO 都是从 Channel（通道） 开始的。

- 从通道进行数据读取 ：创建一个缓冲区，然后请求通道读取数据。
- 从通道进行数据写入 ：创建一个缓冲区，填充数据，并要求通道写入数据。

### NIO 核心组件

NIO 包含下面几个核心的组件：

- **Channel(通道)**
- **Buffer(缓冲区)**
- **Selector(选择器)**

通道与流的不同之处在于通道是双向的。而流只是在一个方向上移动(一个流必须是 `InputStream` 或者 `OutputStream` 的子类)， 而 `通道 `可以用于读、写或者同时用于读写。

## 如何使用？

### 内存映射文件 I/O

内存映射文件 I/O 是一种读和写文件数据的方法，它可以比常规的基于流或者基于通道的 I/O 快得多。

下面代码行将文件的前 1024 个字节映射到内存中，map() 方法返回一个 MappedByteBuffer，它是 ByteBuffer 的子类。因此，可以像使用其他任何 ByteBuffer 一样使用新映射的缓冲区，操作系统会在需要时负责执行映射。

```
MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_WRITE, 0, 1024);
```

这里多说一句，在Kafka源码中索引就大量使用了内存映射文件。

### 文件NIO

#### 读文件

读取文件涉及三个步骤：(1) 从 `FileInputStream` 获取 `Channel`，(2) 创建 `Buffer`，(3) 将数据从 `Channel` 读到 `Buffer `中

第一步：获取通道

```
FileInputStream fis = new FileInputStream("C:\\demo.txt");
FileChannel fileChannel = fis.getChannel();
```

然后创建Buffer,这里选择的间接缓冲

```
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
//ByteBuffer buffer = ByteBuffer.allocateDirect( 1024 ); 直接缓存,速度更快，JVM层面尽肯能避免多次拷贝
```

最后将数据从 `Channel` 读到 `Buffer `中

```
fileChannel.read(byteBuffer)
```



#### 写文件

几乎和读文件一样。

首先从 `FileOutputStream` 获取一个通道

```
FileOutputStream fos = new FileOutputStream("C:\\demo.txt");
FileChannel channel = fos.getChannel();
```

下一步是创建一个缓冲区并在其中放入一些数据

```
        ByteBuffer buffer = ByteBuffer.allocate( 1024 );
        for (int i=0; i<byteBuffer.limit(); ++i) {
            buffer.put( byteBuffer.get(i));
        }

        buffer.flip();
```

最后一步是写入缓冲区中：

```
 channel.write(buffer);
```

### 套接字NIO

**NIO 实现了 IO 多路复用中的 Reactor 模型**：

- 一个线程（`Thread`）使用一个**选择器 `Selector` 通过轮询的方式去监听多个通道 `Channel` 上的事件**，从而让一个线程就可以处理多个事件。
- 通过**配置监听的通道 `Channel` 为非阻塞**，那么当 `Channel` 上的 IO 事件还未到达时，就不会进入阻塞状态一直等待，而是继续轮询其它 `Channel`，找到 IO 事件已经到达的 `Channel` 执行。

#### 创建选择器

```
       Selector selector = Selector.open();
```

#### 将通道注册到选择器上

```
        ServerSocketChannel channel = ServerSocketChannel.open();
        channel.configureBlocking(false);
        channel.register(selector, SelectionKey.OP_ACCEPT);
```

注册的具体事件，主要有以下几类：

- `SelectionKey.OP_CONNECT`
- `SelectionKey.OP_ACCEPT`
- `SelectionKey.OP_READ`
- `SelectionKey.OP_WRITE`

#### 监听事件

```
int num = selector.select();
```

使用 `select()` 来监听到达的事件，它会一直阻塞直到有至少一个事件到达。

#### 处理事件

```
while (true) {
    int num = selector.select();
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = keys.iterator();
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        if (key.isAcceptable()) {
            // ...
        } else if (key.isReadable()) {
            // ...
        }
        keyIterator.remove();
    }
}
```



## 高性能网络编程

### Reactor模式

![](https://images0.cnblogs.com/blog2015/434101/201503/112105210899525.jpg)

### Reactor模式+任务池模式

![](https://images0.cnblogs.com/blog2015/434101/201503/112148547777256.jpg)

### 多Reactor模式

![](https://images0.cnblogs.com/blog2015/434101/201503/112151380898648.jpg)

## 参考

1. [Java NIO](https://github.com/dunwu/javacore/blob/master/docs/io/java-nio.md)

2. [Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)

3. [《Scalable IO in Java》笔记](https://www.cnblogs.com/luxiaoxun/p/4331110.html)