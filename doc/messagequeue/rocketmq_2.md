本文讲述 RocketMQ 存储一条消息的流程。

## 一、存储位置

当有一条消息过来之后，Broker 首先需要做的是确定这条消息应该存储在哪个文件里面。在 RocketMQ 中，这个用来存储消息的文件被称之为 MappedFile。这个文件默认创建的大小为 1GB。

![消息文件大小](/images/messagequeue/rocketmq-message-store-flow/mappedfile_size.png)

一个文件为 1GB 大小，也即 `1024 * 1024 * 1024 = 1073741824` 字节，这每个文件的命名是按照总的字节偏移量来命名的。例如第一个文件偏移量为 0，那么它的名字为 `00000000000000000000`；当当前这 1G 文件被存储满了之后，就会创建下一个文件，下一个文件的偏移量则为 1GB，那么它的名字为 `00000000001073741824`，以此类推。

![文件命名规则](/images/messagequeue/rocketmq-message-store-flow/mappedfile_offset_naming.png)

默认情况下这些消息文件位于 `$HOME/store/commitlog` 目录下，如下图所示:

![消息目录](/images/messagequeue/rocketmq-message-store-flow/2018_02_13_14_16_11.png)

## 二、文件创建

当 Broker 启动的时候，其会将位于存储目录下的所有消息文件加载到一个列表中:

![消息文件列表](/images/messagequeue/rocketmq-message-store-flow/mappedFiles_list.png)

当有新的消息到来的时候，其会默认选择列表中的最后一个文件来进行消息的保存:

![最后一个消息文件](/images/messagequeue/rocketmq-message-store-flow/last_mapped_file.png)

```java
public class MappedFileQueue {

    public MappedFile getLastMappedFile() {
        MappedFile mappedFileLast = null;

        while (!this.mappedFiles.isEmpty()) {
            try {
                mappedFileLast = this.mappedFiles.get(this.mappedFiles.size() - 1);
                break;
            } catch (IndexOutOfBoundsException e) {
                //continue;
            } catch (Exception e) {
                log.error("getLastMappedFile has exception.", e);
                break;
            }
        }

        return mappedFileLast;
    }
    
}
```

当然如果这个 Broker 之前从未接受过消息的话，那么这个列表肯定是空的。这样一旦有新的消息需要存储的时候，其就得需要立即创建一个 `MappedFile` 文件来存储消息。

RocketMQ 提供了一个专门用来实例化 `MappedFile` 文件的服务类 `AllocateMappedFileService`。在内存中，也同时维护了一张请求表 `requestTable` 和一个优先级请求队列 `requestQueue` 。当需要创建文件的时候，Broker 会创建一个 `AllocateRequest` 对象，其包含了文件的路径、大小等信息。然后先将其放入 `requestTable` 表中，再将其放入优先级请求队列 `requestQueue` 中:

```java
public class AllocateMappedFileService extends ServiceThread {

    public MappedFile putRequestAndReturnMappedFile(String nextFilePath,
                                                    String nextNextFilePath,
                                                    int fileSize) {

        // ...

        AllocateRequest nextReq = new AllocateRequest(nextFilePath, fileSize);
        boolean nextPutOK = this.requestTable.putIfAbsent(nextFilePath, nextReq) == null;
        if (nextPutOK) {
            // ...
            boolean offerOK = this.requestQueue.offer(nextReq);
        }
        
    }
    
}
```

服务类会**一直**等待优先级队列是否有新的请求到来，如果有，便会从队列中取出请求，然后创建对应的 `MappedFile`，并将请求表 requestTable 中 `AllocateRequest` 对象的字段 `mappedFile` 设置上值。最后将 `AllocateRequest` 对象上的 `CountDownLatch` 的计数器减 1 ，以标明此分配申请的 `MappedFile` 已经创建完毕了:

```java
public class AllocateMappedFileService extends ServiceThread {

    public void run() {
        // 一直运行
        while (!this.isStopped() && this.mmapOperation()) {
        }
    }

    private boolean mmapOperation() {
        req = this.requestQueue.take();

        if (req.getMappedFile() == null) {
            MappedFile mappedFile;
            // ...
            mappedFile = new MappedFile(req.getFilePath(), req.getFileSize());
            // 设置上值
            req.setMappedFile(mappedFile);
        }

        // ...
        // 计数器减 1
        req.getCountDownLatch().countDown();

        // ...
        return true;
    }
    
}
```

