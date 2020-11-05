### 学习计划
<!--![20201023112106](https://raw.githubusercontent.com/weifangZ/image/master/image/20201023112106.png)-->

### 一、准备环境
运维学习环境以清算生产环境部署为模型进行模拟，搭建三点集群环境。
```
kafka4 ： 192.168.131.131
kafka3 ： 192.168.131.129
kafka2 ： 192.168.131.128
```
### 二、kafka运维学习重点
**2.1、kafka常用运维操作指令。（学习中）**

1、创建topic
```
./kafka-topics.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --create --replication-factor 3 --partitions 1 --topic mcTrade
```
以上命令代表分区1副本3\
分区的原因：\
1、提高扩展性\
2、提高并发性

生产者的分区原则：\
1、指明对应的partition就操作对应的partition。\
2、没指明，但是有key时候使用key对topic的partition数量进行取余数，得到的值取操作相应的partition。\
3、没partition与没key时随机出一个整数，然后按照这个值递增，对topic的partition数量进行取余数，得到的值取操作相应的partition。\
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/3A82DC842F2A4675BCB4249E58168D00/23460)

2、查看topic
```
./kafka-topics.sh --list --bootstrap-server 192.168.131.128:9092
```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/E0BE9D583C454B139DA9D063172A9D59/23351)
3、删除topic
```
./kafka-topics.sh --delete --bootstrap-server 192.168.131.128:9092 --topic mcTrade
```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/55160F3A36DF4FC1A74A875D8CE46152/23361)


4、描述
```
./kafka-topics.sh --describe --bootstrap-server 192.168.131.128:9092 --topic mcTrade
```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/728A18674D094E0190A9E62E1A26CFE4/23368)

5、修改分区副本
```
./kafka-topics.sh --bootstrap-server 192.168.131.128:9092,192.168.131.129:9092,192.168.131.131:9092 --alter --replication-factor 4 --partitions 2 --topic mcTrade
```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/C518E57A18184C6E850583D3ECA41B7C/23389)
变为
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/BF6B8AF12CA34CD99AA2E339BF998986/23393)

