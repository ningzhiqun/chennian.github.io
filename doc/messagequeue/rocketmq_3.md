本篇讲述 RocketMQ 消息接受流程

## 一、消费者注册

**生产者**负责往服务器 Broker 发送消息，消费者则从 Broker 获取消息。消费者获取消息采用的是**订阅者模式**，即消费者客户端可以任意订阅一个或者多个话题来消费消息:

```java
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {
        /*
         * 订阅一个或者多个话题
         */
        consumer.subscribe("TopicTest", "*");
    }
}
```

当消费者客户端启动以后，其会每隔 30 秒从命名服务器查询一次**用户订阅的所有话题路由信息**:

```java
public class MQClientInstance {

    private void startScheduledTask() {
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    // 从命名服务器拉取话题信息
                    MQClientInstance.this.updateTopicRouteInfoFromNameServer();
                }
            }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);
    }
    
}
```

我们由 [RocketMQ 消息发送流程](https://kunzhao.org/docs/rocketmq/rocketmq-send-message-flow/) 这篇文章知道 RocketMQ 在发送消息的时候，每条消息会以轮循的方式均衡地分发的不同 Broker 的不同队列中去。由此，消费者客户端从服务器命名服务器获取下来的便是话题的所有消息队列:

![所有话题队列](/images/messagequeue/rocketmq-message-receive-flow/topic_all_message_queue.png)

在获取话题路由信息的时候，客户端还会将话题路由信息中的**所有 Broker 地址**保存到本地:

```java
public class MQClientInstance {

    public boolean updateTopicRouteInfoFromNameServer(final String topic,
                                                      boolean isDefault,
                                                      DefaultMQProducer defaultMQProducer) {

        // ...
        
        if (changed) {
            TopicRouteData cloneTopicRouteData = topicRouteData.cloneTopicRouteData();

            // 更新 Broker 地址列表
            for (BrokerData bd : topicRouteData.getBrokerDatas()) {
                this.brokerAddrTable.put(bd.getBrokerName(), bd.getBrokerAddrs());
            }

            return true;
        }

        // ...
    }
    
}
```

当消费者客户端获取到了 Broker 地址列表之后，其便会每隔 30 秒给服务器发送一条**心跳数据包**，告知所有 Broker 服务器这台消费者客户端的存在。在每次发送心跳包的同时，其数据包内还会捎带这个客户端消息订阅的一些组信息，比如用户订阅了哪几个话题等，与此相对应，每台 Broker 服务器会在内存中维护一份当前所有的**消费者客户端列表信息**:

```java
public class ConsumerManager {
 
    private final ConcurrentMap<String/* Group */, ConsumerGroupInfo> consumerTable =
        new ConcurrentHashMap<String, ConsumerGroupInfo>(1024);
    
}
```

消费者客户端与 Broker 服务器进行沟通的整体流程如下图所示：

![消费者与 Broker 沟通](/images/messagequeue/rocketmq-message-receive-flow/broker_consumer_info.png)

## 二、消息队列负载均衡

我们知道无论发送消息还是接受消息都需要指定消息的话题，然而实际上消息在 Broker 服务器上并不是以话题为单位进行存储的，而是采用了比话题更细粒度的队列来进行存储的。当你发送了 10 条相同话题的消息，这 10 条话题可能存储在了不同 Broker 服务器的不同队列中。由此，我们说 RocketMQ 管理消息的单位不是**话题**，而是**队列**。

当我们讨论消息队列负载均衡的时候，就是在讨论服务器端的所有队列如何给所有消费者消费的问题。在 RocketMQ 中，客户端有两种消费模式，一种是**广播模式**，另外一种是**集群模式**。

我们现在假设总共有两台 Broker 服务器，假设用户使用 Producer 已经发送了 8 条消息，这 8 条消息现在均衡的分布在两台 Broker 服务器的 8 个队列中，每个队列中有一个消息。现在有 3 台都订阅了 Test 话题的消费者实例，我们来看在不同消费模式下，不同的消费者会收到哪几条消息。

### (1) 广播模式

广播模式是指所有消息队列中的消息都会广播给所有的消费者客户端，如下图所示，每一个消费者都能收到这 8 条消息:

![广播模式](/images/messagequeue/rocketmq-message-receive-flow/broadcasting_mode.png)

### (2) 集群模式

集群模式是指所有的消息队列会按照某种分配策略来分给不同的消费者客户端，比如消费者 A 消费前 3 个队列中的消息，消费者 B 消费中间 3 个队列中的消息等等。我们现在着重看 RocketMQ 为我们提供的三个比较重要的消息队列分配策略:

#### 1. 平均分配策略

平均分配策略下，三个消费者的消费情况如下所示：

- Consumer-1 消费前 3 个消息队列中的消息
- Consumer-2 消费中间 3 个消息队列中的消息
- Consumer-3 消费最后 2 个消息队列中的消息

![平均分配策略](/images/messagequeue/rocketmq-message-receive-flow/allocate_message_queue_strategy_average.png)

#### 2. 平均分配轮循策略

平均分配轮循策略下，三个消费者的消费情况如下所示：

- Consumer-1 消费 1、4、7消息队列中的消息
- Consumer-2 消费 2、5、8消息队列中的消息
- Consumer-3 消费 3、6消息队列中的消息

![平均分配轮循](/images/messagequeue/rocketmq-message-receive-flow/allocate_messagequeue_averagely_by_circle.png)

#### 3. 一致性哈希策略

一致性哈希算法是根据这三台消费者各自的某个有代表性的属性(我们假设就是客户端ID)来计算出三个 Hash 值，此处为了减少由于 Hash 函数选取的不理想的情况， RocketMQ 算法对于每个消费者通过在客户端ID后面添加 1、2、3 索引来使每一个消费者多生成几个哈希值。那么现在我们需要哈希的就是九个字符串:

- Consumer-1-1
- Consumer-1-2
- Consumer-1-3
- Consumer-2-1
- Consumer-2-2
- Consumer-2-3
- Consumer-3-1
- Consumer-3-2
- Consumer-3-3

计算完这 9 个哈希值以后，我们按照从小到大的顺序来排列成一个环 (如图所示)。这个时候我们需要一一对这 8 个消息队列也要计算一下 Hash 值，当 Hash 值落在两个圈之间的时候，我们就选取沿着环的方向的那个节点作为这个消息队列的消费者。如下图所示 (注意: 图只是示例，并非真正的消费情况):

在一致性哈希策略下，三个消费者的消费情况如下所示：

- Consumer-1 消费 1、2、3、4消息队列中的消息
- Consumer-2 消费 5、8消息队列中的消息
- Consumer-3 消费 6、7消息队列中的消息

![一致性哈希](/images/messagequeue/rocketmq-message-receive-flow/allocate_message_queue_consistent_hash.png)

消息队列的负载均衡是由一个不停运行的**均衡服务**来定时执行的:

```java
public class RebalanceService extends ServiceThread {
    // 默认 20 秒一次
    private static long waitInterval =
        Long.parseLong(System.getProperty("rocketmq.client.rebalance.waitInterval", "20000"));

    @Override
    public void run() {
        while (!this.isStopped()) {
            this.waitForRunning(waitInterval);
            // 重新执行消息队列的负载均衡
            this.mqClientFactory.doRebalance();
        }
    }

}
```

接着往下看，会知道在广播模式下，当前这台消费者消费和话题相关的所有消息队列，而集群模式会先按照某种分配策略来进行消息队列的分配，得到的结果就是当前这台消费者需要消费的消息队列:

```java
public abstract class RebalanceImpl {

    private void rebalanceByTopic(final String topic, final boolean isOrder) {
        switch (messageModel) {
            // 广播模式
        case BROADCASTING: {
            // 消费这个话题的所有消息队列
            Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
            if (mqSet != null) {
                // ...
            }
            break;
        }
            // 集群模式
        case CLUSTERING: {
            // ...

            // 按照某种负载均衡策略进行消息队列和消费客户端之间的分配
            // allocateResult 就是当前这台消费者被分配到的消息队列
            allocateResult = strategy.allocate(
                                               this.consumerGroup,
                                               this.mQClientFactory.getClientId(),
                                               mqAll,
                                               cidAll);

            // ...
            }
            break;
        }

    }
    
}
```

## 三、Broker 消费队列文件

现在我们再来看 Broker 服务器端。首先我们应该知道，消息往 Broker 存储就是在向 `CommitLog` 消息文件中写入数据的一个过程。在 Broker 启动过程中，其会启动一个叫做 `ReputMessageService` 的服务，这个服务每隔 1 秒会检查一下这个 `CommitLog` 是否有新的数据写入。`ReputMessageService` 自身维护了一个偏移量 `reputFromOffset`，用以对比和 `CommitLog` 文件中的消息总偏移量的差距。当这两个偏移量不同的时候，就代表有新的消息到来了:

```java
class ReputMessageService extends ServiceThread {

    private volatile long reputFromOffset = 0;

    private boolean isCommitLogAvailable() {
        // 看当前有没有新的消息到来
        return this.reputFromOffset < DefaultMessageStore.this.commitLog.getMaxOffset();
    }

    @Override
    public void run() {
        while (!this.isStopped()) {
            try {
                Thread.sleep(1);
                this.doReput();
            } catch (Exception e) {
                DefaultMessageStore.log.warn(this.getServiceName() + " service has exception. ", e);
            }
        }
    }
    
}
```

在有新的消息到来之后，`doReput()` 函数会取出新到来的所有消息，每一条消息都会封装为一个 `DispatchRequest` 请求，进而将这条请求分发给不同的请求消费者，我们在这篇文章中只会关注利用消息创建消费队列的服务 `CommitLogDispatcherBuildConsumeQueue`:

```java
class ReputMessageService extends ServiceThread {

    // ... 部分代码有删减
    private void doReput() {
        SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
        if (result != null) {
            this.reputFromOffset = result.getStartOffset();

            for (int readSize = 0; readSize < result.getSize() && doNext; ) {
                // 读取一条消息，然后封装为 DispatchRequest
                DispatchRequest dispatchRequest =
                    DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
                int size = dispatchRequest.getMsgSize();

                if (dispatchRequest.isSuccess()) {
                    // 分发这个 DispatchRequest 请求
                    DefaultMessageStore.this.doDispatch(dispatchRequest);
                    this.reputFromOffset += size;
                    readSize += size;
                }

                // ...
            }
        }
    }

}
```

`CommitLogDispatcherBuildConsumeQueue` 服务会根据这条请求按照不同的队列 ID 创建不同的消费队列文件，并在内存中维护一份消费队列列表。然后将 `DispatchRequest` 请求中这条消息的消息偏移量、消息大小以及消息在发送时候附带的标签的 Hash 值写入到相应的消费队列文件中去。

消费队列文件的创建与[消息存储 CommitLog 文件的创建过程](https://kunzhao.org/docs/rocketmq/rocketmq-message-store-flow/#二文件创建)是一致的，只是路径不同，这里不再赘述。

寻找消费队列的代码如下:

```java
public class DefaultMessageStore implements MessageStore {    private final ConcurrentMap<String/* topic */, ConcurrentMap<Integer/* queueId */, ConsumeQueue>> consumeQueueTable;        public void putMessagePositionInfo(DispatchRequest dispatchRequest) {        ConsumeQueue cq = this.findConsumeQueue(dispatchRequest.getTopic(), dispatchRequest.getQueueId());        cq.putMessagePositionInfoWrapper(dispatchRequest);    }    }
```

向消费队列文件中存储数据的代码如下:

```java
public class ConsumeQueue {    private boolean putMessagePositionInfo(final long offset, final int size, final long tagsCode,                                           final long cqOffset) {        // 存储偏移量、大小、标签码        this.byteBufferIndex.flip();        this.byteBufferIndex.limit(CQ_STORE_UNIT_SIZE);        this.byteBufferIndex.putLong(offset);        this.byteBufferIndex.putInt(size);        this.byteBufferIndex.putLong(tagsCode);        // 获取消费队列文件        final long expectLogicOffset = cqOffset * CQ_STORE_UNIT_SIZE;        MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile(expectLogicOffset);                if (mappedFile != null) {            // ...            return mappedFile.appendMessage(this.byteBufferIndex.array());        }        return false;    }    }
```

以上阐述了消费队列创建并存储消息的一个过程，但是消费队列文件中的消息是需要持久化到磁盘中去的。持久化的过程是通过后台服务 `FlushConsumeQueueService` 来定时持久化的:

```java
class FlushConsumeQueueService extends ServiceThread {    private void doFlush(int retryTimes) {        // ...        ConcurrentMap<String, ConcurrentMap<Integer, ConsumeQueue>> tables = DefaultMessageStore.this.consumeQueueTable;        for (ConcurrentMap<Integer, ConsumeQueue> maps : tables.values()) {            for (ConsumeQueue cq : maps.values()) {                boolean result = false;                for (int i = 0; i < retryTimes && !result; i++) {                    // 刷新到磁盘                    result = cq.flush(flushConsumeQueueLeastPages);                }            }        }        // ...    }}
```

上述过程体现在磁盘文件的变化如下图所示，`commitLog` 文件夹下面存放的是完整的消息，来一条消息，向文件中追加一条消息。同时，根据这一条消息属于 `TopicTest` 话题下的哪一个队列，又会往相应的 `consumequeue` 文件下的相应消费队列文件中追加消息的偏移量、消息大小和标签码:

![文件存储方式](/images/messagequeue/rocketmq-message-receive-flow/2018_03_16_11_54_01.png)

总流程图如下所示:

![消费队列](/images/messagequeue/rocketmq-message-receive-flow/broker_consume_queue.png)

## 四、消息队列偏移量

Broker 服务器存储了各个消费队列，客户端需要消费每个消费队列中的消息。消费模式的不同，每个客户端所消费的消息队列也不同。那么客户端如何记录自己所消费的队列消费到哪里了呢？答案就是**消费队列偏移量**。

针对同一话题，在**集群模式**下，由于每个客户端所消费的消息队列不同，所以每个消息队列已经消费到哪里的消费偏移量是**记录在 Broker 服务器端**的。而在广播模式下，由于每个客户端分配消费这个话题的所有消息队列，所以每个消息队列已经消费到哪里的消费偏移量是记录在**客户端本地**的。

下面分别讲述两种模式下偏移量是如何获取和更新的:

### (1) 集群模式

在集群模式下，消费者客户端在内存中维护了一个 `offsetTable` 表:

```java
public class RemoteBrokerOffsetStore implements OffsetStore {    private ConcurrentMap<MessageQueue, AtomicLong> offsetTable =        new ConcurrentHashMap<MessageQueue, AtomicLong>();    }
```

同样在 Broker 服务器端也维护了一个偏移量表:

```java
public class ConsumerOffsetManager extends ConfigManager {    private ConcurrentMap<String/* topic@group */, ConcurrentMap<Integer, Long>> offsetTable =        new ConcurrentHashMap<String, ConcurrentMap<Integer, Long>>(512);    }
```

在消费者客户端，`RebalanceService` 服务会定时地 (默认 20 秒) 从 Broker 服务器获取当前客户端所需要消费的消息队列，并与当前消费者客户端的消费队列进行对比，看是否有变化。对于每个消费队列，会从 Broker 服务器查询这个队列当前的消费偏移量。然后根据这几个消费队列，创建对应的拉取请求 `PullRequest` 准备从 Broker 服务器拉取消息，如下图所示:

![定时创建请求拉取](/images/messagequeue/rocketmq-message-receive-flow/create_pull_request_from_messagequeue.png)

当从 Broker 服务器拉取下来消息以后，只有当用户成功消费的时候，才会更新本地的偏移量表。本地的偏移量表再通过定时服务每隔 5 秒同步到 Broker 服务器端:

```java
public class MQClientInstance {    private void startScheduledTask() {        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {                @Override                public void run() {                    MQClientInstance.this.persistAllConsumerOffset();                }            }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);            }    }
```

而维护在 Broker 服务器端的偏移量表也会每隔 5 秒钟序列化到磁盘中:

```java
public class BrokerController {    public boolean initialize() throws CloneNotSupportedException {        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {                @Override                public void run() {                    BrokerController.this.consumerOffsetManager.persist();                }            }, 1000 * 10, this.brokerConfig.getFlushConsumerOffsetInterval(), TimeUnit.MILLISECONDS);    }    }
```

保存的格式如下所示：

![文件保存形式](/images/messagequeue/rocketmq-message-receive-flow/2018_03_16_22_52_14.png)

上述整体流程如下所示，红框框住的是这个话题下面的队列的 ID，箭头指向的分别是每个队列的消费偏移量：

![消费队列偏移量](/images/messagequeue/rocketmq-message-receive-flow/consume_queue_offset_update.png)

### (2) 广播模式

对于广播模式而言，每个消费队列的偏移量肯定不能存储在 Broker 服务器端，因为多个消费者对于同一个队列的消费可能不一致，偏移量会互相覆盖掉。因此，在广播模式下，每个客户端的消费偏移量是存储在本地的，然后每隔 5 秒将内存中的 `offsetTable` 持久化到磁盘中。当首次从服务器获取可消费队列的时候，偏移量不像集群模式下是从 Broker 服务器读取的，而是直接从本地文件中读取的:

```java
public class LocalFileOffsetStore implements OffsetStore {    @Override    public long readOffset(final MessageQueue mq, final ReadOffsetType type) {        if (mq != null) {            switch (type) {             case READ_FROM_STORE: {                // 本地读取                offsetSerializeWrapper = this.readLocalOffset();                // ...            }            }        }        // ...    }    }
```

![广播模式偏移量](/images/messagequeue/rocketmq-message-receive-flow/create_pull_request_from_messagequeue_broadcasting.png)

当消息消费成功后，偏移量的更新也是持久化到本地，而非更新到 Broker 服务器中。这里提一下，在广播模式下，消息队列的偏移量默认放在用户目录下的 `.rocketmq_offsets` 目录下:

```java
public class LocalFileOffsetStore implements OffsetStore {    @Override    public void persistAll(Set<MessageQueue> mqs) {        // ...        String jsonString = offsetSerializeWrapper.toJson(true);        MixAll.string2File(jsonString, this.storePath);        // ...    }    }
```

存储格式如下：

![广播模式偏移量存储格式](/images/messagequeue/rocketmq-message-receive-flow/2018_03_16_23_15_16.png)

简要流程图如下：

![广播模式偏移量更新流程](/images/messagequeue/rocketmq-message-receive-flow/consume_queue_offset_update_broadcasting.png)

## 五、拉取消息

在客户端运行着一个专门用来拉取消息的后台服务 `PullMessageService`，其接受每个队列创建 `PullRequest` 拉取消息请求，然后拉取消息:

```java
public class PullMessageService extends ServiceThread {    @Override    public void run() {        while (!this.isStopped()) {            PullRequest pullRequest = this.pullRequestQueue.take();            if (pullRequest != null) {                this.pullMessage(pullRequest);            }        }    }    }
```

每一个 `PullRequest` 都关联着一个 `MessageQueue` 和一个 `ProcessQueue`，在 `ProcessQueue` 的内部还维护了一个用来等待用户消费的消息树，如下代码所示:

```java
public class PullRequest {    private MessageQueue messageQueue;    private ProcessQueue processQueue;    }public class ProcessQueue {    private final TreeMap<Long, MessageExt> msgTreeMap = new TreeMap<Long, MessageExt>();    }
```

当真正尝试拉取消息之前，其会检查当前请求的内部缓存的消息数量、消息大小、消息阈值跨度是否超过了某个阈值，如果超过某个阈值，则推迟 50 毫秒重新执行这个请求:

```java
public class DefaultMQPushConsumerImpl implements MQConsumerInner {
    
    public void pullMessage(final PullRequest pullRequest) {
        // ...
    
        final ProcessQueue processQueue = pullRequest.getProcessQueue();
        long cachedMessageCount = processQueue.getMsgCount().get();
        long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);

        // 缓存消息数量阈值，默认为 1000
        if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
            return;
        }

        // 缓存消息大小阈值，默认为 100 MB
        if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
            return;
        }

        if (!this.consumeOrderly) {
            // 最小偏移量和最大偏移量的阈值跨度，默认为 2000 偏移量，消费速度不能太慢
            if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
                this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
                return;
            }
        }

        // ...
    }
    
}
```

当执行完一些必要的检查之后，客户端会将用户指定的过滤信息以及一些其它必要消费字段封装到请求信息体中，然后才开始从 Broker 服务器拉取这个请求从当前偏移量开始的消息，默认一次性最多拉取 32 条，服务器返回的响应会告诉客户端这个队列下次开始拉取时的偏移量。客户端每次都会注册一个 `PullCallback` 回调，用以接受服务器返回的响应信息，根据响应信息的不同状态信息，然后修正这个请求的偏移量，并进行下次请求:

```java
public void pullMessage(final PullRequest pullRequest) {
    PullCallback pullCallback = new PullCallback() {
            @Override
            public void onSuccess(PullResult pullResult) {
                if (pullResult != null) {
                    // ...
                    switch (pullResult.getPullStatus()) {
                    case FOUND:
                        // ...
                        break;
                        
                    case NO_NEW_MSG:
                        // ...
                        break;
                        
                    case NO_MATCHED_MSG:
                        // ...
                        break;
                        
                    case OFFSET_ILLEGAL:
                        // ...
                        break;
                        
                    default:
                        break;
                    }
                }
            }

            @Override
            public void onException(Throwable e) {
                // ...
            }
        };

}
```

上述是客户端拉取消息时的一些机制，现在再说一下 Broker 服务器端与此相对应的逻辑。

服务器在收到客户端的请求之后，会根据话题和队列 ID 定位到对应的**消费队列**。然后根据这条请求传入的 offset 消费队列偏移量，定位到对应的**消费队列文件**。偏移量指定的是消费队列文件的消费下限，而最大上限是由如下算法来进行约束的:

```java
final int maxFilterMessageCount = Math.max(16000, maxMsgNums * ConsumeQueue.CQ_STORE_UNIT_SIZE);
```

有了上限和下限，客户端便会开始从消费队列文件中取出每个消息的**偏移量和消息大小**，然后再根据这两个值去 `CommitLog` 文件中寻找相应的完整的消息，并添加到最后的消息队列中，精简过的代码如下所示：

```java
public class DefaultMessageStore implements MessageStore {

    public GetMessageResult getMessage(final String group, final String topic, final int queueId, final long offset,
                                       final int maxMsgNums,
                                       final MessageFilter messageFilter) {
        // ...
        ConsumeQueue consumeQueue = findConsumeQueue(topic, queueId);
    
        if (consumeQueue != null) {
            // 首先根据消费队列的偏移量定位消费队列
            SelectMappedBufferResult bufferConsumeQueue = consumeQueue.getIndexBuffer(offset);
            if (bufferConsumeQueue != null) {
                try {
                    status = GetMessageStatus.NO_MATCHED_MESSAGE;

                    // 最大消息长度
                    final int maxFilterMessageCount = Math.max(16000, maxMsgNums * ConsumeQueue.CQ_STORE_UNIT_SIZE);
                    // 取消息
                    for (; i < bufferConsumeQueue.getSize() && i < maxFilterMessageCount; i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
                        long offsetPy = bufferConsumeQueue.getByteBuffer().getLong();
                        int sizePy = bufferConsumeQueue.getByteBuffer().getInt();

                        // 根据消息的偏移量和消息的大小从 CommitLog 文件中取出一条消息
                        SelectMappedBufferResult selectResult = this.commitLog.getMessage(offsetPy, sizePy);
                        getResult.addMessage(selectResult);
                        
                        status = GetMessageStatus.FOUND;
                    }

                    // 增加下次开始的偏移量
                    nextBeginOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
                } finally {
                    bufferConsumeQueue.release();
                }
            }
        }
        // ...
    }
    
}
```

客户端和 Broker 服务器端完整拉取消息的流程图如下所示：

![客户端和 Broker 服务器端完整拉取消息的流程图](/images/messagequeue/rocketmq-message-receive-flow/pull_message_from_broker.png)

## 六、消费消息

依赖于用户指定的消息回调函数的不同，消息的消费分为两种: **并发消费**和**有序消费**。

并发消费没有考虑消息发送的顺序，客户端从服务器获取到消息就会直接回调给用户。而有序消费会考虑每个队列消息发送的顺序，注意此处并不是每个话题消息发送的顺序，一定要记住 RocketMQ 控制消息的最细粒度是**消息队列**。当我们讲有序消费的时候，就是在说对于某个话题的某个队列，发往这个队列的消息，客户端接受消息的顺序与发送的顺序完全一致。

下面我们分别看这两种消费模式是如何实现的。

### (1) 并发消费

当用户注册消息回调类的时候，如果注册的是 `MessageListenerConcurrently` 回调类，那么就认为用户不关心消息的顺序问题。我们在上文提到过每个 `PullRequest` 都关联了一个处理队列 `ProcessQueue`，而每个处理队列又都关联了一颗消息树 `msgTreeMap`。当客户端拉取到新的消息以后，其先将消息放入到这个请求所关联的处理队列的消息树中，然后提交一个消息消费请求，用以回调用户端的代码消费消息:

```java
public class DefaultMQPushConsumerImpl implements MQConsumerInner {

    public void pullMessage(final PullRequest pullRequest) {
        PullCallback pullCallback = new PullCallback() {
                @Override
                public void onSuccess(PullResult pullResult) {
                    if (pullResult != null) {
                        switch (pullResult.getPullStatus()) {
                        case FOUND:
                            // 消息放入处理队列的消息树中
                            boolean dispathToConsume = processQueue
                                .putMessage(pullResult.getMsgFoundList());

                            // 提交一个消息消费请求
                            DefaultMQPushConsumerImpl.this
                                .consumeMessageService
                                .submitConsumeRequest(
                                                      pullResult.getMsgFoundList(),
                                                      processQueue,
                                                      pullRequest.getMessageQueue(),
                                                      dispathToConsume);
                            break;
                        }
                    }
                }

            };

    }
    
}
```

当提交一个消息消费请求后，对于**并发消费**，其实现如下:

```java
public class ConsumeMessageConcurrentlyService implements ConsumeMessageService {

    class ConsumeRequest implements Runnable {

        @Override
        public void run() {
            // ...
            status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
            // ...
        }

    }
    
}
```

我们可以看到 `msgs` 是直接从服务器端拿到的最新消息，直接喂给了客户端进行消费，并未做任何有序处理。当消费成功后，会从消息树中将这些消息再给删除掉:

```java
public class ConsumeMessageConcurrentlyService implements ConsumeMessageService {

    public void processConsumeResult(final ConsumeConcurrentlyStatus status, /** 其它参数 **/) {
        // 从消息树中删除消息
        long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
        if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
            this.defaultMQPushConsumerImpl.getOffsetStore()
                .updateOffset(consumeRequest.getMessageQueue(), offset, true);
        }
    }
    
}
```

![并发消费](/images/messagequeue/rocketmq-message-receive-flow/consume_message_service.png)

### (2) 有序消费

RocketMQ 的有序消费主要依靠**两把锁**，一把是维护在 Broker 端，一把维护在消费者客户端。Broker 端有一个 `RebalanceLockManager` 服务，其内部维护了一个 `mqLockTable` 消息队列锁表:

```java
public class RebalanceLockManager {

    private final ConcurrentMap<String/* group */, ConcurrentHashMap<MessageQueue, LockEntry>> mqLockTable =
        new ConcurrentHashMap<String, ConcurrentHashMap<MessageQueue, LockEntry>>(1024);
    
}
```

在有序消费的时候，Broker 需要确保任何一个队列在任何时候都只有一个客户端在消费它，都在被一个客户端所锁定。当客户端在本地根据消息队列构建 `PullRequest` 之前，会与 Broker 沟通尝试锁定这个队列，另外当进行有序消费的时候，客户端也会周期性地 (默认是 20 秒) 锁定所有当前需要消费的消息队列:

```java
public class ConsumeMessageOrderlyService implements ConsumeMessageService {

    public void start() {
        if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())) {
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        ConsumeMessageOrderlyService.this.lockMQPeriodically();
                    }
                }, 1000 * 1, ProcessQueue.REBALANCE_LOCK_INTERVAL, TimeUnit.MILLISECONDS);
        }
    }
    
}
```

由上述这段代码也能看出，只在集群模式下才会周期性地锁定 Broker 端的消息队列，因此在**广播模式下是不支持进行有序消费的**。

而在 Broker 这端，每个客户端所锁定的消息队列对应的锁项 `LogEntry` 有一个上次锁定时的时间戳，当超过锁的超时时间 (默认是 60 秒) 后，也会判定这个客户端已经不再持有这把锁，以让其他客户端能够有序消费这个队列。

在前面我们说到过 `RebalanceService` 均衡服务会定时地依据不同消费者数量分配消费队列。我们假设 Consumer-1 消费者客户端一开始需要消费 3 个消费队列，这个时候又加入了 Consumer-2 消费者客户端，并且分配到了 MessageQueue-2 消费队列。当 Consumer-1 内部的均衡服务检测到当前消费队列需要移除 MessageQueue-2 队列，这个时候，会首先解除 Broker 端的锁，确保新加入的 Consumer-2 消费者客户端能够成功锁住这个队列，以进行有序消费。

```java
public abstract class RebalanceImpl {

    private boolean updateProcessQueueTableInRebalance(final String topic,
                                                       final Set<MessageQueue> mqSet,
                                                       final boolean isOrder) {
        while (it.hasNext()) {
            // ...
            
            if (mq.getTopic().equals(topic)) {
                // 当前客户端不需要处理这个消息队列了
                if (!mqSet.contains(mq)) {
                    pq.setDropped(true);
                    // 解锁
                    if (this.removeUnnecessaryMessageQueue(mq, pq)) {
                        // ...
                    }
                }

                // ...
            }
        }
    }
    
}
```

![有序消费加锁](/images/messagequeue/rocketmq-message-receive-flow/remove_unnessary_message_queue.png)

消费者客户端每一次拉取消息请求，如果有发现新的消息，那么都会将这些消息封装为 `ConsumeRequest` 来喂给消费线程池，以待消费。如果消息特别多，这样一个队列可能有多个消费请求正在等待客户端消费，用户可能会先消费偏移量大的消息，后消费偏移量小的消息。所以消费同一队列的时候，需要一把锁以消费请求顺序化:

```java
public class ConsumeMessageOrderlyService implements ConsumeMessageService {

    class ConsumeRequest implements Runnable {
        @Override
        public void run() {
            final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
            synchronized (objLock) {
                // ...
            }
        }
    }
    
}
```

![有序消费锁](/images/messagequeue/rocketmq-message-receive-flow/message_order_consume.png)

RocketMQ 的消息树是用 `TreeMap` 实现的，其内部基于消息偏移量维护了消息的有序性。每次消费请求都会从**消息树中拿取**偏移量最小的几条消息 (默认为 1 条)给用户，以此来达到有序消费的目的:

```java
public class ConsumeMessageOrderlyService implements ConsumeMessageService {

    class ConsumeRequest implements Runnable {
        @Override
        public void run() {
            // ...
            final int consumeBatchSize =
                ConsumeMessageOrderlyService.this
                .defaultMQPushConsumer
                .getConsumeMessageBatchMaxSize();
            List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize);
            
        }
    }
    
}
```

![有序消费树](/images/messagequeue/rocketmq-message-receive-flow/order_take_message.png)