其上述整体流程如下所示:

![创建文件过程](/images/messagequeue/rocketmq-message-store-flow/AllocateMappedFileService.png)

等待 `MappedFile` 创建完毕之后，其便会从请求表 `requestTable` 中取出并删除表中记录:

```java
public class AllocateMappedFileService extends ServiceThread {

    public MappedFile putRequestAndReturnMappedFile(String nextFilePath,
                                                    String nextNextFilePath,
                                                    int fileSize) {
        // ...
        AllocateRequest result = this.requestTable.get(nextFilePath);
        if (result != null) {
            // 等待 MappedFile 的创建完成
            boolean waitOK = result.getCountDownLatch().await(waitTimeOut, TimeUnit.MILLISECONDS);
            if (!waitOK) {
                return null;
            } else {
                // 从请求表中删除
                this.requestTable.remove(nextFilePath);
                return result.getMappedFile();
            }
        }
    }
    
}
```

然后再将其放到列表中去:

```java
public class MappedFileQueue {

    public MappedFile getLastMappedFile(final long startOffset, boolean needCreate) {
        MappedFile mappedFile = null;

        if (this.allocateMappedFileService != null) {
            // 创建 MappedFile
            mappedFile = this.allocateMappedFileService
                .putRequestAndReturnMappedFile(nextFilePath,
                                               nextNextFilePath,
                                               this.mappedFileSize);
        }

        if (mappedFile != null) {
            // ...
            // 添加至列表中
            this.mappedFiles.add(mappedFile);
        }

        return mappedFile;

    }
    
}
```

![转移文件到列表](/images/messagequeue/rocketmq-message-store-flow/transfer_mappedfile_from_requesttable.png)

至此，`MappedFile` 已经创建完毕，也即可以进行下一步的操作了。

## 三、文件初始化

在 `MappedFile` 的构造函数中，其使用了 `FileChannel` 类提供的 map 函数来将磁盘上的这个文件映射到进程地址空间中。然后当通过 `MappedByteBuffer` 来读入或者写入文件的时候，磁盘上也会有相应的改动。采用这种方式，通常比传统的基于文件 IO 流的方式读取效率高。

```java
public class MappedFile extends ReferenceResource {
    
    public MappedFile(final String fileName, final int fileSize)
        throws IOException {
        init(fileName, fileSize);
    }

    private void init(final String fileName, final int fileSize)
        throws IOException {
        // ...
        this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
        this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
        // ...
    }
    
}
```

## 四、消息文件加载

前面提到过，Broker 在启动的时候，会加载磁盘上的文件到一个 `mappedFiles` 列表中。但是加载完毕后，其还会对这份列表中的消息文件进行**验证 (恢复)**，确保没有错误。

验证的基本想法是通过一一读取列表中的每一个文件，然后再一一读取每个文件中的每个消息，在读取的过程中，其会更新整体的消息写入的偏移量，如下图中的红色箭头 (我们假设最终读取的消息的总偏移量为 905):

![恢复文件](/images/messagequeue/rocketmq-message-store-flow/mappedfile_recover.png)

当确定消息整体的偏移量之后，Broker 便会确定每一个单独的 `MappedFile` 文件的**各自的偏移量**，每一个文件的偏移量是通过**取余**算法确定的:

```java
public class MappedFileQueue {

    public void truncateDirtyFiles(long offset) {

        for (MappedFile file : this.mappedFiles) {
            long fileTailOffset = file.getFileFromOffset() + this.mappedFileSize;
            if (fileTailOffset > offset) {
                if (offset >= file.getFileFromOffset()) {
                    // 确定每个文件的各自偏移量
                    file.setWrotePosition((int) (offset % this.mappedFileSize));
                    file.setCommittedPosition((int) (offset % this.mappedFileSize));
                    file.setFlushedPosition((int) (offset % this.mappedFileSize));
                } else {
                    // ...
                }
            }
        }

        // ...
    }
    
}
```