6、生产者写数据与消费者消费数据命令productor&consumer
```
//productor
./kafka-console-producer.sh --topic mcTrade --bootstrap-server 192.168.131.128:9092
//consumer
./kafka-console-consumer.sh --topic mcTrade --bootstr-server 192.168.131.128:9092 --from-beginning

```
![](http://note.youdao.com/yws/public/resource/ffefb6fa5bca403ed5711d3e6aed479d/xmlnote/6E1598CEA6F149B29C120175C4E0BE6B/23377)







工作流程\
当生产者向不存在的topic里面发送数据，会默认生产一个1个分区1个副本的topic。
每个分区都维护一个offet
forllower  要同步leader数据

**2.2、配置（学习中）** ==（2020年11月2-6日）==

**数据可靠性保证方式：**
为了保证producor发送的数据尽量可靠的发送到指定的topic中，topic的每个partition收到producer发送的数据后都向producer发送ack，如果producer收到ack后进行下一轮发送。如果收不到ack则进行重新发送。

这样就会出现何时发送ack呢？
- 1、leader和follower都收到了在发ack \
优点：副本少 缺点：延迟高
- 2、半数以上的follower收到后发送ack \
优点：延迟低 缺点：副本多

kafka 选用第二种方案，做了个优化提出了**ISR**


**1、ISR:in-sync replica set （同步副本）**\
应对数据可靠性保证。解决副本故障不能发送ack的问题\
kafka查看ISR中的follower是否同步完成，完成后直接发ack不在等待不在ISR中的follower是否同步完成。当然如果出现ISR中的follower出现了长时间不同步数据，就将它提出ISR重新选取follower。
选取follower \
进入ISR原则：
- leader节点挂掉了后选取新的leader尽量不丢数据。
- 尽量选offset高的，丢数据丢的少。
- replica.lag.time.max.ms 如果follower不能按时发送一个请求，或者不能消费最后一下offset时会移除isr队列，默认10秒。
- replica.lag.time.max.message（被移除了）
（两个条件的时候因为，会出现，满足1进去，后不满足2时剔除isr，这样会频繁的操作内存。，还要操作Zookeeper。0.9版本后移除）
- replica.fetch.wait.max.ms	副本follow同leader之间通信的最大等待时间，失败了会重试。 此值始终应始终小于replica.lag.time.max.ms，以防止针对低吞吐量topic频繁收缩ISR
- replica.lag.time.max.ms	如果一个follower在这个时间内没有发送fetch请求或消费leader日志到结束的offset，leader将从ISR中移除这个follower，并认为这个follower已经挂了	long	10000		高

**2、ack应答机制**
提供三种级别（acks配置）：
- 0，leader写完无论follower写没写完，producer只发一遍不等leader与follower时候写完。**会丢数据**
- 1，只等待leader，不等到follower写完就同步数据，此时leader挂了会丢数据。 **会丢数据**
- -1（all），此时等待leader与isr里面的follower全部完成后来返回ack。极限情况也会有丢数据时，isr只有leader了，follower特别慢时候isr就只有leader，退化到了1的情况。**重复数据，保证producer数据不丢失**\
==补充：==
- **min.insync.replicas**	当producer将ack设置为“全部”（或“-1”）时，min.insync.replicas指定了被认为写入成功的最小副本数。如果这个最小值不能满足，那么producer将会引发一个异常（NotEnoughReplicas或NotEnoughReplicasAfterAppend）。当一起使用时，min.insync.replicas和acks允许您强制更大的耐久性保证。 一个经典的情况是创建一个复本数为3的topic，将min.insync.replicas设置为2，并且producer使用“all”选项。 这将确保如果大多数副本没有写入producer则抛出异常。

等于-1时会重复数据，leader与follower已经同步完成，此时leader挂了，还没发送ack，选一个follower当leader，此时producer会重复发送数据。

**3、数据存储一致性问题**

**保证消费者消费数据一致性**
- **HW** 高水位线（多个isr中最小的offset）
replica.high.watermark.checkpoint.interval.ms	high watermark被保存到磁盘的频率，用来标记日后恢复点/td>
- **LEO** 最后一个offset （每个副本中的最新的那个offset）\
kafka对于消费者暴露的只有hw，此时能够保证消费者消费的数据一致性问题。木桶短板效应。

**存储数据一致性问题**

重新选择leader时，给所有follower发消息要截取到HW位置。高于HW的数据全部截取掉，不进行存储。

**HW**只保证数据一致性而不保证数据丢不丢与数据重复不重复问题。

 **4、Broker Configs**
 
    参考文献：详情请查看中文官网：https://kafka.apachecn.org/documentation.html#brokerconfigs
    核心基础配置如下:
    - broker.id 用于服务的broker id。如果没设置，将生存一个唯一broker id。为了避免ZooKeeper生成的id和用户配置的broker id相冲突，生成的id将在reserved.broker.max.id的值基础上加1。	int	-1		高
    - log.dirs
    - zookeeper.connect
    其他认为有用的配置：
    - auto.create.topics.enable	是否允许在服务器上自动创建topic,无topic时向这个topic中生产数据，会自动创建一个分区1复本为3（3点集群）的topic	boolean	true		高
    
    - delete.topic.enable	是否允许删除topic。如果关闭此配置，通过管理工具删除topic将不再生效。	boolean	true		高
    - listeners	监听器列表 - 使用逗号分隔URI列表和监听器名称。如果侦听器名称不是安全协议，则还必须设置listener.security.protocol.map。指定主机名为0.0.0.0来绑定到所有接口。留空则绑定到默认接口上。合法监听器列表的示例：PLAINTEXT：// myhost：9092，SSL：//：9091 CLIENT：//0.0.0.0：9092，REPLICATION：// localhost：9093	string	null		高
    - log.dir（迁就手残党，非专业人士）	保存日志数据的目录（对log.dirs属性的补充）	string	/tmp/kafka-logs		高
    - log.dirs	保存日志数据的目录，如果未设置将使用log.dir的配置。	string	null		高
    - log.retention.bytes	日志删除的大小阈值	long	-1（不限制）		高
    - log.retention.hours	日志删除的时间阈值（小时为单位）	int	168（7天）		高 对应还有分钟、毫秒的设置。
    - log.segment.bytes	单个日志段文件最大大小	int	1073741824（1GB）	[14,...]	高
![zookeeper-connect](https://github.com/weifangZ/accumulate/blob/main/images/zookeeper-connect.png)
![auto.create.topics.enable](https://github.com/weifangZ/accumulate/blob/main/images/cteate%20topic.png)

**5、Topic-Level Configs**

**--config** 
可以在create时和alter   
**--alter --delete-config**
```
bin/kafka-topics.sh --bootstrap-service 170.0.131.128:9092 --create --topic mcTrade --partitions 3
    --replication-factor 3 --config max.message.bytes=64000 --config flush.messages=1
```

**2.3、kafka性能调优（待学习）**

**2.4、APIs（待学习）**


**2.5、存储（学习中）**

index文件与
log文件
快速查找定位。
每个数据块大小相同，通过offset乘以块大小直接定位数据位置。




### 三、kafka常见问题汇总

**问题1**：kafka topic数据与内存数据不一致的情况。\
**首先：** 
查看消费情况
```
./kafka-consumer-groups.sh --describe --bootstrap-server 170.0.39.164:9092 --group dce-mt-realTime-wanghong12345
```
执行下面命令即可将日志数据文件内容dump出来 
```
./kafka-run-class.sh kafka.tools.DumpLogSegments --files /home/dce/clear/allmetadata/mcTrade-0/00000000000022372103.log --print-data-log > 00000000000022372103_txt.log
```
通过对比内存与kafka中的导出数据进行对比可以解决。

问题2：==（2020年11月2-6日）==
kafka.clients.consumer.commitFailedException:commit cannot be completed

原因

![](https://github.com/weifangZ/accumulate/blob/main/images/re-balance.png)

有个很重要的概念，kafka会管理如何分配分区的消费。 

（1）当某个consumer消费了数据，但是在特定的时间内没有commit，它就认为consumer挂了。这个时候就要rebalance了。这个时间和“heartbeat.interval.ms”配置相关。 

（2）每次consumer从kafka poll数据时，每次poll会有一个量的控制，“max.partition.fetch.bytes”配置决定。

如果有两个消费者同时消费一个topic时，而且他们的groupId相同，会只有一个消费者进行消费，如果这个消费者组出现被consumer判断为挂了，则会让另一个消费之进行消费。导致单点丢数据。
