
#### Kafka事务性

在分布式系统中，构成系统的每个节点都是容许挂掉的。挂掉后还能正常的提供服务。比如kafka中的broker挂了，producer发送数据时网络不通等问题，在kafka里面按照语义分为以下三种：
- **at-least-once**：如果producer收到来自Kafka broker的ack或者acks = all，则表示该消息已经写入到Kafka log里面。但如果producer 收取ack超时或收到错误，则可能会retry发送消息，Client会认为该消息未写入Kafka log。如果broker在发送Ack之前失败，但在消息成功写入Kafka之后，将导致该消息被写入两次，因此消息会被不止一次地传递给最终consumer，这种策略可能导致consumer消费重复的数据。
- **at-most-once**：如果在ack超时或返回错误时producer不重试，则该消息可能最终不会写入Kafka，因此不会传递给consumer。在大多数情况下，这样做是为了避免重复的可能性，业务上必须接收数据传递可能的丢失。
- **exactly-once**：即使producer重试发送消息，消息也会保证最多一次地传递给最终consumer。该语义是最理想的，但也难以实现，这是因为它需要消息系统本身与生产和消费消息的应用程序进行协作。例如，如果在consumer消费消息成功后，将Kafka consumer的偏移量rollback，我们将会再次从该偏移量开始接收消息,此时数据又重复了，所以在consumer端消费了数据后应该主动维护一个offset记录该数据已经被消费了不再消费该offset的数据了。这表明消息传递系统和客户端应用程序必须配合调整才能实现excactly-once。

了解了kafka producer 场景语义后，kafka社区文档中举了几个例子来解释Exactly-once的语义。

例子：正常情况下是这样的。
一个单进程的Producer向某个partition的topic发送数据，然后consumer订阅了该topic时，此时consumer消费完该数据后进行commit到kafka来完成此次数据的处理。这样是满足消息的exactly once传递。
但是我们不仅要考虑争确的场景，还需要考虑异常场景：

1、**Broker宕机**：在前面提到了分布式系统的特点就是其中某个组成分布式系统的点宕机，不会影响集群服务，因为kafka的备份机制保证了一旦消息被成功写入leader 副本，将会把数据同步到其他ISR所有副本。

2、**Producer到Broker的RPC失败**：此时producer只知道ack没有返回，但并不知道是kafka的数据有没有写入到log中，所以他会尝试重复发送数据，此时如果broker上次写了这条数据，那么这条数据就是重读数据了，此时consumer消费就会消费到了重复数据。


##### Apache Kafka的exactly-once语义
exactly-once语义主要三个逻辑：

1、幂等性
这样producer发送重复数据给broker时，broker只会写一次，需要在producer配置：

```
enable.idempotence=true
```
工作原理就是在producer发送数据是带了个key在写入broker时对比key是否相同，相同就不写入broker了。

2、事务：跨partition的原子性写操作(Atomic writes across multiple partitions)
kafka通过新增的事务API来保证跨分区的事务原子性操作：
``` java
producer.initTransactions();
try {
  producer.beginTransaction();
  producer.send(record1);
  producer.send(record2);
  producer.commitTransaction();
} catch(ProducerFencedException e) {
  producer.close();
} catch(KafkaException e) {
  producer.abortTransaction();
}
```
通过以上的API可以进行跨分区批量发送message的事务一致性。该功能同样支持在同一个事务中提交消费者offsets。因此真正意义上实现了end-to-end的exactly-once 传输语义。

从consumer的角度来看，有两种策略去读取事务写入的消息，通过"isolation.level"来进行配置：
- read_committed：可以同时读取事务执行过程中的部分写入数据和已经完整提交的事务写入数据；
- read_uncommitted（默认）：完全不等待事务提交，按照offsets order去读取消息，也就是兼容0.11.x版本前Kafka的语义

3、Exactly-once 流处理
at lestest once+幂等性= exactly once

#### Consumer端的exactly once处理的条件，经过思考与实践发现，应该是这样的：

首先（配置保证）消费者端的配置应为：
```
enable-auto-commit: false
auto-offset-reset: earliest
```
这样一来我们就能在程序中自主控制commit offset的时机了。

其次（业务保证）消费者对消费数据的业务要轻处理，业务逻辑要简单，比如直接写库，直接写内存，直接打印等操作。这样会避免因为消费者端在消费数据时有bug导致不能commit offset，可能会导致消费之端循环报错跑不下去的问题发生。


我们的项目可能存在以上问题。例如：在测试阶段：ehcache超时，导致内存结构销毁，消费者拿到数据无法放入到内存中，导致失败，这样retry几次也是报错，导致数据丢失，所以我们为了解决因为数据丢失导致闭市结算发生错误的情况，我们在实时消费kafka topic数据时检测，若发生错误后导致丢失数据时，为了减少后续报错信息在控制台打印，我们直接将消费者监听关闭，不在继续接收kafka数据，在闭市时直接从数据库中拉取数据。
这样就不会出现了闭市报错的情况。


另外，我进行多活场景测试，测试结果显示如果未进行手动提交offset，consumer每次重启都会重新拉取全量数据。这个现象表示：在某些特殊情况，比如多活场景，当一个点进行消费数据时，发生宕机，另一个点需要顶上来进行继续工作时，由于第一个节点未进行提交offset，kafka检测出第一个consumer失败了，kafka自动rebalance 到相同组的其他消费者上面如下图：
![rebalance](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/46A1DAD5DC2F498F9AF672F63170AB31/25000)
然后第二个consumer 会从头消费所所有kafka里面的数据：
![多活时第一个点活着在工作](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/8F52F16D281144BB83F6590216D39EA4/24987)
![](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/D9A07B100D904F6195649CC0AA44EECB/24989)
![](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/EACD70AD6760486EB0C3B9105576BA87/24991)
多活时第一个点活着在工作
此时我将consumer1kill掉：
![](http://note.youdao.com/yws/public/resource/c336c9f401f7ed2acff65c1b781d2c6c/xmlnote/E006FC9AF0A54EC29439AC0AD02435C8/24995)
consumer2 从头拉取数据。

综合项目考虑单机方案有两种：

方案一、两个点同时进行结算初始化（待讨论如何做这个操作而不会两个点同时进行闭市），
两个点的kafka consumer 的group id 配置**不同**情况此时，在连续交易阶段我们的两个点会同时接收topic里面的数据，当第一个点挂了，第二个点会完整的跑完闭市结算。

方案二、两个点同时进行结算初始化（待讨论如何做这个操作而不会两个点同时进行闭市），
两个点的kafka consumer 的group id 配置**相同**情况此时，连续交易阶段我们的只有主点进行接收topic数据，当第一个点挂了而为commit offset，第二个点会从新从头拉取topic所有数据后完整的跑完闭市结算。

以上两个方案同时存在一个问题：
需要两个点同时进行结算初始化而不会两个点同时进行闭市的实现。
给出方案解决该问题,**在启动两个点时直接将结算初始化启动**，不在客户端点击该按钮。数据块闭市结算也不需要点击结算初始化。



参考文献

1、[Kafka EOS](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)