![更新文件位置](/images/messagequeue/rocketmq-message-store-flow/update_mappedfile_position.png)

在确定每个消息文件各自的写入位置的同时，其还会删除**起始偏移量**大于当前总偏移量的消息文件，这些文件可以视作脏文件，或者也可以说这些文件里面一条消息也没有。这也是上述文件 `1073741824` 被打上红叉的原因:

```java
public void truncateDirtyFiles(long offset) {
    List<MappedFile> willRemoveFiles = new ArrayList<MappedFile>();

    for (MappedFile file : this.mappedFiles) {
        long fileTailOffset = file.getFileFromOffset() + this.mappedFileSize;
        if (fileTailOffset > offset) {
            if (offset >= file.getFileFromOffset()) {
                // ...
            } else {
                // 总偏移量 < 文件起始偏移量
                // 加入到待删除列表中
                file.destroy(1000);
                willRemoveFiles.add(file);
            }
        }
    }

    this.deleteExpiredFile(willRemoveFiles);
}
```

## 五、写入消息

一旦我们获取到 `MappedFile` 文件之后，我们便可以往这个文件里面写入消息了。写入消息可能会遇见如下两种情况，一种是这条消息可以完全追加到这个文件中，另外一种是这条消息完全不能或者只有一小部分只能存放到这个文件中，其余的需要放到新的文件中。我们对于这两种情况分别讨论:

### (1) 文件可以完全存储消息

`MappedFile` 类维护了一个用以标识当前写位置的指针 `wrotePosition`，以及一个用来映射文件到进程地址空间的 `mappedByteBuffer`:

```java
public class MappedFile extends ReferenceResource {

    protected final AtomicInteger wrotePosition = new AtomicInteger(0);
    private MappedByteBuffer mappedByteBuffer;
    
}
```

由这两个数据结构我们可以看出来，单个文件的消息写入过程其实是非常简单的。首先获取到这个文件的写入位置，然后将消息内容追加到 `byteBuffer` 中，然后再更新写入位置。

```java
public class MappedFile extends ReferenceResource {

    public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
        // ...
    
        int currentPos = this.wrotePosition.get();

        if (currentPos < this.fileSize) {
            ByteBuffer byteBuffer =
                writeBuffer != null ?
                writeBuffer.slice() :
                this.mappedByteBuffer.slice();

            // 更新 byteBuffer 位置
            byteBuffer.position(currentPos);

            // 写入消息内容
            // ...

            // 更新 wrotePosition 指针的位置
            this.wrotePosition.addAndGet(result.getWroteBytes());

            return result;
        }

    }
    
}
```

示例流程如下所示:

![消息文件的位置指针](/images/messagequeue/rocketmq-message-store-flow/message_in_one_mappedfile.png)

### (2) 文件不可以完全存储消息

在写入消息之前，如果判断出文件已经满了的情况下，其会直接尝试创建一个新的 `MappedFile`:

```java
public class CommitLog {

    public PutMessageResult putMessage(final MessageExtBrokerInner msg) {

        // 文件为空 || 文件已经满了
        if (null == mappedFile || mappedFile.isFull()) {
            mappedFile = this.mappedFileQueue.getLastMappedFile(0);
        }

        // ...
        
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        
    }
    
}
```

如果文件未满，那么在写入之前会先计算出消息体长度 `msgLen`，然后判断这个文件剩下的空间是否有能力容纳这条消息。在这个地方我们还需要介绍下每条消息的存储方式。

每条消息的存储是按照一个 4 字节的长度来做界限的，这个长度本身就是整个消息体的长度，当读完这整条消息体的长度之后，下一次再取出来的一个 4 字节的数字，便又是下一条消息的长度:

![消息存储格式](/images/messagequeue/rocketmq-message-store-flow/length_based_message_limit.png)

围绕着一条消息，还会存储许多其它内容，我们在这里只需要了解前两位是 4 字节的长度和 4 字节的 MAGICCODE 即可:

![消息头](/images/messagequeue/rocketmq-message-store-flow/message_serialize_header.png)

`MAGICCODE` 的可选值有:

- `CommitLog.MESSAGE_MAGIC_CODE`
- `CommitLog.BLANK_MAGIC_CODE`

