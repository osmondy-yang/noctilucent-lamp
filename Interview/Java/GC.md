## GC时间过长怎么办

### 对象创建的速度过高
  如果应用创建对象的速度非常高，随之而来的就是GC频率也会变快，然后会导致GC的停顿时间变长。所以说，优化代码以降低对象的创建速率是降低GC停顿时间最有效的方法。可以使用JProfiler, YourKit, JVisualVM这样的性能监控工具来帮助优化对象的创建速度，这些工具会分析出：应用到底创建了哪些对象？对象创建的速度是多少？这些对象占用了多少内存空间？是谁创建的这些对象？

> jps：查看所有的jvm进程，包括进程ID，进程启动的路径等等。
>
> jstack：查看jvm中当前所有线程的运行情况和线程当前状态。
>
> jstat：利用JVM内建的指令对Java应用程序的资源和性能进行实时的命令行的监控，包括了对进程的classloader，compiler，gc情况；特别的，可以用来监视VM内存内的各种堆和非堆的大小及其内存使用量，以及加载类的数量。
>
> jmap：查看jvm中，所有对象的情况，例如产生了哪些对象，对象数量。
>
> jinfo：查看进程运行环境参数，包括Java System属性和JVM命令行参数

### Young区过小
  如果Young过小，对象就会过早的晋升到Old区，Old区的垃圾回收一般比Young区会花费更多的时间，因此，可以通过增大Young区来有效的降低长时间GC停顿。可以用下面两个JVM参数来设置Young区的大小：

1. -Xmn: 设置Young区所占的字节数
2. -XX:NewRatio: 设置Old区和Young区的比例。eg: -XX:NewRatio=3 也就是说Old区和Young区的比例是3:1，
4. Young区占整个堆的1/4，如果堆是2G，那么Young区就是0.5G。

### 选择合适的GC算法
  GC算法是影响GC停顿时间的一个非常重要的因素。选择使用G1收集器，因为G1是自动调优的，只需要设置一个停顿时间的目标就可以了，比如： -XX:MaxGCPauseMillis=200。这个例子设置了最大停顿时间的目标是200ms，JVM会尽最大努力来满足这个目标。

### 进程被交换（Swap）出内存
  有时候由于系统内存不足，操作系统会把应用从内存中交换出去。Swap是非常耗时的，因为需要访问磁盘，相对于访问物理内存来说要慢得多的多。生产环境下的应用是不应该被Swap出内存的。当发生进程Swap的时候，GC停顿时间也会变长。如果应用被Swap了，你需要：
a:给机器增加内存
b:减少机器上运行的进程数，以释放更多的内存
c:减少应用分配的内存（不推荐，可能会引起其他问题）

### GC线程数过少
GC日志中的每一个GC事件都会打印 user、sys、real 的时间，比如：

```shell
[Times: user=25.56 sys=0.35, real=20.48 secs]
```


如果GC日志中，real time并不是明显比user time小，这就说明GC线程数是不够的，这就需要增加GC线程了。假如说，user time是25秒，GC线程数是5，那么real time大概是5左右才是正常的（25/5=5）。注意：GC线程过多会占用大量的系统CPU，从而会影响应用能使用的CPU资源。

### IO负载重
  如果系统的IO负载很重（大量的文件读写）也会导致GC停顿时间过长。这些IO读写不一定是应用引起的，可能是机器上其他的进程导致的，但是这仍然会导致应用的停顿时间变长。当IO负载很重的时候，real time会明显比user time长，比如：

```shell
[Times: user=0.20 sys=0.01, real=18.45 secs]
```


如果发生了这种情况，可以这么办：
a:如果是应用导致的，优化代码
b:如果是别的进程导致的，把它杀掉或者迁走
c:把应用迁到一个IO负载小的机器上
tip：如何来监控IO负载？在linux上可以用sar命令来监控IO的负载：sar -d -p 1，这个命令每隔一秒会打印一次每秒的读写数量。

### 显式调用了System.gc()
  当调用了System.gc()或者是Runtime.getRuntime().gc()以后，就会导致FullGC。FullGC的过程当中，整个JVM是暂停的（所有的应用都被暂停掉）。System.gc()可能是以下几种情况产生的：
a:应用的程序员手动调用了System.gc()
b:应用引用的三方库或者框架甚至是应用服务器可能调用了System.gc()
c:可能是由外部使用了JMX的工具触发
d:如果应用使用了RMI，RMI会每隔一段时间调用一次System.gc()，这个时间间隔是可以设置的：
– Dsun.rmi.dgc.server.gcInterval=n
– Dsun.rmi.dgc.client.gcInterval=n

### 什么情况下，会手动调用System.gc()
  一般来说，在编写Java代码并将其留给JVM时，不要考虑内存管理。在一些特殊情况下，如正在编写一个性能基准，可以在运行之间调用System.gc()。  MappedByteBuffer用于需要最佳IO性能的地方。MappedByteBuffer提供到底层文件的直接内存映射。内存映射文件的许多细节固有地依赖于底层操作系统，因此是未指定的。当请求的区域未完全包含在该通道的文件中时，此方法的行为是未指定的。对该程序或另一个的底层文件的内容或大小进行的更改是否被传播到缓冲区是未指定的。没有指定将缓冲区的更改传播到文件的速率。
  因为MappedByteBuffer依赖于操作系统，垃圾收集器不能立即回收。但是当调用System.gc()垃圾收集器释放句柄，可以删除该文件。在使用MappedByteBuffer的同时，如果我们正在使用内存敏感程序，那么最好调用System.gc()。

### 堆内存过大
  堆内存过大也会导致GC停顿时间过长，如果堆内存过大，那么堆中就会累计过多的垃圾，当发生FullGC要回收所有的垃圾的时候，就会花费更多的时间。如果你的JVM的堆内存有18G，可以考虑分成3个6G的JVM实例，堆内存小会降低GC的停顿时间。

### GC任务分配不均
  就算有多个GC线程，线程之间的任务分配可能也不是均衡的，这个可能有很多种原因：
a:扫描大的线性的数据结构目前是无法并行的。
b:有些GC事件只发生在单个线程上，比如CMS中的‘concurrent mode failure’。如果你碰巧使用的CMS，可以使用-XX:+CMSScavengeBeforeRemark 这个参数，它可以让多个GC线程之间任务分配的更平均。



## Refrence

[1]:https://tech.meituan.com/2017/12/29/jvm-optimize.html	"从实际案例聊聊Java应用的GC优化"