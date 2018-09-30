## JDK并发包LinkedBlockingQueue使用异常导致GC不可用


###  问题背景
2018年4月某服务提交全面性能测试，在系统达到瓶颈压力后。服务RT大幅度抖动，且监控平台发现该服务GC情况异常，具体表现为YGC单次高达150ms，且YGCT越来越高最终无法YGC，不断执行FGC，且应用最终不可用。

- **应用的整个GC过程：**
![GC情况](img/gc情况.png)

- **最后的GC不可用，每秒均执行FGC：**
![GC情况](img/gc情况2.png)

### 问题初步排查与猜测

遇到到问题后，首先怀疑出现了内存泄露问题，保持系统持续的高压力，执行内存dump操作，得到hprof结果，用MAT进行分析，MAT分析工具warning：LinkedBlockingQueue是可能存在的内存泄露对象。

来看看代码中使用了JDK并发包阻塞队列的地方：

```java
private ThreadPoolExecutor executor = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(),
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(),
            new DefaultThreadFactory("mset"));
```

该线程池使用的队列用于从DB查询数据后，以redis-mset的方式将数据写入redis，以确保下次请求读取的数据可以直接从预热后的redis中获取，而不用再次查询DB。

初步看逻辑没有问题，结合服务的实际表现与dump内存得到的警告信息，LinkedBlockingQueue这个队列是存在问题的，他的功能是把些执行过的请求得到的数据以队列暂存的形式写入到redis。那么会不会是队列poll的速度赶不上队列put的速度造成的呢。

至此，我们提出了这个猜想。

### 证实猜想

我们大胆的直接在mset使用ThreadPoolExecutor后打印executor的Queue长度：

```java
logger.error("[new cache setex]elapse={} [queue]={}",rt,executor.getQueue().size());
```


并继续重现问题，密切关注日志中的队列长度变化，得到了以下的结果：

- **日志中打印的LinkedBlockingQueue长度：**
![GC情况](img/queuesize.png)

至此，我们有很大的理由可以认为，YGCT不断增加，是因为CMS收集器使用的标记-复制算法，在本应用中需要标记的对象：LinkedBlockingQueue中的任务元素越来越多，才导致了标记时间的增加，同时，Y区每次GC后能够可用的堆大小也越来越小。最终导致了不断FGC，整个Y区不可用的情况。

### 问题解决

如果上述的猜想与判断，经过实际的优化后解决了该问题，那么才可以100%断定问题的原因。因此，我们修改了对并发包中无界阻塞队列的使用，改为使用有界的阻塞队列：

```java
private ThreadPoolExecutor executor = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(),
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(20000),
            new DefaultThreadFactory("mset"),
            new ThreadPoolExecutor.DiscardPolicy()
            );
```
其中的20000，就是我们暂定的队列长度。

- **GC恢复正常**
![GC情况](img/gc正常.png)

至此应用的GC恢复到了正常情况，压力表现中的TPS与RT也不在大幅度抖动，顺带锤实了业内一直错误的认知：YGC不会导致SWT。在我们的长年性能测试经验中，CMS收集器的YGC是有可能导致SWT的，具体需要看GC的对象。

20000的队列长度会导致在高压力时刻有的数据被线程池拒绝而无法被预热的情况。因此只是用于判断该问题，实际我们采用了SychronousQueque来替代LinkedBlockingQueue，最终解决了该问题。

### 总结回顾

- **本文总体阐述了一个GC不可用问题发生的背景**
- **使用内存dump与codereview的形式大致判断了代码异常点**
- **大胆猜测问题可能造成的原因并予以论证**
- **修改代码后再次尝试重现场景，问题不再发生**
- **得出结论1：CMS收集器YGC的标记时间、堆可用大小与并发包中的LinkedBlockingQueue母密切相关**
- **得出结论2：executor调用方的请求速度若超过executor的执行速度，将会导致队列不断增加，即同步请求需与异步请求保持一致**

