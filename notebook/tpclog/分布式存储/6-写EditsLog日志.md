# **透彻理解分布式存储（六）——写Edits Log日志**

Edits Log日志，就是NameNode用来记录文件操作命令的日志。NameNode每次对**内存文件目录树**进行增删改时，都必须将操作记录到edits log日志中。这样即使NameNode宕机了，重启NameNode后也可以读取日志进行回放，恢复出一份完整的元数据。

## 一、写日志

NameNode在持久化Edits Log日志时，如果直接写磁盘，那么性能必然很差，所以一般都会 **先写缓存，然后异步刷盘** 。但是，因为Edits Log会先在内存中缓存，所以如果有些Log还没刷入磁盘，此刻NameNode就宕机了，会导致内存中的部分数据丢失。当然，你也可以使用同步刷盘模式，不过性能会大幅下降。很多开源框架，比如Elasticsearch、Redis、Kafka等，在做持久化时，都会提供异步/同步刷盘策略，允许用户在性能和可用性之间做权衡。

所以，我们的分布式文件系统在做数据持久化时，设计了以下几个方面的机制：

* **edits log双缓冲机制：** 每次只往一块缓冲区写记录，写满后就由一个线程执行刷盘操作；
* **edits log日志结构：** edits log日志结构应该是一种紧凑的二进制格式，减少磁盘I/O的数据量；
* **checkpoint机制：** edits log日志文件不能无限写，我们需要在一些checkpoint时间点生成内存快照，并将checkpoint之前的日志清除掉。

> checkpoint机制由BackupNode来实现，本章我们专注于写日志的逻辑和双缓冲机制。

### 1.1 日志结构

我们先来考虑下Edits Log的日志结构。也就是说，Edits Log的格式应该设计成什么样？如何序列化后保存到磁盘文件上？

对于我们这种小的分布式文件系统， 每天有10万次的文件增删改操作就已经很不错了，也就是说每天的Edits Log也就10万条左右，假设我们每隔1小时执行一次checkpoint，那么最多也就保留最近1小时的edits log，也就是1万条记录（假设一天的操作平均分布在10个小时内）。每条Edits Log可能就几十个字节，所以一个Edits Log日志文件一般也就几百kb。

综上，我们直接基于JSON格式来存储Edits Log日志就可以了，比如：

```json
{"OP": "MKDIR", "PATH": "/usr/warehouse/hive"}
{"OP": "RM", "PATH": "/usr/warehouse/hive"}
{"OP": "CREATE", "PATH": "/usr/warehouse/hive/access.log"}
```

我们定义Edits Log的Java Bean如下，每条操作日志都有唯一的txid：

```java
/**
 * 代表了一条edits log
 */
public class EditLog {
    // log id，唯一
    private long txid;
    // log内容，我们这里直接用json格式
    private String content;
    public EditLog(long txid, String content) {
        this.txid = txid;
        JSONObject jsonObject = JSONObject.parseObject(content);    
	jsonObject.put("txid", txid);
        this.content = jsonObject.toJSONString();
    }    public long getTxid() {
        return txid;
    }    public void setTxid(long txid) {
        this.txid = txid;
    }    public String getContent() {
        return content;
    }    public void setContent(String content) {
        this.content = content;
    }    @Override
    public String toString() {
        return "EditLog [txid=" + txid + ", content=" + content + "]";
    }
}
```

> 开源的分布式中间件一般都会采用紧凑的二进制格式存储文件，比如Kafka就自定义了消息格式，用最少的Bit位去存储尽可能多的信息，节约磁盘I/O开销。

### 1.2 双缓冲区

