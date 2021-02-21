
#### kafka-hw 高水位线 与 leader epoch

KAFKA 服务端一个日志文件包含两个特性：日志末端位移（log-end-offset）和高水印（high-watermask），无论是LEADER还是follow副本均含有这两个特性：

LEO：该副本底层log文件下一条要写入的消息的位移，例如LEO=10则当前文件已经写了了10条消息，位移是[0,10)。

HW：所有分区已提交的的位移，HW之外consumer无法消费，一般HW<=LEO。

#### 一 LEO更新机制

1.1 follow的LEO更新机制

follow的LEO更新机制包括follow所在broker的follow的LEO更新机制以及Leader所在broker的follow的LEO更新机制。

follow所在broker的follow的LEO更新机制主要是为了follow的HW更新；

Leader所在broker的follow的LEO更新机制主要是为了Leader的HW更新。

（1）follow所在broker的follow的LEO更新机制

在follow向Leader发送fetch同步请求后，Leader将数据返回，此时follow向底层log写入数据然后更新LEO。

（2）Leader所在broker的follow的LEO更新机制

在follow向Leader发送fetch同步请求后，Leader收到后首先从自己底层log读取数据，然后根据Leader上的follow的LEO。再向follow发送响应。

1.2 Leader的LEO更新机制

在producer向Leader发送producer同步请求后，Leader收到数据，此时Leader向底层log写入数据然后更新LEO。


#### 二 HW更新机制

2.1 follow的HW更新机制

更新时机：follow收到Leader的fetch响应更新follow端的LEO后，尝试更新follow的HW。

更新方式：HW=min(follow端的LEO,fetch响应中LEADER的HW)

2.2 LEADER的HW更新机制

更新时机：
- (1)新副本成为LEADER副本
- (2)broker崩溃；
- (3)producer向LEADER请求写入了数据更新了LEADER的LEADER LEO；
- (4)follow向LEADER请求同步，更新了LEADER的follow的LEO

更新方式：HW=min(LEADER LEO, all follows' LEO in LEADR broker)

这里的所有follow包括ISR以及即将具备入ISR还没来得及入ISR的副本（副本LEO落后LEADER LEO的时长低于replica.lag.time.max.ms）

设一个主题，分区数1，副本因子2。


第一轮fetch



生产者给该topic发送了一条信息，待Leader写入log后更新Leader端的LEO=1；尝试更新Leader 的hw，hw=min(leader leo,follow leo)=0故hw=0

follow发送fetch请求，请求中附带follow端的follow leo=0,Leader读取log然后更新Leader的follow leo=0（因为fetch请求中附带follow端的follow leo=0）；尝试更新Leader hw=min(leader leo,follow leo)=0,故塞给fetch相应中的leader hw=0

follow收到fetch请求后，写log，更新leo=1，follow的hw=min(leo, leader hw)=0

第二轮fetch



follow发送fetch请求，请求中附带follow端的follow leo=1,Leader读取log然后更新Leader的follow leo=1（因为fetch请求中附带follow端的follow leo=1）；尝试更新Leader hw=min(leader leo,follow leo)=1故塞给fetch相应中的leader hw=1

follow收到fetch请求后，写log,此时无数据，故leo=1，follow的hw=min(leo, leader hw)=1


### spring kafka 注解学习

##### @KafkaListener
```
@KafkaListener(topics = KafkaProducer.TOPIC_TEST, groupId = KafkaProducer.TOPIC_GROUP1, id = "1", containerFactory = "123")
```

```
//The unique identifier of the container for this listener. listener ID 
String id() default "";

//The bean name of the KafkaListenerContainerFactory to use to create the message listener container responsible to serve this endpoint.
String containerFactory() default "";

//可以写多个topic同时被监听。
String[] topics() default {};
// 正则匹配 topic
String topicPattern() default "";
//topic 对应的分区 可以写入多个。
TopicPartition[] topicPartitions() default {};

//If provided, the listener container for this listener will be added to a bean with this value as its name, of type Collection<MessageListenerContainer>.
String containerGroup() default "";

//Set an KafkaListenerErrorHandler bean name to invoke if the listener method throws an exception.
String errorHandler() default "";

//消费者组ID
String groupId() default "";

//When groupId is not provided, use the id (if provided) as the group.id property for the consumer.
boolean idIsGroup() default true;

//When provided, overrides the client id property in the consumer factory configuration.
String clientIdPrefix() default "";

//A pseudo bean name used in SpEL expressions within this annotation to reference the current bean within which this listener is defined.
String beanRef() default "__listener";

//Override the container factory's concurrency setting for this listener.
String concurrency() default "";

//自动开启监听：true or false
String autoStartup() default "";

//Kafka consumer properties; they will supersede any properties with the same name defined in the consumer factory (if the consumer factory supports property overrides).
String[] properties() default {};

//	
When false and the return type is an Iterable return the result as the value of a single reply record instead of individual records for each element.
boolean splitIterables()
```
