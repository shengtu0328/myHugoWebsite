---
title: "RocketMQ"
date: 2023-01-17T17:21:36+08:00
draft: true
---

### RoctetMQ基本架构

![RocketMQ基本模型](https://rocketmq.apache.org/zh/assets/images/RocketMQ%E5%9F%BA%E6%9C%AC%E6%A8%A1%E5%9E%8B-ebcf3458d04b36f47f4c9633c1e36bf7.png)



![RocketMQ架构](https://cdn.tobebetterjavaer.com/tobebetterjavaer/images/nice-article/weixin-mianznxrocketmqessw-d4c0e036-0f0e-466f-bd4b-7e6ee10daca4.jpg)





### 几种MQ对比

Kafka：追求高吞吐量，一开始的目的就是用于日志收集和传输，**适合产生大量数据的互联网服务的数据收集业务**，大型公司建议可以选用，**如果有日志采集功能，肯定是首选 kafka。**

RocketMQ：**天生为金融互联网领域而生，对于可靠性要求很高的场景**，尤其是电商里面的订单扣款，以及业务削峰，在大量交易涌入时，后端可能无法及时处理的情况。RoketMQ 在稳定性上可能更值得信赖，这些业务场景在阿里双 11 已经经历了多次考验，**如果你的业务有上述并发场景，建议可以选择 RocketMQ。**

RabbitMQ：结合 erlang 语言本身的并发优势，性能较好，社区活跃度也比较高，但是不利于做二次开发和维护，不过 RabbitMQ 的社区十分活跃，可以解决开发过程中遇到的 bug。**如果你的数据量没有那么大，小公司优先选择功能比较完备的 RabbitMQ。**



作者：楼仔
链接：https://juejin.cn/post/7096095180536676365
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

#### rocketmq

支持集群模型，负载均衡，水平扩容

采用零拷贝原理，顺序写盘，随机读

丰富API（顺序消息，延迟消息，事物消息）

消息失败自动重试机制

消息可查询 messagekey，messageid

java写的

数据持久化：RocketMQ将消息存储在磁盘上，保证消息的可靠性



#### Kafka

会丢数据，主要用于日志



  ![](Kafka高可用.png)

#### rabbitmq

erlang

吞吐量比较低

支持优先级队列

RabbitMQ默认将消息保存在内存中





### Producer Group有什么用

Rroducer Group：生产者集合，一般用于发送一类消息。



1.一个组里多个生产者并行发消息，提高性能

2.主要是在做事物消息回查的时候用，如果连一个生产者失败，可以连另一个

一个jvm只能定义一个 Producer Group



### Topic



默认一个topic下有四个队列

集群环境中一个 topic下的队列，有可能会出现在多个broker下，这样可以保证高可用。

  ![](Kafka_Partition.png)





### Message

messageId 自动生成，唯一

messagekey 手动指定的业务key，最好也设置成唯一

按 Message Key 查询 

按 Message Key 查询消息的原理是，消息队列 RocketMQ 根据您设置的 Message Key 建立消息的索引信息 ，当您输入 Key 进行查询时，消息队列 RocketMQ 根据该索引即可匹配相关的消息返回。

 注意： 按 Message Key 查询的条件是用户设置的 Message Key 属性。 按 Message Key 查询仅仅返回符合条件的最近的 64 条消息，因此建议您尽可能保证设置的 Key 是 唯一的，并具有业务区分度。 设置 Message Key 的方法如下：



```
Message msg = new Message("Topic","*","Hello MQ".getBytes()); 

/** * 对每条消息设置其检索的 Key，该 Key 值代表消息的业务关键属性，请尽可能全局唯一。 * 以方便您在无法正常收到消息情况下，可通过消息队列 RocketMQ 控制台查询消息。 不设置也不会影响消息正常收发。 */
msg.setKey("TestKey"+System.currentTimeMillis());
```



#### 顺序消息

在需要保证 一个订单如果有5个步骤，每个订单里都必须要按照 1,2,3,4,5执行。但是订单a和订单b之前的消费顺序有可能会穿插。这种局部有序，总体无序就是顺序消息。一组有序的消息发送到topic下的一个message queue。



#### 延迟消息

> messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h





#### 事物消息

保证生产者发送消息到broker 和 生产者写入本地数据库 在一个事物里，至于消费者成功失败不管。



  ![](transactionMessage.png)





### Producer



### Consumer

消费者的监听 默认要实现 MessageListenerConcurrently接口



消息消费失败会自动重试，rockemq根据消费者组会有重试队列。如果超过16次，放入死信队列。









### 主从

主节点挂了，也可以从从节点消费，并且主再次启动后，会同步从的offset，会知道消息已经被消费过了。













### 源码版本下载编译

1. 解压下载的源码包并编译构建二进制可执行文件

   unzip rocketmq-rocketmq-all-4.3.0.zip

   cd rocketmq-rocketmq-all-4.3.0

   mvn -Prelease-all -DskipTests clean install -U

   cd distribution/target 



2. 取出 distribution/target 目录下的 apache-rocketmq.tar.gz 作为rocketmq启动包，

   解压到./bin-release/ 目录

   tar -zxvf  apache-rocketmq.tar.gz -C ./bin-release/ 



3. 查看解压结果

    ls
   LICENSE		README.md	bin		lib
   NOTICE		benchmark	conf



#在bin-release下配置rocketmq生成文件的路径

#mkdir store

#mkdir store/commitlog

#mkdir store/consumequeue

#mkdir store/index



#修改rocketmq配置文件
#mkdir store



### rocketmq-dashboard(控制台)

https://github.com/apache/rocketmq-dashboard



### 消息的存储结构

![在这里插入图片描述](https://img-blog.csdnimg.cn/54b0b787ca1a4860ba7c7353025fdc55.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzQ1NDA2MDky,size_16,color_FFFFFF,t_70)

#### 存储

#### commitLog

实际存储消息数据（物理文件）**所有的Topic**的消息实体内容都存储于一个CommitLog中

顺序写（写入效率高），随机读

commitLog 除了保存了消息信息，还包含了consumequeue的信息





2.2 RocketMQ消息存储架构深入分析
从上面的整体架构图中可见，RocketMQ的混合型存储结构针对Producer和Consumer分别采用了数据和索引部分相分离的存储结构，Producer发送消息至Broker端，然后Broker端使用同步或者异步的方式对消息刷盘持久化，保存至CommitLog中。只要消息被刷盘持久化至磁盘文件CommitLog中，那么Producer发送的消息就不会丢失。正因为如此，Consumer也就肯定有机会去消费这条消息，至于消费的时间可以稍微滞后一些也没有太大的关系。退一步地讲，即使Consumer端第一次没法拉取到待消费的消息，Broker服务端也能够通过长轮询机制等待一定时间延迟后再次发起拉取消息的请求。

这里，RocketMQ的具体做法是，使用Broker端的后台服务线程—ReputMessageService不停地分发请求并异步构建ConsumeQueue（逻辑消费队列）和IndexFile（索引文件）数据（ps：对于该服务线程在消息消费篇幅也有过介绍，不清楚的童鞋可以跳至消息消费篇幅再理解下）。

然后，Consumer即可根据ConsumerQueue来查找待消费的消息了。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。

而IndexFile（索引文件）则只是为了消息查询提供了一种通过key或时间区间来查询消息的方法（ps：这种通过IndexFile来查找消息的方法不影响发送与消费消息的主流程）。
————————————————
版权声明：本文为CSDN博主「云川之下」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/m0_45406092/article/details/119671706

#### 检索

##### consumequeue根据tag查询  

ConsumeQueue文件可以看做是基于topic的CommitLog索引文件，所以ConsumeQueue文件夹的组织方式如下：topic/queue/file三层组织结构；具体存储路径为：$HOME/store/consumequeue/{topic}/{queueId}/{fileName}。

Consumer 消费时使用，存储了消息在commitLog中的位置 （类似数据库索引文件）

1. 通过broker保存的offset（offsetTable.offset json文件中保存的ConsumerQueue的下标）可以在ConsumeQueue中获取消息，从而快速的定位到commitLog的消息位置

   

##### indexFile根据 MessageID查询  

因为MessagelD就是用broker+-offset生成的（这里Msgld指的是服务端的），所以很容易就找到对应的commitLog.文件来读取消息。

index

通过MessageID或者MessageKey消息会使用到



如果我们需要根据消息ID，来查找消息，consumequeue 中没有存储消息ID,如果不采取其他措施，又得遍历 commitlog文件了，indexFile就是为了解决这个问题的文件





### 零拷贝

  ![](ByteBuffer.png)

- Java 本身并不具备 IO 读写能力，因此 read 方法调用后，要从 Java 程序的**用户态切换至内核态**，去调用操作系统（Kernel）的读能力，将数据读入**内核缓冲区**。这期间用户线程阻塞，操作系统使用 DMA（Direct Memory Access）来实现文件读，其间也不会使用 CPU

- 从**内核态**切换回**用户态**，将数据从**内核缓冲区**读入**用户缓冲区**（即 byte[] buf），这期间 **CPU 会参与拷贝**，无法利用 DMA
- 调用 write 方法，这时将数据从**用户缓冲区**（byte[] buf）写入 **socket 缓冲区，CPU 会参与拷贝**



  ![](DirectByteBuffer.png)

大部分步骤与优化前相同，唯有一点：**Java 可以使用 DirectByteBuffer 将堆外内存映射到 JVM 内存中来直接访问使用**

### 同步刷盘，异步刷盘，同步复制，异步复制

https://www.cnblogs.com/toUpdating/p/10021372.html

通常情况下，应该把Master和Slave设置成ASYNC_FLUSH的刷盘方式，

主从之间配置成SYNC_MASTER的复制方式，这样即使有一台机器出故障，仍然可以保证数据不丢。

**也就是推荐 同步双写，异步刷盘。**

DirectBuffer.png



### 

### 怎么保证消息不丢失

一.Producer 新建消息，然后通过网络将消息投递给 MQ Broker

1.同步发送/异步发送 同步还是异步的方式，都会碰到网络问题导致发送失败的情况。针对这种情况，我们可以设置合理的重试次数，当出现网络问

2.记录失败日志，保存到数据库后定时重发

二.Broker 存储阶段

消息只要到了 Broker 端，将会优先保存到内存中，然后立刻返回确认响应给生产者。 

同步刷盘：当消息持久化到broker的磁盘后才算是消息写入成功。
异步刷盘：当消息写入到broker的内存后即表示消息写入成功，无需等待消息持久化到磁盘。异步刷盘策略会降低系统的写入延迟，RT变小，提高了系统的吞吐量 。消息写入到Broker的内存，一般是写入到了PageCache。对于异步刷盘策略，消息会写入到PageCache后立即返回成功ACK。但并不会立即做落盘操 作，而是当PageCache到达一定量时会自动进行落盘

三.消费者从 broker 拉取消息

消息队列 RocketMQ 默认允许每条消息最多重试 16 次

ACK确认

### 消息重试

若Consumer消费某条消息失败，则RocketMQ会在重试间隔时间后，将消息重新投递给Consumer消费，若达到最大重试次数后消息还没有成功被消费，则消息将被投递至死信队列













![消费位点](https://rocketmq.apache.org/zh/assets/images/%E6%B6%88%E8%B4%B9%E4%BD%8D%E7%82%B9-3b0320b183d4318d6b75e3504027e436.png)

如上图所示，在Apache RocketMQ中每个队列都会记录自己的最小位点、最大位点。针对于消费组，还有消费位点的概念，

**在集群模式下，消费位点是由客户端提给交服务端保存的，**

**在广播模式下，消费位点是由客户端自己保存的。**

一般情况下消费位点正常更新，不会出现消息重复，但如果消费者发生崩溃或有新的消费者加入群组，就会触发重平衡，重平衡完成后，每个消费者可能会分配到新的队列，而不是之前处理的队列。为了能继续之前的工作，消费者需要读取每个队列最后一次的提交的消费位点，然后从消费位点处继续拉取消息。但在实际执行过程中，由于客户端提交给服务端的消费位点并不是实时的，所以重平衡就可能会导致消息少量重复。

一个 Topic 可能有多个队列，并且可能分布在不同的 Broker 上。
