参考： [JVM（Java虚拟机）优化大全和案例实战](http://blog.csdn.net/kthq/article/details/8618052)

|参数名称|描述|备注|
|:---|:----|:----|
|-Xmx|最大堆内存|如-Xmx1024m就是配置最大堆内存为1024MB|
|-Xms|初始堆内存|这个值可以和Xmx配置成相同的，避免每次回收重新分配内存|
|-Xss|线程栈大小 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|设置每个线程的栈大小。JDK5.0以后每个线程栈大小为1M，之前每个线程栈大小为256K。应当根据应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。需要注意的是：当这个值被设置的较大（例如>2MB）时将会在很大程度上降低系统的性能|
|-Xmn|年轻代大小|如设置年轻代大小为2G-Xmn2G，这个值设置了的话就可以不用设置-XX:NewSize和XX:MaxNewSize，它可以直接配置年轻代的大小|
|-XX:NewSize|年轻代初始值|如设置年轻代初始值为100M：-XX:NewSize=100m|
|-XX:MaxNewSize|年轻代最大值|如设置年轻代上限为100M：-XX:MaxNewSize=100m|
|~~-XX:PermSize~~|持久代初始值|JDK8已经废弃了持久代了|
|~~-XX:MaxPermSize~~|持久代最大值|JDK8已经废弃了持久代了|
|-XX:MetaspaceSize|非堆区初始值|如初始值为2G，-XX:MetaspaceSize=2G|
|-XX:MaxMetaspaceSize|非堆区初始最大值|如最大值配置为2G，-XX:MaxMetaspaceSize=2G|
|-XX:NewRatio|年轻代与老年代的比值|如-XX:NewRatio=2，表示年轻代：老年代=1:2|
|-XX:SurvivorRatio|设置年轻代中的新生代与幸存区的比值|如-XX:SurvivorRatio=3，表示eden:survivor=3:2,为什么不是3:1呢，年轻代中有两个相同的Survivor区和一个eden区|
|-XX:MaxTenuringThreshold|最大挽留次数|表示一个对象如果在Survivor区（救助空间）移动了7次还没有被垃圾回收就进入年老代。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代，对于需要大量常驻内存的应用，这样做可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象在年轻代存活时间，增加对象在年轻代被垃圾回收的概率，减少Full GC的频率，这样做可以在某种程度上提高服务稳定性。|
|-XX:+UseSerialGC|--|设置串行收集器|
|-XX:+UseParallelGC|--|设置为并行收集器，这个配置仅对年轻代有效，对老年代仍旧采用串行收集|
|-XX:ParallelGCThreads|并行收集器线程数目|-XX:ParallelGCThreads=20，20个线程同时进行垃圾回收，建议与CPU数目相同|
|-XX:+UseParallelOldGC|--|配置年老代垃圾收集方式为并行收集。JDK6.0开始支持对年老代并行收集。|
|-XX:MaxGCPauseMillis|年轻代垃圾回收最长时间|-XX:MaxGCPauseMillis=100，最大回收时间100毫秒，如果无法满足此时间，JVM会自动调整年轻代大小，以满足此时间。|
|-XX:+UseAdaptiveSizePolicy|--|设置此选项后，并行收集器会自动调整年轻代Eden区大小和Survivor区大小的比例，以达成目标系统规定的最低响应时间或者收集频率等指标。此参数建议在使用并行收集器时，一直打开。|
|-XX:+UseConcMarkSweepGC|--|CMS垃圾回收器，它的主要适合场景是对响应时间的重要性需求大于对吞吐量的需求，能够承受垃圾回收线程和应用线程共享CPU资源，并且应用中存在比较多的长生命周期对象。CMS收集的目标是尽量减少应用的暂停时间，减少Full GC发生的几率，利用和应用程序线程并发的垃圾回收线程来标记清除年老代内存|
|-XX:+UseParNewGC|--|设置年轻代为并发收集。可与CMS收集同时使用。JDK5.0以上，JVM会根据系统配置自行设置，所以无需再设置此参数。|
|-XX:CMSFullGCsBeforeCompaction|运行n次之后整理内存碎片|-XX:CMSFullGCsBeforeCompaction=5，由于并发收集器不对内存空间进行压缩和整理，所以运行一段时间并行收集以后会产生内存碎片，内存使用效率降低。此参数设置为0的话，运行0次Full GC后对内存空间进行压缩和整理，即每次Full GC后立刻开始压缩和整理内存|
|-XX:+UseCMSCompactAtFullCollection|--|打开内存空间的压缩和整理，在Full GC后执行。可能会影响性能，但可以消除内存碎片。|
|-XX:+CMSIncrementalMode|--|设置为增量收集模式。一般适用于单CPU情况。|
|-XX:CMSInitiatingOccupancyFraction|--|-XX:CMSInitiatingOccupancyFraction=70，表示年老代内存空间使用到70%时就开始执行CMS收集，以确保年老代有足够的空间接纳来自年轻代的对象，避免Full GC的发生。|
