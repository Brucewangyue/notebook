# RocketMQ


[官方Quick-Start](https://rocketmq.apache.org/docs/quick-start/)

[官方中文文档](https://github.com/apache/rocketmq/tree/master/docs/cn)

## 介绍

- 削峰/限流
- 服务解耦



## 版本

4.8版本



## RocketMQ-Console

控制台

[部署方法](https://juejin.cn/post/6948790866962022436)



## CommitLog

一个commitLog文件大小为1G，文件名是偏移量的起始位置，不够20位(10进制)在前面用0补齐

假设是单机Broker实例，那么

- 一个Topic所对应的逻辑消息队列，共同存储在一个commitLog中
- producer发送过来的消息也是通过刷盘写到一个commitLog中
- 一个Broker实例下，所有的队列共用一个commitLog

消息存储的时候是顺序写入到commitLog文件



broker会锁定commitLog对应内存的映射空间，不让它释放物理内存，而且当前顺序写的commitLog对应的内存预热缓冲区，在创建的时候呢会进行预热，避免写消息时产生缺页异样，然后才开始去申请虚拟内存页对应的物理内存页，另外平台的物理内存是有限的，那些被写满的commitLog文件内存映射缓冲区会解锁，让内存空间紧张的时候把这个虚拟内存页对应的物理内存也置换出去

释放缓冲区的算法一般都是LRU算法，由于RocketMQ是顺序写，那么这个LRU算法会退化为FIFO算法，也就是说最早的内存页会先被释放



## 事务消息

4.1版本之后提供

最终一致性

不适用于一致性要求高的场景，如：

互联网项目，用户表都比较大，采用分库分表存储用户数据，假设张三给李四转账，张三和李四在不同的数据库，采用RocketMQ来实现的话，张三扣款成功，李四立马进行了销户处理，那么MQ消费者客户端不知道该如何处理这条消息了，只能是再发一条MQ消息来让张三恢复余额，那么这里是让另外一个程序来保证程序，相当复杂而且不靠谱

### 事务消息流程

1. 创建并发送half消息到RocketMQ (unit_id 作为事务id)
   1. half消息的状态为prepare
   2. half消息的属性保存了原消息的topic, queueid
2. RocketMQ收到half消息并保存到commitLog文件，因为消费者这边订阅的是原topic，所以这个时间段内并不会消费
3. 执行half消息关联的本地事务，最终commit/rollback
4. broker接收到commit，会根据half消息的属性创建一个原topic的消息，并且写入commitLog
5. 软删除half消息



#### 软删除half消息：

broker有一个确认队列，保存opMsg，消息体是half消息的偏移量，通过标记的方式达到软删除的效果



**为什么不直接删除？**

因为rocketmq是顺序写的，如果直接物理删除，性能会受到影响



#### 事务消息二阶段如何保障

broker有一个定时的事务回查服务，每分钟都会检查处于prepare状态的half消息，如果发现了就会向消息生产者组内的任意一个生产者发起事务状态回查的rpc请求

**消息回查**

通常都要在本地创建本地事务表，存储事务ID，用于检查本地事务状态，然后生产者再次向broker发送事务状态

## 负载均衡

### Producer负载均衡

发送端指定message queue发送消息到相应的Broker，来达到写入时的负载均衡

默认策略是：跟queue列表取模



### Broker负载均衡



### Consumer负载均衡

**消费者组如何知道自己该处理某一个topic的哪些队列？**

一个jvm进程正常只有一个MQ客户端实例(MQClientInstance)，提供了很多定时服务，其中就包括了“负载均衡”定时服务(RebalanceService)，每20秒会触发一次ReBalance操作，每次触发都会调用消费者对象的rebalance接口，这个消费者对象内部有一个rebalance的实现对象，在消费者启动的时候，它会将topic订阅信息copy到rebalance实现对象的map中，这个负载均衡实现对象会根据订阅信息和主题队列分布信息去计算分配给自己的队列

#### 消费者负载流程

1. 消费者启动向MQClientInstance注册
2. 消费者内部会有一个ReBalance实现类，消费者会将Topic和ReBalance的实现类注册进MQClientInstance的consumerTable
3. MQClientInstance 内部的RebalanceService（线程任务），每20秒执行一次 doReBalance，遍历consumerTable，调用消费者内部ReBalance的实现类的doRebalance
4. 消费者内部对消费者组内成员和Topic对应的消息队列排序，做统一视图
5. 根据负载算法确认自己可以消费的消息队列

#### 负载分配算法

首先消费者知道两组信息：

1. 消费者组内有哪些成员（通过向broker拉取，nameServer？）nameServer通过心跳收集消费者信息
2. topic在broker中消息队列的分配信息

**平均分配算法（默认）**：消费者对象会在本地对消费者列表排序，得到老一、老二、老三这样一个顺序

计算每个消费者分配的队列数，队列总数/消费者数，余数按消费者列表由大到小再分配

其他还有：

- 环形分配策略(AllocateMessageQueueAveragelyByCircle)
- 手动配置分配策略(AllocateMessageQueueByConfig)
- 机房分配策略(AllocateMessageQueueByMachineRoom)
- 一致性哈希分配策略(AllocateMessageQueueConsistentHash)
- 靠近机房策略(AllocateMachineRoomNearby)



## 消费消息

### 如何发起拉消息

首先我们知道：每个队列的消费进度是存储在队列所属broker节点上的

当消费者知道自己需要消费哪些队列后，就开始向Broker发送RPC请求，拉取消息队列的消费进度，然后保存在消费者内部的 OffsetStore 对象

有了这个消费进度后就可以创建一个PullRequest对象，用于保存队列信息和拉消息的位点信息，然后将这个对象交给PullMessageService，这个拉消息服务内部有一个LinkedBlockingQueue，用于存放PullRequest对象

这个拉消息服务有自己的线程资源 ScheduledExecutorService（1核心线程数），这个线程启动之后会不停的消费这个BrockingQueue，异步的读取PullRequest对象，然后发起拉消息请求

拉下来一批消息以后呢，会更新PullRequest下一次拉消息的位点信息，然后再放入BrockingQueue，形成一个拉消息的闭环

### 拉下消息后消费

消费消息也有自己的线程资源

拉下来的消息都会被封装成消息消费任务，然后提交到“并发消息消费”(consumeMessageService的其中实现)内的线程池，最终都会被线程池内的线程get走

```java
// ConsumeMessageOrderlyService
this.consumeExecutor = new ThreadPoolExecutor(
    this.defaultMQPushConsumer.getConsumeThreadMin(),
    this.defaultMQPushConsumer.getConsumeThreadMax(),
    1000 * 60,
    TimeUnit.MILLISECONDS,
    this.consumeRequestQueue,
    new ThreadFactoryImpl("ConsumeMessageThread_"));
```

“消息消费任务”最核心的逻辑就是执行用户注册在consumer的messageListener对象

**如果消费成功，直接更新消费者本地的offsetStore对象里的消费进度，如果消费失败，会回退消息到broker节点，会进入“重试主题”(**每个消费者组都有自己专属的重试主题 %GROUP%RetryTopic)

### 重试主题

消费者在启动的时候，会订阅自己组相关的重试主题



## 延迟队列

**如果消费消息失败，那么消费者会立马再次消费吗？**

不会，立马再次消费，大概率也是消费失败，不是一个好的策略

broker收到重试消息以后，不会直接将消息放入重试主题，而是先放入延时主题(SCHEDULE_TOPIC_xxx)

### 延时队列创建

延时主题有很多个级别，每个级别对应不同的延时时常，那么是如何实现的呢？

延时Topic的属性保存了重试Topic和队列，延时主题根据重试次数设置queueId，比如第一次是0，第二次重试是1，延时Topic根据queueId来决定延时级别

延时时间到了之后，处理逻辑跟half主题一样



## 长轮询

如果消费者的进度赶上了broker的队列进度，消费者是否会不停的rpc拉消息？

首先客户端是无脑拉，但是broker会做处理，通过长轮询技术

### 拉消息请求处理

拉消息请求，进入到broker后，由PullMessageProcessor协议处理器去处理，核心逻辑：

1. 根据拉消息请求的参数来查询指定offset位点的消息
2. 查询到数据，直接返回给消费者
3. 查询不到数据：消费进度赶上了生产进度
   1. 这个时候如果返回给客户端，客户端会立马再次发起拉消息请求，所以是在这一步进行控制
   2. 服务端会创建一个“长轮询”对象，保存两个关键信息
      - 拉消息请求的位点信息
      - broker和客户端的Netty Channel会话对象
   3. 将“长轮询”对象交给拉请求保持服务 （PullRequestHoldService） ，本次拉消息先不给返回结果
   4. 拉请求保持服务有自己的工作线程，是一个死循环，每30秒钟执行一次，检查”长轮询“对象，提取出”长轮询“对象的拉消息参数
   5. 根据这些参数再次检查队列最大的offset，判断是否有新数据
      1. 如果有新数据，那么会接着从“长轮询”对象中提取出Netty Channel会话对象，再次调用PullMessageProcessor协议处理器去处理拉消息逻辑
      2. 这一次如果还是没有查询到消息，也会把结果返回给客户端
4. 当有生产者发来新消息，会主动结束长轮询



## 读写分离

### 写逻辑

消息肯定是要写入Master节点的，然后底层通过HA同步到从节点

**RocketMQ的读写分离和传统数据的读写分离不是一个概念，传统数据库的读写分离是写在主库，读在从库**

### 读逻辑

每次消费者到broker拉消息，broker会根据拉消息的结果集中的最后一条消息来推荐下次拉消息是从”主“还是”从“中拉下一批消息，如果最后一条消息是热数据，那么下一次拉消息还是会从Master来拉取，如果是冷数据才去slave拉取

### 热数据

冷数据：最后一条消息的offset距离commitLog最大offset大小的超过系统内存的40%，此时消费者大概率处于消息积压的状况，然后就推荐消费者下次拉消息从slave节点拉取



## 权限认证ACL

RocketMQ 4.4版本开始支持



## 线上版本升级

https://developer.51cto.com/article/658043.html



## 顺序消费

1. 生产者，将业务Id与固定的消息队列(queueId)做映射：发消息时取模
2. 消息消费者使用的监听器类型为顺序型，而非并发型监听器
3. 顺序型监听器在重试时会阻塞当前重试的topic所在的队列，默认每2秒重试一次



## 保证消息不丢

RocketMQ的消息生命周期分为三个阶段，每个阶段都有可能造成数据的丢失

- 生产者生产消息阶段：通过网络发送消息给Broker，中间可能出现网络抖动，造成Broker没有真正接收到消息
- Broker存储消息阶段：存储消息的过程中宕机了，最后的未真正刷盘的消息会丢失
- 消费者消费消息阶段：消费失败了，但是Broker不知道，没有重试

### Producer生产阶段

在Producer发送消息时，有三种消息发送方式

- 同步发送
- 异步发送
- 单向发送

#### 同步发送

发送消息时同步阻塞等待Broker返回结果，如果没有成功则不会收到SendResult，这种是最可靠的

```java
public SendResult send(Message msg) throws MQClientException, RemotingException, 	 MQBrokerException, InterruptedException {}
```



#### 异步发送

发送消息不阻塞，Broker收到消息会回调Producer发送消息时注册的回调方法得知是否发送成功

```java
public void send(Message msg,SendCallback sendCallback) throws MQClientException, RemotingException, InterruptedException {}
```

这种方式可能会造成消息的重新发送，有一种情况，其实消息已经成功到达Broker，但是Broker的回调失败了，那么Producer会认为没有成功，后续如果业务做了定时回查自己本地业务表状态，发现状态是没有发送成功，那么业务再次将这些失败消息再次发送就会造成消息重复，对于这种消息，消费者需要对业务id做幂等判断



#### 单向发送

最不安全，不清楚有没有发送成功

```java
public void sendOneway(Message msg) throws MQClientException, RemotingException, InterruptedException {}
```



#### 消息重试

Producer在发送消息失败后，会自动向其他的Broker节点再次发送消息

只有同步发送和异步发送有重试策略

- 同步发送：设置的发送失败重试次数(默认是2)+1
- 异步发送：设置的发送失败重试次数  ??

**源码**

```java
/**
 * {@link org.apache.rocketmq.client.producer.DefaultMQProducer#sendDefaultImpl(Message, CommunicationMode, SendCallback, long)}
 */
```

#### 总结

1. 使用同步发送消息
2. 自动重试
3. Broker做集群



### Broker存储阶段

Broker接收到消息，会根据配置的刷盘策略刷盘，如果在下次刷盘前宕机，那么消息会丢失，刷盘策略有：

- 同步刷盘
- 异步刷盘（默认）

#### 同步刷盘

当Broker收到消息后，会立马同步刷盘，等待刷盘完成才会向Producer返回，这样就不会造成生成者发送的消息丢失，但是性能差，需要按业务场景取舍

```properties
## 默认情况为 ASYNC_FLUSH，修改为同步刷盘：SYNC_FLUSH
flushDiskType = ASYNC_FLUSH 
```

对应的类

```java
public enum FlushDiskType {
    // 同步刷盘
    SYNC_FLUSH,
    // 异步刷盘（默认）
    ASYNC_FLUSH
}
```

#### 异步刷盘

当Broker收到消息后，先存放在缓冲区，然后立马给Producer返回，这种方式最多会丢失一个缓冲区的数据

异步刷盘默认10s执行一次，源码如下：

```java
// {@link org.apache.rocketmq.store.CommitLog#run()}
while (!this.isStopped()) {
    try {
        // 等待10s
        this.waitForRunning(10);
        // 刷盘
        this.doCommit();
    } catch (Exception e) {
        CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
    }
}
```

#### Broker集群

有以下极端情况

1. 如果是单节点，就算是刷盘完成了，那么此时磁盘坏了，数据没了
2. 如果是1主1从，在主刷盘完成后，给生成这返回，但是在将数据同步到从节点前挂了，从节点没同步到消息，也就丢失了

RocketMQ还提供了更为严格的数据一致性方式：主节点配置阻塞同步，必须等消息同步到从节点才给生产者返回

这种配置就更影响性能了，适用需要一致性比较强的场景才适用

```properties
## 默认为 ASYNC_MASTER
brokerRole=SYNC_MASTER
```

这么强的一致性必然带来可用性的问题，其中一个节点挂了那么服务就不可用了 ？？

#### 总结

主从节点都开启同步刷盘、主节点配置成同步主节点



### Consumer消费阶段

消费者从Broker拉取新消息，消费成功后需要给Broker返回ack，如果Broker没有收到ack，那么会针对不同的消费者做不同的处理

#### 消费者消费模式

消费模式由Consumer决定

- 集群模式：一个消费者组内只有一个成员会处理某一条消息
- 广播模式：一个消费者组内所有消费者都会处理同一条消息

广播模式是没有重试机制的



#### 并发消费

集群模式下的并发消费：如果消费者没有向Broker返回ack，那么Broker会根据有问题的Topic和重试次数来创建**延时消息(SCHEDULE_TOPIC_XXXX)**，等待一段时间之后会根据延时消息来创建**重试消息(“%RETRY%+consumerGroup”)**，消费者在启动的时候已经对相关的Topic的重试消息进行了订阅

> 延时消息包含一个延时级别，延时消息根据Topic重试次数来决定使用哪个延时级别
>
> 延时级别：1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h (18个level)
>
> - 可以在Broker配置自定义messageDelayLevel，作用于所有Topic-
> - 每个延时级别对应一个Queue，queueId = delayTimeLevel – 1，即一个queue只存相同延迟的消息



#### 顺序消费

集群模式下的顺序消费：如果消费者没有向Broker返回ack，那么Broker会阻塞这个Topic所在的队列，然后每两秒重试一次，没有重试限制，直到消费者最终返回消息

**如果消费者挂了怎么处理？**

首先我们知道消费者会向NameServer上报心跳包，消费者如果所属的消费者组有其他成员，他们在一段时间后是会知道消费者组的成员列表少了，并且消费者在启动的时候就开启一个Rebalance的线程，默认每20秒更新自己能处理哪些Topic所属的消息队列，在经过平衡后，Broker的消息队列会被其他的消费者接管



## 消息重复处理



## 消息积压处理

下游消费系统如果宕机了，导致几百万条消息在消息中间件里积压，此时怎么处理?

你们线上是否遇到过消息积压的生产故障?如果没遇到过，你考虑一下如何应对?

然后看下消息消费速度是否正常，正常的话，可以通过上线更多consumer临时解决消息堆积问题



**追问：如果Consumer和Queue不对等，上线了多台也在短时间内无法消费完堆积的消息怎么办？**

也就是说一开始的Topic设置的队列数过少，上线多台Consumer也无法消费

- 准备一个临时的topic
- queue的数量是堆积的几倍

queue分布到多Broker中

上线一台Consumer做消息的搬运工，把原来Topic中的消息挪到新的Topic里，不做业务逻辑处理，只是挪过去

上线N台Consumer同时消费临时Topic中的数据

改bug



**追问：堆积时间过长消息超时了？**

RocketMQ中的消息只会在commitLog被删除的时候才会消失，不会超时。也就是说未被消费的消息不会存在超时删除这情况



## 问题

### 生产者的组有什么用？

高可用

### 重试次数

默认16次

### 什么时候会清理过期消息？

默认是48小时后会删除不再使用的CommitLog，另外可以指定删除时间（默认凌晨4点）

```java
/**
 * {@link org.apache.rocketmq.store.DefaultMessageStore.CleanCommitLogService#isTimeToDelete()}
 */
private boolean isTimeToDelete() {
    // when = "04";
    String when = DefaultMessageStore.this.getMessageStoreConfig().getDeleteWhen();
    // 是04点，就返回true
    if (UtilAll.isItTimeToDo(when)) {
        return true;
    }
	// 不是04点，返回false
    return false;
}

/**
 * {@link org.apache.rocketmq.store.DefaultMessageStore.CleanCommitLogService#deleteExpiredFiles()}
 */
private void deleteExpiredFiles() {
    // isTimeToDelete()这个方法是判断是不是凌晨四点，是的话就执行删除逻辑。
    if (isTimeToDelete()) {
        // 默认是72，但是broker配置文件默认改成了48，所以新版本都是48。
        long fileReservedTime = 48 * 60 * 60 * 1000;
        deleteCount = DefaultMessageStore.this.commitLog.deleteExpiredFile(72 * 60 * 60 * 1000, xx, xx, xx);
    }
}
                                                                       
/**
 * {@link org.apache.rocketmq.store.CommitLog#deleteExpiredFile()}
 */
public int deleteExpiredFile(xxx) {
    // 这个方法的主逻辑就是遍历查找最后更改时间+过期时间，小于当前系统时间的话就删了（也就是小于48小时）。
    return this.mappedFileQueue.deleteExpiredFileByTime(72 * 60 * 60 * 1000, xx, xx, xx);
}
```

### 为什么要主动拉取消息而不使用事件监听方式？

对于时间监听方式，是通过建立长连接，由Broker主动给消费者推

如果Broker推送消息过快，消费者会积压消息，且到达消费者后，不能被同一个消费者组的其他成员消费，而Pull的方式是消费者根据自身情况来Pull，不会造成过多的压力而造成瓶颈