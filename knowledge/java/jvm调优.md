# jvm调优

## 常用垃圾回收器

- CMS
- G1
- ZGC

通过如下命令，可以查看垃圾回收器的使用情况

```
$ java -XX:+PrintCommandLineFlags -version
```

jdk1.8 如下：

```
-XX:InitialHeapSize=534944128
-XX:MaxHeapSize=8559106048
-XX:+PrintCommandLineFlags
-XX:+UseCompressedClassPointers
-XX:+UseCompressedOops
-XX:-UseLargePagesIndividualAllocation 
-XX:+UseParallelGC
openjdk version "1.8.0_181-1-redhat"
OpenJDK Runtime Environment (build 1.8.0_181-1-redhat-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
```

jdk1.8 `-XX:+UseParallelGC` 默认使用的是Parallel垃圾回收器

jdk11 如下：

```
-XX:G1ConcRefinementThreads=10 -XX:GCDrainStackTargetSize=64 -XX:InitialHeapSize=534944128 -XX:MaxHeapSize=8559106048 -XX:+PrintCommandLineFlags -XX:ReservedCodeCacheSize=251658240 -XX:+SegmentedCodeCache -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseG1GC -XX:-UseLargePagesIndividualAllocation
openjdk version "11.0.6" 2020-01-14
OpenJDK Runtime Environment AdoptOpenJDK (build 11.0.6+10)
OpenJDK 64-Bit Server VM AdoptOpenJDK (build 11.0.6+10, mixed mode)
```

jdk11 `-XX:+UseG1GC`默认使用的垃圾回收器是**G1** 

## 理解JVM垃圾回收有用的资源

1. [新一代垃圾回收器ZGC的探索与实践](https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html)
2. [JVM系列--还不会选择合适的垃圾收集器？](https://cloud.tencent.com/developer/article/1619330)
3. [Java——七种垃圾收集器+JDK11最新ZGC](https://blog.csdn.net/CrankZ/article/details/86009279)
4. [JVM调优](https://github.com/thestar111/study/blob/master/JVM%E8%B0%83%E4%BC%98%E6%95%B4%E7%90%86/JVM%E8%B0%83%E4%BC%98.md)

## jvm启动参数

java启动参数共分为三类；
其一是**标准参数**（-），所有的JVM实现都必须实现这些参数的功能，而且向后兼容；
其二是**非标准参数**（-X），默认jvm实现这些参数的功能，但是并不保证所有jvm实现都满足，且不保证向后兼容；
其三是**非Stable参数**（-XX），此类参数各个jvm实现会有所不同，将来可能会随时取消，需要慎重使用；

### 常用非标准参数

通过` java -X` 可以查看支持的非标准参数

```
-Xms6g //最小堆
-Xmx6g //最大堆
-Xss1m //线程栈的大小
-Xmn200m //设置年轻代大小为200M
```

### 常用非Stable参数

通过`java -XX:+PrintFlagsFinal -version`可以查看支持的非Stable参数

- `-XX:+<option>`：代表启用 true
- `-XX:-<option>`：代表禁用 false

```
-XX:+PrintGC
-XX:+PrintGCDetails
```



## G1

G1全称为Garbage-First，意为垃圾优先，哪一块的垃圾最多就优先清理它。

### 常用参数

```
-XX:+UseG1GC //使用G1回收器
-XX:MaxGCPauseMillis=20 // 最大暂停时间为20ms 越小意味着堆小，回收频繁
```

## ZGC

[ZGC](https://wiki.openjdk.java.net/display/zgc/Main)（The Z Garbage Collector）是JDK 11中推出的一款低延迟垃圾回收器，它的设计目标包括：

- 停顿时间不超过10ms；
- 停顿时间不会随着堆的大小，或者活跃对象的大小而增加；
- 支持8MB~4TB级别的堆（未来支持16TB）。

```
-XX:+UnlockExperimentalVMOptions -XX:+UseZGC 
-XX:ConcGCThreads=2 -XX:ParallelGCThreads=6 
```

超低延迟（TP999 < 20ms）和高延迟（TP999 > 200ms）服务收益不大,吞吐量优先的场景，ZGC可能并不适合。

更多信息参考：[新一代垃圾回收器ZGC的探索与实践](https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html)

## jvm性能查看工具

### jstat 

利用jvm内建的指令对java应用程序的资源和性能进行实时的命令行监控，包括针对heap size和垃圾回收情况的监控

```
$ jstat -gc 14724 1000 2 //间隔1s,统计两次
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
19968.0 22528.0  0.0    0.0   429056.0 185319.3  395264.0   33163.2   59160.0 55725.5 7808.0 7049.8      8    0.063   3      0.156    0.219
19968.0 22528.0  0.0    0.0   429056.0 185855.6  395264.0   33163.2   59160.0 55725.5 7808.0 7049.8      8    0.063   3      0.156    0.219
```

参数含义：

S0C:年轻代的第一个survicor(幸存区)的容量(字节)

S1C:年轻代的第二个survicor(幸存区)的容量(字节)

S0U:年轻代的第一个survicor(幸存区)已经使用的空间(字节)

S1U:年轻代的第二个survicor(幸存区)已经使用的空间(字节)

EC:年轻代中的Eden(伊甸园)的容量

EU:年轻代中的Een(伊甸园)已经使用的容量

OC:年老代的容量

OU:年老代已经使用的空间

MC:metadata的容量

MU:metadata已经使用的空间

CCSC:压缩类空间的大小

CCSU:压缩类空间的使用大小

YG:从应用程序启动到采样时年轻代中gc次数

YGCT:从应用程序启动到采样时年轻代中gc所用时间(s)

FGC:从应用程序启动到采样时年老代gc次数

FGCT:从应用程序启动到采样时年老代(全gc)gc所用时间(s)

GC:	从应用程序启动到采样时gc用的总时间(s)


### jmap

查看jvm的内存分配情况

```
$ jmap -heap 14724
$ jmap -dump:format=b,file=D:\test\heap.hprof 14724
$ jmap -histo:live 14724 //只统计活的对象数量
```

### jstack

查看java程序的线程运行情况

```
$ jstack 14724
```

### jinfo

```
$ jinfo 14724 //查看jvm系统详细参数及环境变量
$ jinfo -flag MaxHeapSize 14724 //查看最大堆内存
```



## 最佳实践

### 输出GC日志

jdk11 

```
java -Xlog:gc*:file=gc.log,filecount=10,filesize=10m
```



jdk8

```
-XX:+PrintGCDetails -Xloggc:<PATH_TO_GC_LOG_FILE>
```



### dump

```
#出现 OOM 时生成堆 dump:
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/home/jvmlogs/
```



