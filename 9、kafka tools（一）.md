## kafka tools 学习（一）
目前本机kafka版本与项目用一致：2.5.0，提供以下工具命令。

``` shell
 connect-distributed.sh
 connect-mirror-maker.sh
 connect-standalone.sh
 kafka-acls.sh
 kafka-broker-api-versions.sh
 kafka-configs.sh
 kafka-console-consumer.sh
 kafka-console-producer.sh
 kafka-consumer-groups.sh
 kafka-consumer-perf-test.sh
 kafka-delegation-tokens.sh
 kafka-delete-records.sh
 kafka-dump-log.sh
 kafka-leader-election.sh
 kafka-log-dirs.sh
 kafka-mirror-maker.sh
 kafka-preferred-replica-election.sh
 kafka-producer-perf-test.sh
 kafka-reassign-partitions.sh
 kafka-replica-verification.sh
 kafka-run-class.sh
 kafka-server-start.sh
 kafka-server-stop.sh
 kafka-streams-application-reset.sh
 kafka-topics.sh
 kafka-verifiable-consumer.sh
 kafka-verifiable-producer.sh
 nohup.out
 trogdor.sh
 zookeeper-security-migration.sh
 zookeeper-server-start.sh
 zookeeper-server-stop.sh
 zookeeper-shell.sh

```