当这个文件有能力容纳这条消息体的情况下，其便会存储 `MESSAGE_MAGIC_CODE` 值；当这个文件没有能力容纳这条消息体的情况下，其便会存储 `BLANK_MAGIC_CODE` 值。所以这个 `MAGICCODE` 是用来界定这是空消息还是一条正常的消息。

当判定这个文件不足以容纳整个消息的时候，其将消息体长度设置为这个文件剩余的最大空间长度，将 `MAGICCODE` 设定为这是一个空消息文件 (需要去下一个文件去读)。由此我们可以看出消息体长度 和 `MAGICCODE` 是判别一条消息格式的最基本要求，这也是 `END_FILE_MIN_BLANK_LENGTH` 的值为 8 的原因:

```java
// CommitLog.java
class DefaultAppendMessageCallback implements AppendMessageCallback {

    // File at the end of the minimum fixed length empty
    private static final int END_FILE_MIN_BLANK_LENGTH = 4 + 4;
    
    public AppendMessageResult doAppend(final long fileFromOffset,
                                        final ByteBuffer byteBuffer,
                                        final int maxBlank,
                                        final MessageExtBrokerInner msgInner) {

        // ...
        
        if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
            // ...

            // 1 TOTALSIZE
            this.msgStoreItemMemory.putInt(maxBlank);
            // 2 MAGICCODE
            this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
            // 3 The remaining space may be any value
            byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);
            
            return new AppendMessageResult(AppendMessageStatus.END_OF_FILE,
                                           /** other params **/ );
        }
        
    }
    
}
```

由上述方法我们看出在这种情况下返回的结果是 `END_OF_FILE`。当检测到这种返回结果的时候，`CommitLog` 接着又会申请创建新的 `MappedFile` 并尝试写入消息。追加方法同 (1) 相同，不再赘述:

![文件尾部](/images/messagequeue/rocketmq-message-store-flow/file_tail_message.png)

> 注: 在消息文件加载的过程中，其也是通过判断 `MAGICCODE` 的类型，来判断是否继续读取下一个 `MappedFile` 来计算整体消息偏移量的。

## 六、消息刷盘策略

当消息体追加到 `MappedFile` 以后，这条消息实际上还只是存储在内存中，因此还需要将内存中的内容刷到磁盘上才算真正的存储下来，才能确保消息不丢失。一般而言，刷盘有两种策略: 异步刷盘和同步刷盘。

### (1) 异步刷盘

当配置为异步刷盘策略的时候，Broker 会运行一个服务 `FlushRealTimeService` 用来刷新缓冲区的消息内容到磁盘，这个服务使用一个独立的线程来做刷盘这件事情，默认情况下每隔 500ms 来检查一次是否需要刷盘:

```java
class FlushRealTimeService extends FlushCommitLogService {

    public void run() {

        // 不停运行
        while (!this.isStopped()) {

            // interval 默认值是 500ms
            if (flushCommitLogTimed) {
                Thread.sleep(interval);
            } else {
                this.waitForRunning(interval);
            }

            // 刷盘
            CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);

        }
        
    }
    
}
```

在追加消息完毕之后，通过**唤醒**这个服务立即检查以下是否需要刷盘:

```java
public class CommitLog {

    public void handleDiskFlush(AppendMessageResult result,
                                PutMessageResult putMessageResult,
                                MessageExt messageExt) {
        // Synchronization flush
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            // ...
        }
        // Asynchronous flush
        else {
            if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                // 消息追加成功后，立即唤醒服务
                flushCommitLogService.wakeup();
            } else {
                // ...
            }
        }
    }
    
}
```

### (2) 同步刷盘

当配置为同步刷盘策略的时候，Broker 运行一个叫做 `GroupCommitService` 服务。在这个服务内部维护了一个**写请求**队列和一个**读请求**队列，其中这两个队列每隔 10ms 就交换一下“身份”，这么做的目的其实也是为了**读写分离**:

![读队列和写队列交换](/images/messagequeue/rocketmq-message-store-flow/swap_list.png)

在这个服务内部，每隔 10ms 就会检查读请求队列是否不为空，如果不为空，则会将读队列中的所有请求执行刷盘，并清空读请求队列:

```java
class GroupCommitService extends FlushCommitLogService {

    private void doCommit() {
        // 检查所有读队列中的请求
        for (GroupCommitRequest req : this.requestsRead) {
            // 每个请求执行刷盘
            CommitLog.this.mappedFileQueue.flush(0);
            req.wakeupCustomer(flushOK);
        }

        this.requestsRead.clear();
    }
    
}
```

在追加消息完毕之后，通过创建一个请求刷盘的对象，然后通过 `putRequest()` 方法放入写请求队列中，这个时候会立即唤醒这个服务，写队列和读队列的角色会进行交换，交换角色之后，读请求队列就不为空，继而可以执行所有刷盘请求了。而在这期间，Broker 会一直阻塞等待最多 5 秒钟，在这期间如果完不成刷盘请求的话，那么视作刷盘超时:

```java
public class CommitLog {

    public void handleDiskFlush(AppendMessageResult result,
                                PutMessageResult putMessageResult,
                                MessageExt messageExt) {
        // Synchronization flush
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            // ...
            if (messageExt.isWaitStoreMsgOK()) {
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                service.putRequest(request);
                // 等待刷盘成功
                boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                if (!flushOK) {
                    // 刷盘超时
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
                }
            } else {
                // ...
            }
        }
        // Asynchronous flush
        else {
            // ...
        }
    }
    
}
```

通过方法 `putRequest` 放入请求后的服务执行流程:

![放入请求](/images/messagequeue/rocketmq-message-store-flow/putRequest.png)

## 七、消息刷盘理念

我们在这里已经知道消息刷盘有同步刷盘和异步刷盘策略，对应的是 `GroupCommitService` 和 `FlushRealTimeService` 这两种不同的服务。

这两种服务都有定时请求刷盘的机制，但是机制背后最终调用的刷盘方式全部都集中在 `flush` 这个方法上:

```java
public class MappedFileQueue {

    public boolean flush(final int flushLeastPages) {
        // ...
    }
    
}
```

再继续向下分析这个方法之前，我们先对照着这张图说明一下使用 `MappedByteBuffer` 来简要阐述读和写文件的简单过程：

![mmap](/images/messagequeue/rocketmq-message-store-flow/mmap_readInt_and_writeInt.png)

操作系统为了能够使多个进程同时使用内存，又保证各个进程访问内存互相独立，于是为每个进程引入了**地址空间**的概念，地址空间上的地址叫做**虚拟地址**，而程序想要运行必须放到**物理地址**上运行才可以。地址空间为进程**营造出了一种假象**：”整台计算机只有我一个程序在运行，这台计算机内存很大”。一个地址空间内包含着这个进程所需要的全部状态信息。通常一个进程的地址空间会按照逻辑分成好多**段**，比如**代码段、堆段、栈段**等。为了进一步有效利用内存，每一段又细分成了不同的**页 (page)**。与此相对应，计算机的物理内存被切成了**页帧 (page frame)**，文件被分成了**块 (block)**。既然程序实际运行的时候还是得依赖物理内存的地址，那么就需要将虚拟地址转换为物理地址，这个映射关系是由**页表 (page table)**来完成的。

另外在操作系统中，还有一层**磁盘缓存 (disk cache)\**的概念，它主要是用来减少对磁盘的 I/O 操作。磁盘缓存是以页为单位的，内容就是磁盘上的物理块，所以又称之为\**页缓存 (page cache)**。当进程发起一个读操作 （比如，进程发起一个 read() 系统调用），它首先会检查需要的数据是否在页缓存中。如果在，则放弃访问磁盘，而直接从页缓存中读取。如果数据没在缓存中，那么内核必须调度块 I/O 操作从磁盘去读取数据，然后将读来的数据放入页缓存中。系统并不一定要将整个文件都缓存，它可以只存储一个文件的一页或者几页。

如图所示，当调用 `FileChannel.map()` 方法的时候，会将这个文件**映射**进用户空间的地址空间中，注意，建立映射不会拷贝任何数据。我们前面提到过 Broker 启动的时候会有一个消息文件加载的过程，当第一次开始读取数据的时候:

