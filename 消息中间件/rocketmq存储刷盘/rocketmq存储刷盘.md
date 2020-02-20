因为业务需要，所以学习了两天相关的源码，当有同事问我相关存储的机制时，我写了这个文档
因为是给同行看的，所以这里就直接开始看源码（大篇幅的贴源代码看着会有点费劲，如果有兴趣可以根据方法名去github上下载源代码来看）

``` java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {     
………........……….
调用mapedFile，写文件
 result = mapedFile.appendMessage(msg, this.appendMessageCallback);
…………....……………….
}

public AppendMessageResult appendMessage(final Object msg, final AppendMessageCallback cb) {
cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, msg);
}

public AppendMessageResult doAppend(final long fileFromOffset,
                     final ByteBuffer byteBuffer, final int maxBlank, final Object msg) {
     // Write messages to the queue buffer
    byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);
}
```
msgStoreItemMemory是一个内存映射文件MappedByteBuffer
将filechannle中的全部或者部分映射到内存中
MappedByteBuffer mbb = fc.map( FileChannel.MapMode.READ_WRITE, 0, 1024 );
好了，方法调用到这里你可以发现，消息已经被存储到了内存中，还没有被持久化，所以我们继续研究。
消息持久化呢有两种方式，同步刷盘和异步刷盘
我们直接来看源码

``` java
if (FlushDiskType.SYNC_FLUSH== this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
      GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
          …….
        启用同步刷盘线程进行刷盘
                service.wakeup();
        …….
        // Asynchronous flush
        else {
        启用flushCommitLogService线程，进行异步刷盘喽
            this.flushCommitLogService.wakeup();
 }
```
下面先讲解一下同步刷盘

``` java
GroupCommitService service = (GroupCommitService) this.flushCommitLogService
private void doCommit() {
        ……..
       线程调用这个方法将文件持久化写入硬盘中
       CommitLog.this.mapedFileQueue.commit(0);
       …………
}

CommitLog初始化

this.mapedFileQueue = new MapedFileQueue(
           defaultMessageStore.getMessageStoreConfig().getStorePathCommitLog(),
           defaultMessageStore.getMessageStoreConfig().getMapedFileSizeCommitLog(), 
           defaultMessageStore.getAllocateMapedFileService());
说白了就是
this.mapedFileQueue = new MapedFileQueue(文件路径，文件大小，存文件的实际服务，叫它方法也行)
看完初始化，揭开commit的神秘面纱

public boolean commit(final int flushLeastPages) {
        MapedFile mapedFile = this.findMapedFileByOffset(this.committedWhere, true);
            int offset = mapedFile.commit(flushLeastPages);
}

到这里我们还没有看到MapedFile没有初始化，它在MapedFileQueue中的load方法中初始化

public boolean load() {
  MapedFile mapedFile = new MapedFile(file.getPath(), mapedFileSize);
}

  //刷盘，内存到硬盘                          
public int commit(final int flushLeastPages) {
    this.mappedByteBuffer.force();
}
```
但是同步刷盘会影响我们消息收发的整体性能，你想一下，如果你发一条数据，就调用io写入硬盘，当然只有一条数据，确实没问题，但是数量大的时候，开启大量线程，或者线程池中的线程被占用来写数据了，那肯定会影响它收发消息了，毕竟系统的整体性能是固定的。

这个时候就要提到异步刷盘的概念了，即收集一批数据后在写入硬盘中，可以极大的降低所消耗的性能。那么什么时候进行异步刷盘呢？异步刷盘又分为定时刷盘和实时刷盘，一般默认的是实时刷盘，接下展示rocketmq的刷盘机制。

``` java
//是否定时方式刷盘，默认是实时刷盘real time
  boolean flushCommitLogTimed =
  CommitLog.this.defaultMessageStore.getMessageStoreConfig().isFlushCommitLogTimed();
 //CommitLog刷盘间隔时间 default 1s
  int interval =
  CommitLog.this.defaultMessageStore.getMessageStoreConfig() .getFlushIntervalCommitLog();
 //刷CommitLog，至少刷几个PAGE default 4
 int flushPhysicQueueLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig()
                            .getFlushCommitLogLeastPages();
 //刷CommitLog，彻底刷盘间隔时间  default 10s
 int flushPhysicQueueThoroughInterval =
            CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogThoroughInterval();
            …………
             进行刷盘
            CommitLog.this.mapedFileQueue.commit(flushPhysicQueueLeastPages);
            …………
接下来就和同步刷盘方式一致了，都是写入硬盘的操作了
```

## 删除文件机制

``` java
deleteCount = DefaultMessageStore.this.commitLog.deleteExpiredFile(fileReservedTime, deletePhysicFilesInterval,
        destroyMapedFileIntervalForcibly, cleanAtOnce);
public int deleteExpiredFile(
                     final long expiredTime, 
                     final int deleteFilesInterval, 
                     final long intervalForcibly, 
                     final boolean cleanImmediately
) {
    return this.mapedFileQueue.deleteExpiredFileByTime(expiredTime, deleteFilesInterval, intervalForcibly, cleanImmediately);
}
public int deleteExpiredFileByTime(
                                  final long expiredTime, 
                                  final int deleteFilesInterval, 
                                  final long intervalForcibly,
                                  final boolean cleanImmediately
) {
    Object[] mfs = this.copyMapedFiles(0);
    if (null == mfs)
        return 0;
    int mfsLength = mfs.length - 1;
    int deleteCount = 0;
    List files = new ArrayList();
    if (null != mfs) {
        for (int i = 0; i < mfsLength; i++) {
            MapedFile mapedFile = (MapedFile) mfs[i];
            long liveMaxTimestamp = mapedFile.getLastModifiedTimestamp() + expiredTime;
            if (System.currentTimeMillis() >= liveMaxTimestamp
                    || cleanImmediately) {
                if (mapedFile.destroy(intervalForcibly)) {
                    files.add(mapedFile);
                    deleteCount++;
                    if (files.size() >= DeleteFilesBatchMax) {
                        break;
                    }
                    if (deleteFilesInterval > 0 && (i + 1) < mfsLength) {
                        try {
                            Thread.sleep(deleteFilesInterval);
                        } catch (InterruptedException e) {
                        }
                    }
                } else {
                    break;
                }
            }
        }
    }
    deleteExpiredFile(files);
    return deleteCount;
}
```
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)