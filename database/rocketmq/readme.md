# RocketMQ


[官方Quick-Start](https://rocketmq.apache.org/docs/quick-start/)

[官方中文文档](https://github.com/apache/rocketmq/tree/master/docs/cn)



## 版本

4.8版本



## RocketMQ-Console

控制台

[部署方法](https://juejin.cn/post/6948790866962022436)



## CommitLog

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



## 消费者客户端负载

**消费者组如何知道自己该处理某一个topic的哪些队列？**

一个jvm进程正常只有一个MQ客户端实例(MQClientInstance)，提供了很多定时服务，其中就包括了“负载均衡”定时服务(RebalanceService)，每20秒会触发一次ReBalance操作，每次触发都会调用消费者对象的rebalance接口，这个消费者对象内部有一个rebalance的实现对象，在消费者启动的时候，它会将topic订阅信息copy到rebalance实现对象的map中，这个负载均衡实现对象会根据订阅信息和主题队列分布信息去计算分配给自己的队列

#### 消费者负载流程

1. 消费者启动向MQClientInstance注册
2. 消费者内部会有一个ReBalance实现类，消费者会将Topic和ReBalance的实现类注册进MQClientInstance的consumerTable
3. MQClientInstance 内部的RebalanceService（线程任务），每20秒执行一次 doReBalance，遍历consumerTable，调用消费者内部ReBalance的实现类的doRebalance
4. 消费者内部对消费者组内成员和Topic对应的消息队列排序，做统一视图
5. 根据负载算法确认自己可以消费的消息队列

#### 计算分配算法

首先消费者知道两组信息：

1. 消费者组内有哪些成员（通过向broker拉取，nameServer？）nameServer通过心跳收集消费者信息
2. topic在broker中消息队列的分配信息

平均分配算法（默认）：消费者对象会在本地对消费者列表排序，得到老一、老二、老三这样一个顺序

计算每个消费者分配的队列数，队列总数/消费者数，余数按消费者列表由大到小再分配



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
   3. 将“长轮询”对象交给长轮询服务 （PullRequestHoldService） ，本次拉消息先不给返回结果
   4. 长轮询服务有自己的工作线程，是一个死循环，每几秒钟执行一次，检查”长轮询“对象，提取出”长轮询“对象的拉消息参数
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





## 问题

### 生产者的组有什么用？

高可用

### 重试次数

默认16次