回顾一下[《整体架构》](https://www.tpvlog.com/article/319)一章，我提到过Edits Log双缓冲区：

![](https://files.tpvlog.com/tpvlog/distributed/storage/20210714225850063.png)
Edits Log双缓冲区的本质就是两块ByteBuffer：一块用于写入操作日志，写满之后就与另一块交换，然后刷磁盘。我们在设计时，可以定义一个**DoubleBuffer**组件，里面封装两块Buffer：

* `curBuffer`：表示正在写入log的缓冲区；
* `syncBuffer`：表示正在刷盘的缓冲区。

然后，DoubleBuffer需要对外提供一系列方便使用的接口，包括：

* 缓冲区是否已满（只要curBuffer满了就算满）；
* 交换缓冲区；
* 刷磁盘；
* 写入操作日志。

根据上述思路，我们实现一下Edits Log双缓冲区的代码：

```java
/**
 * Edits Log双缓冲区
 */
public class DoubleBuffer {
    // 单块缓冲区大小：默认512kb
    private static final Integer EDIT_LOG_BUFFER_LIMIT = 512 * 1024;
    // 当前正在写入log的缓存区
    private EditLogBuffer curBuffer = new EditLogBuffer();
    // 当前正在刷盘的缓存区
    private EditLogBuffer syncBuffer = new EditLogBuffer();
    // 当前缓冲区对应的日志文件的第一条Log txid
    private Long startTxid = 1L;
    // 已经刷入磁盘的txid范围
    private List<String> flushedTxids = new CopyOnWriteArrayList<>();
    public void write(EditLog log) throws Exception {
        curBuffer.write(log);    }    /**
     * 判断当前是否需要刷盘
     * <p>
     * 如果写入缓冲区满了，就需要刷盘
     */
    public boolean shouldSyncToDisk() {
        return curBuffer.size() >= EDIT_LOG_BUFFER_LIMIT;
    }    /**
     * 交换缓冲区
     */
    public void swapBuffer() {
        EditLogBuffer tmp = curBuffer;  
	curBuffer = syncBuffer;  
	syncBuffer = tmp;  
    }  
   /**
     * 将syncBuffer缓冲区中的数据刷入磁盘
     */
    public void flush() throws Exception {
        syncBuffer.flush();        syncBuffer.clear();    }    /**
     * 获取已经刷入磁盘的txid索引
     * @return
     */
    public List<String> getFlushedTxids() {
        return flushedTxids;
    }    /**
     * 获取当前缓冲区里的数据
     */
    public String[] getBufferedEditsLog() {
        if (curBuffer.size() == 0) {
            return null;
        }        String editsLogRawData = new String(curBuffer.getBufferData());
        return editsLogRawData.split("\n");
    }}
```

### 1.3 分段日志

Edits Log在磁盘上是分段存储的，每当一块缓冲区写满以后，就要进行刷盘，刷盘时会创建一个新的分段日志文件。我们来看下单块缓冲区**EditLogBuffer**的实现，它直接定义在 `DoubleBuffer`类中：

```java

// DoubleBuffer.java
/**
 * 单块edits log缓冲区
 */
class EditLogBuffer {
    ByteArrayOutputStream buffer;    // 最新写入的log日志的txid
    long endTxid = 0L;
    public EditLogBuffer() {
        this.buffer = new ByteArrayOutputStream(EDIT_LOG_BUFFER_LIMIT);
    }    /**
     * 将editslog日志写入缓冲区
     */
    public void write(EditLog log) throws Exception {
        endTxid = log.getTxid();        buffer.write(log.getContent().getBytes());        buffer.write("\n".getBytes());
        System.out.println("写入一条editslog：" + log.getContent() + "，当前缓冲区的大小是：" + size());
    }    public Integer size() {
        return buffer.size();
    }    /**
     * 将当前缓存区的数据刷入磁盘
     */
    public void flush() throws Exception {
        byte[] data = buffer.toByteArray();
        ByteBuffer dataBuffer = ByteBuffer.wrap(data);        // 这里可配置化，我直接写本地
        String editsLogFilePath ="C:\\Users\\Ressmix\\Desktop\\editslog\\edits-" + startTxid + "-" + endTxid + ".log";
        flushedTxids.add(startTxid + "_" + endTxid);
        RandomAccessFile file = null;
        FileOutputStream out = null;
        FileChannel editsLogFileChannel = null;
        try {
            file = new RandomAccessFile(editsLogFilePath, "rw"); // 读写模式，数据写入缓冲区中
            out = new FileOutputStream(file.getFD());
            editsLogFileChannel = out.getChannel();            editsLogFileChannel.write(dataBuffer);            // 强制刷盘
            editsLogFileChannel.force(false);
        } finally {
            if (out != null) {
                out.close();            }            if (file != null) {
                file.close();            }            if (editsLogFileChannel != null) {
                editsLogFileChannel.close();            }        }        // 新的日志文件的第一条log txid
        startTxid = endTxid + 1;
    }    /**
     * 清空内存缓冲区中的数据
     */
    public void clear() {
        buffer.reset();    }    /**
     * 获取内存缓冲区中的数据
     */
    public byte[] getBufferData() {
        return buffer.toByteArray();
    }
}
```

上述刷盘的代码，需要注意两点：

1. 我利用了Java NIO中的FileChannel；
2. 每执行一次刷盘，就生成一个新的edits log磁盘文件，我这里用 `startTxid`和 `endTxid`来实现日志分段的功能。startTxid表示当前buffer第一条写入的log txid，endTxid表示当前buffer最后一条（最新）写入的log txid。

## 二、日志管理

实现完Edits Log的双缓冲区和分段日志后，我们需要一个组件来进行统一的edits log日志管理，包括并发写入控制和刷盘控制等等，这个组件就是 **FSEditlog** 。

还记得我在[《文件目录树》](https://www.tpvlog.com/article/323)一章中提到的NameNode元数据管理组件——**FSNameSystem**吗？FSDirectory内部除了委托FSDirectory维护内存文件目录树以外，还会委托FSEditlog管理edits log日志：

![](https://files.tpvlog.com/tpvlog/distributed/storage/20210714225903179.png)

### 2.1 FSEditlog

我们来看下FSEditlog组件的实现逻辑，先看几个edits log日志ID相关的字段：

* **txidSeq：** FSEditlog需要给每一条操作日志分配一个全局唯一的日志ID；
* **syncTxid：** FSEditlog需要记录已经刷到磁盘上的最大操作日志ID，只有大于该id的日志才能往磁盘写；
* **localTxid：** 记录当前写日志的线程携带的日志id，方便在程序的各处使用。

FSEditlog写日志的核心流程在 `logEdit`方法，整个过程涉及到两把锁，对应两个重要的标志位：

* **isBufferSwapping：** 表示是否有线程正在进行Edits Log双缓冲区交换，当一个Buffer写满后，会由一个线程进行Buffer交换，在此期间其它线程都得阻塞，交换完成后其它线程就可以往新Buffer写数据了；
* **isSyncRunning：** 表示是否有线程正在进行刷盘操作，Buffer交换完成后，那个负责交换Buffer的线程需要进行刷盘，但此时可能另一块Buffer也很快被写满了（如果并发太高或Buffer大小设置不合理的话就可能出现这种情况），所以必须防止其它线程也同时刷盘。

```java
/**
 * 负责管理edits log日志的核心组件
 */
public class FSEditlog {

    // 全局Log日志ID，递增唯一
    private volatile long txidSeq = 0L;

    // 已经同步到磁盘的最新日志的最大txid
    private volatile Long syncTxid = 0L;

    // 每个写操作日志的线程，本地都有一份日志对应的txid副本
    private ThreadLocal<Long> localTxid = new ThreadLocal<Long>();

    // Edits Log双缓冲区
    private DoubleBuffer doubleBuffer = new DoubleBuffer();

    // 是否有线程正在刷盘
    private volatile Boolean isSyncRunning = false;

    // 是否有一块缓冲区已满，有线程正在执行交换
    private volatile Boolean isBufferSwapping = false;

    /**
     * 记录edits log日志
     */
    public void logEdit(String content) throws Exception {
        synchronized (this) {
            // 1.如果一块缓冲区满了，有线程正在交换Buffer，则阻塞等待
            waitSwapBuffer();

            // 2.为当前日志生成全局唯一递增的txid
            long txid = ++txidSeq;
            localTxid.set(txid);

            // 3.构造一条edits log操作日志
            EditLog log = new EditLog(txid, content);

            // 4.将log写入双缓冲区
            doubleBuffer.write(log);

            // 5.判断缓冲区是否满了，满了则需要刷盘
            if (doubleBuffer.shouldSyncToDisk()) {
                // 置准备交换缓冲区的标志
                isBufferSwapping = true;
            } else {
                return;
            }
        }

        // 6.交换缓冲区并刷盘
        swapAndSync();
    }

    /**
     * 交换缓冲区并刷盘
     */
    private void swapAndSync() {
        // 必须要加锁，防止多个线程同时刷盘
        synchronized (this) {
            // 1.获取当前正在写的log日志的id
            long txid = localTxid.get();

            // 2.1 有线程正在刷盘
            if (isSyncRunning) {
                // 正常txid应该大于磁盘上的最新log的txid
                if (txid <= syncTxid) {
                    return;
                }

                // 等待其它线程刷盘完成
                try {
                    while (isSyncRunning) {
                        wait(1000);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

            // 3.交换两块缓冲区
            doubleBuffer.swapBuffer();
            isBufferSwapping = false;

            // 4.记录最新磁盘日志id，为当前准备刷盘线程的日志id
            syncTxid = txid;
            isSyncRunning = true;

            // 5.唤醒其它等待刷盘的线程
            notifyAll();
        }

        // 6.执行刷盘
        try {
            doubleBuffer.flush();
        } catch (Exception e) {
            e.printStackTrace();
        }

        synchronized (this) {
            // 刷盘完成后，通知其它等待刷盘完成的线程
            isSyncRunning = false;
            notifyAll();
        }
    }

    /**
     * 等待缓冲区交换完成
     */
    private void waitSwapBuffer() {
        try {
            while (isBufferSwapping) {
                wait(1000);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 强制把内存缓冲里的数据刷入磁盘中
     */
    public void flush() {
        try {
            doubleBuffer.swapBuffer();
            doubleBuffer.flush();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取已经刷入磁盘的edits log索引
     */
    public List<String> getFlushedTxids() {
        synchronized (this) {
            return doubleBuffer.getFlushedTxids();
        }
    }

    /**
     * 获取当前缓冲区里的数据
     */
    public String[] getBufferedEditsLog() {
        synchronized (this) {
            return doubleBuffer.getBufferedEditsLog();
        }
    }
}
```

上述整个加锁流程是有点绕，读者可以自己在纸上画图帮助理解，涉及多线程比较难通过文字表达出来，我这里就不赘述了。

### 2.2 停机刷盘

Edits Log的数据在刷盘前是保存在内存中的，如果此时你要重启NameNode的话，就需要提供*优雅停机*接口，必须要把Edits Log缓冲区中的数据刷入磁盘。同时，在关闭期间不再接受客户端发送过来的操作命令。

我们可以提供一个名为 `shutdown`的RPC接口，实现优雅停机功能，通过一个字段 `isRunning`标识是否已停机：

```java
// NameNodeServiceImpl.java

public class NameNodeServiceImpl extends NameNodeServiceGrpc.NameNodeServiceImplBase {
    // 停机状态响应码
    public static final Integer STATUS_SHUTDOWN = 3;

    // 运行标识
    private volatile Boolean isRunning = true;

    private FSNameSystem namesystem;

    /**
     * 创建目录
     */
    @Override
    public void mkdir(MkDirRequest request, StreamObserver<MkDirResponse> responseObserver) {
        try {
            MkDirResponse response = null;
            // 如果已停机，则响应错误码
            if (!isRunning) {
                response = MkDirResponse.newBuilder().setStatus(STATUS_SHUTDOWN).build();
            } else {
                this.namesystem.mkdir(request.getPath());
                response = MkDirResponse.newBuilder().setStatus(STATUS_SUCCESS).build();
            }

            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 优雅停机
     */
    @Override
    public void shutdown(ShutdownRequest request, StreamObserver<ShutdownResponse> responseObserver) {
        isRunning = false;
        namesystem.flush();
        namesystem.saveCheckpointTxid();
        System.out.println("优雅关闭namenode......");
    }
}
```

需要注意上面的 `namesystem.saveCheckpointTxid()`，每次BackupNode生成完fsimage快照后，会发送RPC请求告知NameNode最新的checkpoint，NameNode会记录它，以便宕机恢复，所以NameNode在关闭时也要将它持久化到磁盘上，我在下一章会详细讲解。

## 三、总结

本章，我对NameNode进行Edits Log操作日志的持久化流程进行了详细讲解，并给出了代码实现。我们需要重点理解Edits Log的双缓冲区机制和分段日志机制。下一章，我将讲解Edits Log的日志复制机制。