### kafka-broker-api-versions.sh 版本信息
```
./kafka-broker-api-versions.sh –bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092
```
![20201217194027](https://github.com/weifangZ/image/blob/master/image20201217194027.png)

### kafka-consumer-groups.sh 消费者组

消费组 描述
```
kafka-consumer-groups.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --all-groups --describe
```
![20201217194333](https://github.com/weifangZ/image/blob/master/image20201217194333.png)

消费组 描述
```
kafka-consumer-groups.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --all-groups --describe
```
查看有那些 group ID 正在进行消费：

```
kafka-consumer-groups.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --group all-zwf2 --describe
```
![20201217195609](https://github.com/weifangZ/image/blob/master/image20201217195609.png)


### kafka-delete-records.sh 删除低水位的日志文件
1、删除数据
```
./kafka-delete-records.sh --bootstrap-server 192.168.131.131:9092,192.168.131.129:9092,192.168.131.128:9092 --offset-json-file mcTrade.json 
```
mcTrade.json
```
{
    "partitions": [{
        "topic": "mcTrade",
        "partition": 0,
        "offset": 9
    }],
    "version": 1
}
```
![20201223171339](https://github.com/weifangZ/image/blob/master/image20201223171339.png)
查看对应的offset
![20201223171415](https://github.com/weifangZ/image/blob/master/image20201223171415.png)
发现数据时删除了9条，但是offset并没有降低，说明这个删除只是delete 操作，通过low_watermark: 9这样的进行的操作。

此时"offset": 9这里的9不能高于topic的offset，否则会报错，所以在使用这个命令时，需要知道offset位置。

### kafka-log-dirs.sh kafka消息日志目录信息

```
kafka-log-dirs.sh --bootstrap-server 192.168.131.131:9092 --topic-list mcTrade --describe

```
![20201223191726](https://github.com/weifangZ/image/blob/master/image20201223191726.png)

### kafka-mirror-maker.sh 不同数据中心kafka集群复制工具

```
kafka-mirror-maker.sh 

Option                                   Description                           
------                                   -----------                           
--abort.on.send.failure <String: Stop    Configure the mirror maker to exit on 
  the entire mirror maker when a send      a failed send.（默认为true，决定生产者写入失败时的处理机制） (default: true)      
  failure occurs>                                                              
--consumer.config <String: config file>  Embedded consumer config for consuming
                                           from the source cluster（用于指定消费者的配置文件，配置文件里有两个必填的参数：bootstrap.servers 和 group.id）.            
--consumer.rebalance.listener <String:   The consumer rebalance listener to use
  A custom rebalance listener of type      for mirror maker consumer.（指定再均衡监听器）          
  ConsumerRebalanceListener>                                          
--help                                   Print usage information.              
--message.handler <String: A custom      Message handler which will process    
  message handler of type                  every record in-between consumer and
  MirrorMakerMessageHandler>               producer.（指定消息的处理器。这个处理器会在消费者消费到消息之后且在生产者发送消息之前被调用）                           
--message.handler.args <String:          Arguments used by custom message      
  Arguments passed to message handler      handler for mirror maker.（指定消息处理器的参数，同message.handler一起使用）           
  constructor.>                                                                
--new.consumer                           DEPRECATED Use new consumer in mirror 
                                           maker (this is the default so this  
                                           option will be removed in a future  
                                           version).                           
--num.streams <Integer: Number of        Number of consumption streams. （指定消费线程的数量）       
  threads>                                 (default: 1)                        
--offset.commit.interval.ms <Integer:    Offset commit interval in ms.         
  offset commit interval in                (default: 60000) （指定消费位移提交间隔）                   
  millisecond>                                                                 
--producer.config <String: config file>  Embedded producer config.（用于指定生产者的配置文件，配置文件里唯一必填的参数是 bootstrap.servers）             
--rebalance.listener.args <String:       Arguments used by custom rebalance    
  Arguments passed to custom rebalance     listener for mirror maker consumer. （指定再均衡监听器的参数，同consumer.rebalance.listener一起使用）
  listener constructor as a string.>                                           
--version                                Display Kafka version.                
--whitelist <String: Java regex          Whitelist of topics to mirror. （whitelist 指定需要复制的源集群中的主题。这个参数可以指定一个正则表达式，比如a|b表示复制源集群中主题a和主题b的数据。为了方便使用，这里也允许将“|”替换为“,”）       
  (String)>                                                                 
```
一、MirrorMaker介绍

MirrorMaker是Kafka附带的一个用于在Kafka集群之间制作镜像数据的工具。**该工具从源集群中消费并生产到目标群集**。这种镜像的常见用例是在另一个数据中心提供副本。
其实现原理是通过从源集群中消费消息，然后将消息生产到目标集群中，也就是普通的生产和消费消息。
用户只需要在启动Kafka Mirror Maker时指定一些简单的消费端和生产端配置就可以实现准实时的数据同步。

二、kafka-mirror-maker.sh脚本使用

2.1、先决条件

首先目标集群的kafka服务端必须开启 auto.create.topics.enable=true ，以允许自动创建topic；或者对于需要进行镜像的topic都自己手动进行创建。自动创建的topic会根据服务端配置的num.partitions、default.replication.factor决定。

2.2、演示使用

演示从集群1中将主题test-perf的数据同步到集群2中，首先创建并配置两个配置文件，参考如下：
```
#consumer.properties的配置
bootstrap.servers=kafka1:9092
group.id=groupIdMirror
client.id=sourceMirror
partition.assignment.strategy=org.apache.kafka.clients.consumer.RoundRobinAssignor
```
这里要说一下kafka的三种分区策略：

1、Range(默认策略)
Range分区是针对同一topic的进行分组，分组规则为：
n=分区数/消费者数量，m=分区数%消费者数量，那么前m个消费者每个分配n+1个分区，后面的（消费者数量-m）个消费者每个分配n个分区。

假如有10个分区，3个消费者线程，把分区按照序号排列0，1，2，3，4，5，6，7，8，9；消费者线程为C1-0，C2-0，C2-1，那么用partition数除以消费者线程的总数来决定每个消费者线程消费几个partition，如果除不尽，前面几个消费者将会多消费一个分区。在我们的例子里面，我们有10个分区，3个消费者线程，10/3 = 3，而且除除不尽，那么消费者线程C1-0将会多消费一个分区，所以最后分区分配的结果看起来是这样的：

```
C1-0：0，1，2，3
C2-0：4，5，6
C2-1：7，8，9
```
如果有11个分区将会是：
```
C1-0：0，1，2，3
C2-0：4，5，6，7
C2-1：8，9，10
```
假如我们有两个主题T1,T2，分别有10个分区，最后的分配结果将会是这样：
```
C1-0：T1（0，1，2，3） T2（0，1，2，3）
C2-0：T1（4，5，6） T2（4，5，6）
C2-1：T1（7，8，9） T2（7，8，9）
```
可以看出， C1-0消费者线程比其他消费者线程多消费了2个分区

如上，只是针对 1 个 topic 而言，C1-0消费者多消费1个分区影响不是很大。如果有 N 多个 topic，那么针对每个 topic，消费者 C1-0 都将多消费 1 个分区，topic越多，C1-0 消费的分区会比其他消费者明显多消费 N 个分区。这就是 Range 范围分区的一个很明显的弊端了


2、RoundRobin

RoundRobinAssignor策略的原理是将消费组内所有消费者以及消费者所订阅的所有topic的partition按照字典序排序，然后通过轮询方式逐个将分区以此分配给每个消费者。RoundRobinAssignor策略对应的partition.assignment.strategy参数值为：
```
org.apache.kafka.clients.consumer.RoundRobinAssignor
```
使用RoundRobin策略有两个前提条件必须满足：
- 同一个消费者组里面的所有消费者的num.streams（消费者消费线程数）必须相等；
- 每个消费者订阅的主题必须相同。否则会分配不均匀

假设一个有两个消费者C1、C2的num.streams= 2同时消费同一个主题T1，而这个主题有10分区
我们的例子里面，加入按照 hashCode 排序完的topic-partitions组依次为T1-5, T1-3, T1-0, T1-8, T1-2, T1-1, T1-4, T1-7, T1-6, T1-9，我们的消费者线程排序为C1-0, C1-1, C2-0, C2-1，最后分区分配的结果为：
```
C1-0 将消费 T1-5, T1-2, T1-6 分区；
C1-1 将消费 T1-3, T1-1, T1-9 分区；
C2-0 将消费 T1-0, T1-4 分区；
C2-1 将消费 T1-8, T1-7 分区；
```
3、StickyAssignor

我们再来看一下StickyAssignor策略，“sticky”这个单词可以翻译为“粘性的”，Kafka从0.11.x版本开始引入这种分配策略，它主要有两个目的：
- 分区的分配要尽可能的均匀，分配给消费者者的主题分区数最多相差一个；
- 分区的分配尽可能的与上次分配的保持相同。

举个例子：

假设消费组内有3个消费者：C0、C1和C2，它们都订阅了4个主题：t0、t1、t2、t3，并且每个主题有2个分区，也就是说整个消费组订阅了t0p0、t0p1、t1p0、t1p1、t2p0、t2p1、t3p0、t3p1这8个分区。最终的分配结果如下：
```
消费者C0：t0p0、t1p1、t3p0
消费者C1：t0p1、t2p0、t3p1
消费者C2：t1p0、t2p1
```
这里动态平衡后，假设C1退出了消费者组。此时会rebalance：

```
消费者C0：t0p0、t1p0、t2p0、t3p0
消费者C2：t0p1、t1p1、t2p1、t3p1
```
而robin的会重新分配成：
```
消费者C0：t0p0、t1p1、t3p0、t2p0
消费者C2：t1p0、t2p1、t0p1、t3p1
```
可以看到分配结果中保留了上一次分配中对于消费者C0和C2的所有分配结果，并将原来消费者C1的“负担”分配给了剩余的两个消费者C0和C2，最终C0和C2的分配还保持了均衡。
同理，不同消费者订阅的topic数量不同，分区不同是也是StickyAssignor策略更加优越，算法更加复杂。

```
#producer.properties的配置
bootstrap.servers=kafka2:9092
client.id=sinkMirror
```
consumer.properties和producer.properties这两个配置文件中的配置对应消费者客户端和生产者客户端的配置。

下面启动Kafka Mirror Maker：
```
./kafka-mirror-maker.sh --consumer.config ../config/consumer.properties --producer.config ../config/producer.properties --whitelist 'test-perf' --num.streams 6
```
待补充：执行

2.3、使用注意事项

kafka mirror maker中对于生产者写入失败时，会根据abort.on.send.failure的配置分两种处理方法；
abort.on.send.failure=true ，此时如果生产者经过多次重试依然无法完成消息写入，则会直接停止kafka mirror maker进程；
abort.on.send.failure=false ，此时如果生产者经过多次重试依然无法完成消息写入，则会直接跳过当前写入的批次数据，直接进行下一轮消息的写入；

源集群和目标集群是两个完全独立的实体，对每个主题而言，两个集群之间的分区数可能不同；就算分区数相同，那么经过消费再生产之后消息所规划到的分区号也有可能不同；就算分区数相同，消息所规划到的分区号也相同，那么消息所对应的offset也有可能不相同，例如，源集群中由于执行了某次日志清理操作，某个分区的logStartOffset值变为10，而目标集群中对应分区的logStartOffset还是0，那么从源集群中原封不动的复制到目标集群时，同一条消息的offset也会不相同。



### kafka-preferred-replica-election.sh


kafka-preferred-replica-election命令 是用于对Leader进行重新负载均衡
```
This tool is deprecated. Please use kafka-leader-election tool. Tracking issue: KAFKA-8405
```
使用方式：
1、 触发对所有的topic Leader进行负载均衡
```
./kafka-leader-election.sh --bootstrap-server 192.168.131.128:9092 --all-topic-partitions --election-type preferred
./kafka-leader-election.sh --bootstrap-server 192.168.131.128:9092 --all-topic-partitions --election-type unclean

```
2、对某个topic Leader触发负载均衡
```
./kafka-leader-election.sh --bootstrap-server 192.168.131.128:9092 --topic mcTrade --partition 0  --election-type unclean
./kafka-leader-election.sh --bootstrap-server 192.168.131.128:9092 --topic mcTrade --partition 0  --election-type preferred
```
![20201224145558](https://github.com/weifangZ/image/blob/master/image20201224145558.png)
3、批量触发负载均衡
```
./kafka-leader-election.sh --bootstrap-server 192.168.131.128:9092 --path-to-json-file mcTrade.js  --election-type preferred
```

### kafka-producer-perf-test.sh kafka-consumer-perf-test.sh 官方提供的测试工具

机器配置：
![20201225100906](https://github.com/weifangZ/image/blob/master/image20201225100906.png)

1、创建测试用的topic
```
kafka-topics.sh--create --bootstrap-server 192.168.131.128:9092 --topic test-rep-one --partitions 3 --replication-factor 1
```
2、生产数据

```
kafka-producer-perf-test.sh --topic test-rep-one  --num-records 500000 --record-size 200  --throughput -1  --producer-props  bootstrap.servers=192.168.131.128:9092 acks=-1
```
结果如下：
![20201225100255](https://github.com/weifangZ/image/blob/master/image20201225100255.png)
可以看出Kafka producer的平均吞吐量是6.96MB/s，即占用64Mb/s左右的带宽，平均每秒能发送36501条消息，平均延时是3.24秒，最大延时是6.1秒，平均有50%的消息发送需要花费3.2秒，95%的消息发送需要花费5.6秒 等等。
参数的含义：
num-records：总共需要发送的消息数，本例为500000
record-size：每个记录的字节数，本例为200
throughput：每秒钟发送的记录数

3、消费者数据
```
kafka-consumer-perf-test.sh --broker-list 192.168.131.131:9092  --messages 500000 --topic test-rep-one 
```
结果如下：
![20201225101313](https://github.com/weifangZ/image/blob/master/image20201225101313.png)
### kafka-reassign-partitions.sh

参考文献

1、[脚本kafka-configs.sh用法解析](https://www.cnblogs.com/lizherui/p/12275193.html)

2、[kafka-mirror-maker.sh脚本](https://blog.csdn.net/qq_41154944/article/details/108282641)