```java
// 首次读取数据
int totalSize = byteBuffer.getInt();
```

这个时候，操作系统通过查询页表，会发现文件的这部分数据还不在内存中。于是就会触发一个缺页异常 (page faults)，这个时候操作系统会开始从磁盘读取这一页数据，然后先放入到页缓存中，然后再放入内存中。在第一次读取文件的时候，操作系统会读入所请求的页面，并读入紧随其后的少数几个页面（**不少于一个页面，通常是三个页面**），这时的预读称为**同步预读** (如下图所示，红色部分是需要读取的页面，蓝色的那三个框是操作系统预先读取的):

![同步预读](/images/messagequeue/rocketmq-message-store-flow/linux_filecache_readahead.png)

当然随着时间推移，预读命中的话，那么相应的预读页面数量也会增加，但是能够确认的是，一个文件至少有 4 个页面处在页缓存中。当文件一直处于**顺序读取**的情况下，那么基本上可以保证每次**预读命中**:

![预读命中](/images/messagequeue/rocketmq-message-store-flow/readahead_hit.png)

下面我们来说文件写，正常情况下，当尝试调用 `writeInt()` 写数据到文件里面的话，其写到页缓存层，这个方法就会返回了。这个时候数据还没有真正的保存到文件中去，Linux 仅仅将页缓存中的这一页数据标记为“**脏**”，并且被加入到脏页链表中。然后由一群进程（**flusher 回写进程**）周期性将脏页链表中的页写会到磁盘，从而让磁盘中的数据和内存中保持一致，最后清理“脏”标识。在以下三种情况下，脏页会被写回磁盘:

- 空闲内存低于一个特定阈值
- 脏页在内存中驻留超过一个特定的阈值时
- 当用户进程调用 `sync()` 和 `fsync()` 系统调用时

可见，在正常情况下，即使不采用刷盘策略，数据最终也是会被同步到磁盘中去的:

![页写入缓存](/images/messagequeue/rocketmq-message-store-flow/write_to_page_cache.png)

但是，即便有 `flusher` 线程来定时同步数据，如果此时机器断电的话，消息依然有可能丢失。RocketMQ 为了保证消息尽可能的不丢失，为了最大的高可靠性，做了同步和异步刷盘策略，来手动进行同步:

![强制刷新](/images/messagequeue/rocketmq-message-store-flow/force_sync_data_to_disk.png)

## 八、消息刷盘过程

在介绍完上述消息刷盘背后的一些机制和理念后，我们再来分析刷盘整个过程。首先，无论同步刷盘还是异步刷盘，其线程都在一直周期性的尝试执行刷盘，在真正执行刷盘函数的调用之前，Broker 会检查文件的写位置是否大于 `flush` 位置，避免执行无意义的刷盘：

![写入位置和刷新位置](/images/messagequeue/rocketmq-message-store-flow/wrote_position_vs_flushed_position.png)

其次，对于异步刷盘来讲，Broker 执行了更为严格的刷盘限制策略，当在某个时间点尝试执行刷盘之后，在接下来 10 秒内，如果想要继续刷盘，那么脏页面数量必须不小于 4 页，如下图所示:

![刷盘条件](/images/messagequeue/rocketmq-message-store-flow/FlushRealTimeService_flush_condition.png)

下面是执行刷盘前最后检查的刷盘条件：

```java
public class MappedFile extends ReferenceResource {

    private boolean isAbleToFlush(final int flushLeastPages) {
        int flush = this.flushedPosition.get();
        int write = getReadPosition();

        if (this.isFull()) {
            return true;
        }

        if (flushLeastPages > 0) {
            // 计算当前脏页面算法
            return ((write / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE)) >= flushLeastPages;
        }

        // wrotePosition > flushedPosition
        return write > flush;
    }
    
}
```

当刷盘完毕之后，首先会更新这个文件的 `flush` 位置，然后再更新 `MappedFileQueue` 的整体的 `flush` 位置:

![更新 flush 位置](/images/messagequeue/rocketmq-message-store-flow/update_flush_position.png)

当刷盘完毕之后，便会将结果通知给客户端，告知发送消息成功。至此，整个存储过程完毕。
